---
name: fastapi
description: Boas práticas e convenções para projetos FastAPI. Use ao escrever ou revisar código FastAPI — rotas, schemas Pydantic, injeção de dependência, async, tratamento de erros e testes.
---

# FastAPI

Skill genérico para escrever código FastAPI seguindo boas práticas, mantendo consistência e evitando armadilhas comuns (chamadas bloqueantes em rotas async, validação duplicada, respostas sem tipo).

## Referência Rápida

* Estrutura de projeto: separe routers, schemas e dependências em módulos distintos; veja [Estrutura de Projeto](#estrutura-de-projeto).
* Schemas: use models Pydantic para validar request/response, nunca dicts crus; veja [Schemas Pydantic](#schemas-pydantic).
* Rotas: use `APIRouter` por recurso com `prefix`/`tags`; veja [Routers](#routers).
* Injeção de Dependência: use `Depends()` para lógica compartilhada (sessão de banco, autenticação, paginação); veja [Injeção de Dependência](#injeção-de-dependência) e a [referência de dependências](references/dependencies.pt-br.md) para sub-dependências, dependências com `yield` e `Security()`.
* Async: use `async def` apenas quando o corpo for realmente não bloqueante; execute I/O bloqueante em um thread pool; veja [Async](#async).
* Tratamento de erros: use `HTTPException` para erros esperados; use exception handlers para casos transversais; veja [Tratamento de Erros](#tratamento-de-erros).
* Testes: use `TestClient`/`httpx.AsyncClient` com `pytest`; veja [Testes](#testes) e a [referência de testes](references/testing.pt-br.md) para overrides de dependência e clientes async.

## Estrutura de Projeto

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

Mantenha as funções de path operation enxutas — validação fica nos schemas Pydantic, lógica de negócio em uma camada de serviço e persistência em models/repositórios.

## Schemas Pydantic

Separe schemas de entrada e saída em vez de reutilizar um único model para tudo:

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

Use `response_model` em toda path operation para que o FastAPI filtre e documente o formato de saída:

```python
@router.post("/clientes", response_model=ClienteRead, status_code=201)
def criar_cliente(payload: ClienteCreate) -> Cliente:
    ...
```

## Routers

Agrupe path operations relacionadas com `APIRouter`, e monte-as em `main.py`:

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

## Injeção de Dependência

Use `Depends()` para tudo que for compartilhado entre path operations — sessão de banco, usuário atual, parâmetros de paginação:

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

Veja a [referência de dependências](references/dependencies.pt-br.md) para sub-dependências, dependências baseadas em `yield` com cleanup, cache com `Depends(..., use_cache=True)` e `Security()` para escopos de autenticação.

## Async

Só declare uma path operation como `async def` se o corpo dela realmente não bloquear o event loop. Uma chamada bloqueante (driver de banco síncrono, `requests`, processamento pesado de CPU) dentro de uma rota `async def` trava todas as outras requisições:

```python
# Ruim: chamada síncrona ao banco bloqueia o event loop
@router.get("/clientes")
async def listar_clientes(db: Session = Depends(get_db)):
    return db.query(Cliente).all()

# Bom: def simples deixa o FastAPI executar em um thread pool
@router.get("/clientes")
def listar_clientes(db: Session = Depends(get_db)):
    return db.query(Cliente).all()
```

Use `async def` com um driver async (`asyncpg`, `httpx.AsyncClient`, sessão async do SQLAlchemy) quando toda chamada de I/O da rota for de fato aguardada (`await`).

## Tratamento de Erros

Use `HTTPException` para erros esperados, específicos de uma requisição:

```python
@router.get("/clientes/{cliente_id}", response_model=ClienteRead)
def obter_cliente(cliente_id: int, db: Session = Depends(get_db)):
    cliente = db.get(Cliente, cliente_id)
    if cliente is None:
        raise HTTPException(status_code=404, detail="cliente não encontrado")
    return cliente
```

Use um exception handler para erros que se repetem em várias rotas em vez de repetir `try/except` em cada uma:

```python
@app.exception_handler(ClienteNaoEncontrado)
def handle_cliente_nao_encontrado(request: Request, exc: ClienteNaoEncontrado):
    return JSONResponse(status_code=404, content={"detail": str(exc)})
```

## Testes

Use `TestClient` (síncrono) ou `httpx.AsyncClient` (async) com `pytest`, sobrescrevendo dependências em vez de acessar um banco real ou serviço externo:

```python
from fastapi.testclient import TestClient

client = TestClient(app)


def test_criar_cliente():
    response = client.post("/api/v1/clientes", json={"nome": "João", "email": "joao@example.com"})
    assert response.status_code == 201
    assert response.json()["nome"] == "João"
```

Veja a [referência de testes](references/testing.pt-br.md) para sobrescrever dependências com `app.dependency_overrides`, testar rotas async com `httpx.AsyncClient` e fixtures para um banco por teste.

## Ferramentas

* Gerenciador de pacotes: `uv` ou `pip-tools` para fixação reprodutível de dependências.
* Servidor ASGI: `uvicorn` em desenvolvimento, `gunicorn -k uvicorn.workers.UvicornWorker` (ou `uvicorn` com múltiplos workers) em produção.
* Validação/serialização: Pydantic v2 — evite misturar validadores estilo v1 (`@validator`) com v2 (`@field_validator`).
* Lint/formatação: Ruff/Black.
* Docs: use o schema OpenAPI gerado automaticamente (`/docs`, `/redoc`) — mantenha schemas e status codes corretos em vez de escrever documentação de API paralela à mão.
