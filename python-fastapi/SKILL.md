---
name: fastapi
description: Best practices and conventions for FastAPI projects. Use when writing or reviewing FastAPI code — routing, Pydantic schemas, dependency injection, async, error handling, and testing.
---

# FastAPI

Generic skill for writing FastAPI code following best practices, keeping consistency and avoiding common pitfalls (blocking calls in async routes, duplicated validation logic, untyped responses).

## Quick Reference

* Project structure: split routers, schemas, and dependencies into separate modules; see [Project Structure](#project-structure).
* Schemas: use Pydantic models for request/response validation, never raw dicts; see [Pydantic Schemas](#pydantic-schemas).
* Routing: use `APIRouter` per resource with `prefix`/`tags`; see [Routers](#routers).
* Dependency Injection: use `Depends()` for shared logic (DB sessions, auth, pagination); see [Dependency Injection](#dependency-injection) and the [dependencies reference](references/dependencies.md) for sub-dependencies, `yield` dependencies, and `Security()`.
* Async: use `async def` only when the body is actually non-blocking; run blocking I/O in a thread pool; see [Async](#async).
* Error handling: raise `HTTPException` for expected errors; use exception handlers for cross-cutting cases; see [Error Handling](#error-handling).
* Testing: use `TestClient`/`httpx.AsyncClient` with `pytest`; see [Testing](#testing) and the [testing reference](references/testing.md) for dependency overrides and async test clients.

## Project Structure

```
myproject/
├── main.py
├── myproject/
│   ├── core/
│   │   ├── config.py
│   │   └── security.py
│   ├── api/
│   │   ├── deps.py
│   │   └── v1/
│   │       ├── routers/
│   │       │   └── clientes.py
│   │       └── __init__.py
│   ├── schemas/
│   │   └── cliente.py
│   ├── models/
│   │   └── cliente.py
│   └── services/
│       └── cliente.py
└── tests/
    └── test_clientes.py
```

Keep path operation functions thin — validation lives in Pydantic schemas, business logic in a service layer, and persistence in models/repositories.

## Pydantic Schemas

Separate input and output schemas instead of reusing one model for everything:

```python
from pydantic import BaseModel, EmailStr


class ClienteCreate(BaseModel):
    nome: str
    email: EmailStr


class ClienteRead(BaseModel):
    id: int
    nome: str
    email: EmailStr

    class Config:
        from_attributes = True
```

Use `response_model` on every path operation so FastAPI filters and documents the output shape:

```python
@router.post("/clientes", response_model=ClienteRead, status_code=201)
def criar_cliente(payload: ClienteCreate) -> Cliente:
    ...
```

## Routers

Group related path operations with `APIRouter`, and mount them in `main.py`:

```python
# api/v1/routers/clientes.py
router = APIRouter(prefix="/clientes", tags=["clientes"])


@router.get("/{cliente_id}", response_model=ClienteRead)
def obter_cliente(cliente_id: int, service: ClienteService = Depends(get_cliente_service)):
    return service.buscar_por_id(cliente_id)
```

```python
# main.py
app.include_router(clientes.router, prefix="/api/v1")
```

## Dependency Injection

Use `Depends()` for anything shared across path operations — DB sessions, current user, pagination params:

```python
def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


@router.get("/clientes")
def listar_clientes(db: Session = Depends(get_db)):
    return db.query(Cliente).all()
```

See the [dependencies reference](references/dependencies.md) for sub-dependencies, `yield`-based dependencies with cleanup, caching with `Depends(..., use_cache=True)`, and `Security()` for auth scopes.

## Async

Only declare a path operation `async def` if its body doesn't block the event loop. A blocking call (sync DB driver, `requests`, CPU-bound work) inside an `async def` route stalls every other request:

```python
# Bad: sync DB call blocks the event loop
@router.get("/clientes")
async def listar_clientes(db: Session = Depends(get_db)):
    return db.query(Cliente).all()

# Good: plain def lets FastAPI run it in a thread pool
@router.get("/clientes")
def listar_clientes(db: Session = Depends(get_db)):
    return db.query(Cliente).all()
```

Use `async def` with an async driver (`asyncpg`, `httpx.AsyncClient`, async SQLAlchemy session) when every I/O call in the path is actually awaited.

## Error Handling

Raise `HTTPException` for expected, per-request errors:

```python
@router.get("/clientes/{cliente_id}", response_model=ClienteRead)
def obter_cliente(cliente_id: int, db: Session = Depends(get_db)):
    cliente = db.get(Cliente, cliente_id)
    if cliente is None:
        raise HTTPException(status_code=404, detail="cliente não encontrado")
    return cliente
```

Use an exception handler for errors that recur across many routes instead of repeating `try/except` in each one:

```python
@app.exception_handler(ClienteNaoEncontrado)
def handle_cliente_nao_encontrado(request: Request, exc: ClienteNaoEncontrado):
    return JSONResponse(status_code=404, content={"detail": str(exc)})
```

## Testing

Use `TestClient` (sync) or `httpx.AsyncClient` (async) with `pytest`, overriding dependencies instead of hitting a real database or external service:

```python
from fastapi.testclient import TestClient

client = TestClient(app)


def test_criar_cliente():
    response = client.post("/api/v1/clientes", json={"nome": "João", "email": "joao@example.com"})
    assert response.status_code == 201
    assert response.json()["nome"] == "João"
```

See the [testing reference](references/testing.md) for overriding dependencies with `app.dependency_overrides`, testing async routes with `httpx.AsyncClient`, and fixtures for a per-test database.

## Tooling

* Package manager: `uv` or `pip-tools` for reproducible dependency pinning.
* ASGI server: `uvicorn` for development, `gunicorn -k uvicorn.workers.UvicornWorker` (or `uvicorn` with multiple workers) in production.
* Validation/serialization: Pydantic v2 — avoid mixing v1-style validators (`@validator`) with v2 (`@field_validator`).
* Lint/format: Ruff/Black.
* Docs: rely on the auto-generated OpenAPI schema (`/docs`, `/redoc`) — keep schemas and status codes accurate rather than writing parallel API docs by hand.
