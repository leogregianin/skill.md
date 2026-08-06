# FastAPI: Referência de Dependências

## Sub-dependências

Dependências podem depender de outras dependências — o FastAPI resolve e faz cache da cadeia por requisição:

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

## Dependências com yield e cleanup

Uma dependência que usa `yield` executa seu teardown (o código após o `yield`) mesmo que a path operation lance uma exceção, de forma parecida com `try/finally`:

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

## Cache

Dentro de uma mesma requisição, o FastAPI chama uma dada dependência apenas uma vez e reutiliza o resultado em toda path operation/dependência que a declarar — a menos que você desative isso:

```python
def get_request_id(x_request_id: str = Header(...)) -> str:
    return x_request_id


@router.get("/clientes")
def listar_clientes(
    request_id: str = Depends(get_request_id),
    request_id_again: str = Depends(get_request_id, use_cache=False),  # força uma segunda chamada
):
    ...
```

## Autenticação com Security()

`Security()` é `Depends()` com escopos OAuth2, usado com `SecurityScopes` para aplicar permissões granulares:

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

## Dependências globais

Aplique uma dependência a todas as rotas de um router (ou de toda a aplicação) em vez de repeti-la em cada path operation:

```python
router = APIRouter(dependencies=[Depends(verify_api_key)])
```
