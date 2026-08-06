# FastAPI: Testing Reference

## Overriding dependencies

Replace real dependencies (DB session, external API client, current user) with test doubles via `app.dependency_overrides`, instead of hitting a real database or network:

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

Clear overrides after the test (or scope them to a fixture) so they don't leak into other tests:

```python
@pytest.fixture
def client():
    app.dependency_overrides[get_db] = override_get_db
    yield TestClient(app)
    app.dependency_overrides.clear()
```

## Testing async routes

Use `httpx.AsyncClient` with `ASGITransport` for routes that must actually run inside an event loop (e.g. exercising async DB drivers):

```python
import pytest
from httpx import AsyncClient, ASGITransport


@pytest.mark.anyio
async def test_criar_cliente():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        response = await client.post("/api/v1/clientes", json={"nome": "João", "email": "joao@example.com"})
        assert response.status_code == 201
```

`TestClient` works for both sync and async path operations for simple cases — reach for `AsyncClient` when the test itself needs to run concurrently with other async code.

## Per-test database

Use a fixture that creates and tears down a fresh database (or transaction) per test, rather than sharing state between tests:

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

## Testing validation errors

Assert on the 422 response body FastAPI generates automatically for invalid payloads, instead of only testing the happy path:

```python
def test_criar_cliente_email_invalido():
    response = client.post("/api/v1/clientes", json={"nome": "João", "email": "not-an-email"})
    assert response.status_code == 422
```
