# Task Queues with Procrastinate

Reference for async task queues using [Procrastinate](https://github.com/procrastinate-org/procrastinate): it uses PostgreSQL itself as the queue (via `LISTEN`/`NOTIFY` and `SELECT ... FOR UPDATE SKIP LOCKED`), **with no need for Celery, Redis, or RabbitMQ**. Prefer Procrastinate over Celery on Django projects that already run PostgreSQL — fewer moving parts to operate.

## Installation and Django Integration

```bash
pip install 'procrastinate[django]'
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    "procrastinate.contrib.django",
    ...
]
```

Procrastinate already creates a pre-configured `app` at `procrastinate.contrib.django.app`, using Django's database connection — there's no need to instantiate your own `App`. To customize it (e.g. register blueprints), use a callback:

```python
# settings.py
PROCRASTINATE_ON_APP_READY = "myapp.procrastinate.on_app_ready"
```

```python
# myapp/procrastinate.py
import procrastinate

def on_app_ready(app: procrastinate.App):
    app.add_tasks_from(some_blueprint)
```

## Migrations

Procrastinate ships native Django migrations — don't use the `procrastinate schema` command, just the normal Django flow:

```bash
python manage.py migrate
```

## Defining Tasks

Declare tasks in `tasks.py` inside the Django app, using the `app` imported from `procrastinate.contrib.django`:

```python
# apps/pedidos/tasks.py
from procrastinate.contrib.django import app


@app.task(queue="emails")
def enviar_email_confirmacao(pedido_pk: int) -> None:
    pedido = Pedido.objects.get(pk=pedido_pk)
    send_mail(
        "Pedido confirmado",
        f"Seu pedido {pedido.numero} foi confirmado.",
        "noreply@example.com",
        [pedido.cliente.email],
    )


# native async task support
@app.task(queue="emails")
async def enviar_email_async(pedido_pk: int) -> None:
    pedido = await Pedido.objects.aget(pk=pedido_pk)
    ...
```

Just like with Celery, pass only identifiers (not model instances) as arguments — the worker runs in a separate process.

## Deferring Tasks

```python
enviar_email_confirmacao.defer(pedido_pk=pedido.pk)

# async version
await enviar_email_async.defer_async(pedido_pk=pedido.pk)
```

## Running the Worker

```bash
python manage.py procrastinate worker
```

Use `queue=` on `@app.task` to route tasks to specific queues and run dedicated workers per queue (`python manage.py procrastinate worker -q emails`) when workloads have different priorities.

Verify the setup (database connection, etc.) with:

```bash
python manage.py procrastinate healthchecks
```

## Database Connections in Long-Running Tasks

The worker closes stale connections and resets queries between task runs automatically. For tasks with long stretches without database use, close the connection manually before the non-DB section:

```python
from django.db import close_old_connections

@app.task
def tarefa_longa():
    fazer_trabalho_com_banco()
    close_old_connections()
    fazer_trabalho_sem_banco_por_horas()
```

## Retries

Configure retries directly on the decorator:

```python
# fixed number of attempts
@app.task(retry=5)
def tarefa_instavel():
    ...

# fine-grained control: attempts, wait, and exceptions that trigger a retry
from procrastinate import RetryStrategy

@app.task(retry=RetryStrategy(
    max_attempts=10,
    exponential_wait=5,  # 5s, 25s, 125s...
    retry_exceptions={ConnectionError, TimeoutError},
))
def notificar_webhook(pedido_pk: int) -> None:
    ...
```

As with any retry system, make sure the task is idempotent.

## Periodic Tasks (Cron)

```python
@app.periodic(cron="0 3 * * *")  # every day at 3am
@app.task
def limpar_sessoes_expiradas(timestamp: int) -> None:
    ...
```

Periodic tasks are only scheduled while a worker is running — there's no separate "beat" process like in Celery.

## Admin and Observability

Since Procrastinate uses Django's ORM/migrations, the `ProcrastinateJob`, `ProcrastinateEvent`, `ProcrastinateWorker`, and `ProcrastinatePeriodicDefer` models are available in the Django Admin to inspect and reprocess jobs (including a "Retry Failed Job" button), with no need for an external tool like Celery's Flower.

## Testing

In tests, prefer calling the task function directly (without `.defer()`) or use Procrastinate's `InMemoryConnector` to capture deferred jobs without touching the real database:

```python
def test_confirmacao_dispara_email(self):
    enviar_email_confirmacao(pedido_pk=pedido.pk)
    self.assertEqual(len(mail.outbox), 1)
```
