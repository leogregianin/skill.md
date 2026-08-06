# FastAPI: Referência de Testes

## Sobrescrevendo dependências

Substitua dependências reais (sessão de banco, cliente de API externa, usuário atual) por dublês de teste via `app.dependency_overrides`, em vez de acessar um banco real ou a rede:

```python
from myproject.api.deps import get_db


def override_get_db():
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()


app.dependency_overrides[get_db] = override_get_db
```

Limpe os overrides após o teste (ou escopo-os a uma fixture) para que não vazem para outros testes:

```python
@pytest.fixture
def client():
    app.dependency_overrides[get_db] = override_get_db
    yield TestClient(app)
    app.dependency_overrides.clear()
```

## Testando rotas async

Use `httpx.AsyncClient` com `ASGITransport` para rotas que precisam realmente rodar dentro de um event loop (por exemplo, exercitando drivers de banco async):

```python
import pytest
from httpx import AsyncClient, ASGITransport


@pytest.mark.anyio
async def test_criar_cliente():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        response = await client.post("/api/v1/clientes", json={"nome": "João", "email": "joao@example.com"})
        assert response.status_code == 201
```

`TestClient` funciona tanto para path operations síncronas quanto async em casos simples — use `AsyncClient` quando o próprio teste precisar rodar concorrentemente com outro código async.

## Banco de dados por teste

Use uma fixture que cria e desfaz um banco (ou transação) novo a cada teste, em vez de compartilhar estado entre testes:

```python
@pytest.fixture
def db_session():
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection)
    yield session
    session.close()
    transaction.rollback()
    connection.close()
```

## Testando erros de validação

Verifique o corpo da resposta 422 que o FastAPI gera automaticamente para payloads inválidos, em vez de testar apenas o caminho feliz:

```python
def test_criar_cliente_email_invalido():
    response = client.post("/api/v1/clientes", json={"nome": "João", "email": "not-an-email"})
    assert response.status_code == 422
```
