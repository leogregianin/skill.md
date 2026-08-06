# FastAPI: Dependencies Reference

## Sub-dependencies

Dependencies can depend on other dependencies — FastAPI resolves and caches the chain per request:

```python
def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


def get_cliente_service(db: Session = Depends(get_db)) -> ClienteService:
    return ClienteService(db)


@router.get("/clientes/{cliente_id}")
def obter_cliente(cliente_id: int, service: ClienteService = Depends(get_cliente_service)):
    return service.buscar_por_id(cliente_id)
```

## Yield dependencies with cleanup

A dependency using `yield` runs its teardown (the code after `yield`) even if the path operation raises, similar to `try/finally`:

```python
def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    except Exception:
        db.rollback()
        raise
    finally:
        db.close()
```

## Caching

Within a single request, FastAPI calls a given dependency only once and reuses the result for every path operation/dependency that declares it — unless you opt out:

```python
def get_request_id(x_request_id: str = Header(...)) -> str:
    return x_request_id


@router.get("/clientes")
def listar_clientes(
    request_id: str = Depends(get_request_id),
    request_id_again: str = Depends(get_request_id, use_cache=False),  # forces a second call
):
    ...
```

## Auth with Security()

`Security()` is `Depends()` plus OAuth2 scopes, used with `SecurityScopes` to enforce fine-grained permissions:

```python
def get_current_user(
    security_scopes: SecurityScopes,
    token: str = Depends(oauth2_scheme),
) -> User:
    user = decode_token(token)
    for scope in security_scopes.scopes:
        if scope not in user.scopes:
            raise HTTPException(status_code=403, detail="permissão insuficiente")
    return user


@router.delete("/clientes/{cliente_id}")
def remover_cliente(
    cliente_id: int,
    user: User = Security(get_current_user, scopes=["clientes:delete"]),
):
    ...
```

## Global dependencies

Apply a dependency to every route in a router (or the whole app) instead of repeating it on each path operation:

```python
router = APIRouter(dependencies=[Depends(verify_api_key)])
```
