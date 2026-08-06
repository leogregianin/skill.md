# Filas de Tarefas com Procrastinate

Referência de filas de tarefas assíncronas usando [Procrastinate](https://github.com/procrastinate-org/procrastinate): usa o próprio PostgreSQL como fila (via `LISTEN`/`NOTIFY` e `SELECT ... FOR UPDATE SKIP LOCKED`), **sem precisar de Celery, Redis ou RabbitMQ**. Prefira Procrastinate a Celery em projetos Django que já usam PostgreSQL — menos peças móveis para operar.

## Instalação e Integração com Django

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

O Procrastinate já cria um `app` pré-configurado em `procrastinate.contrib.django.app`, usando a conexão de banco do Django — não é necessário instanciar seu próprio `App`. Para customizá-lo (ex.: registrar blueprints), use um callback:

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

O Procrastinate fornece migrations Django nativas — não use o comando `procrastinate schema`, apenas o fluxo normal do Django:

```bash
python manage.py migrate
```

## Definindo Tasks

Declare tasks em `tasks.py` dentro do app Django, usando o `app` importado de `procrastinate.contrib.django`:

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


# suporte nativo a task async
@app.task(queue="emails")
async def enviar_email_async(pedido_pk: int) -> None:
    pedido = await Pedido.objects.aget(pk=pedido_pk)
    ...
```

Assim como no Celery, passe apenas identificadores (não instâncias de model) como argumento — o worker roda em processo separado.

## Disparando (deferindo) Tasks

```python
enviar_email_confirmacao.defer(pedido_pk=pedido.pk)

# versão assíncrona
await enviar_email_async.defer_async(pedido_pk=pedido.pk)
```

## Executando o Worker

```bash
python manage.py procrastinate worker
```

Use `queue=` no `@app.task` para direcionar tasks a filas específicas e rodar workers dedicados por fila (`python manage.py procrastinate worker -q emails`) quando cargas de trabalho tiverem prioridades diferentes.

Verifique a configuração (conexão com banco, etc.) com:

```bash
python manage.py procrastinate healthchecks
```

## Conexões de Banco em Tasks Longas

O worker fecha conexões antigas e reseta queries entre execuções de tasks automaticamente. Para tasks com longos períodos sem uso do banco, feche a conexão manualmente antes do trecho sem I/O de banco:

```python
from django.db import close_old_connections

@app.task
def tarefa_longa():
    fazer_trabalho_com_banco()
    close_old_connections()
    fazer_trabalho_sem_banco_por_horas()
```

## Retries

Configure retries diretamente no decorator:

```python
# número fixo de tentativas
@app.task(retry=5)
def tarefa_instavel():
    ...

# controle fino: tentativas, espera e exceções que disparam retry
from procrastinate import RetryStrategy

@app.task(retry=RetryStrategy(
    max_attempts=10,
    exponential_wait=5,  # 5s, 25s, 125s...
    retry_exceptions={ConnectionError, TimeoutError},
))
def notificar_webhook(pedido_pk: int) -> None:
    ...
```

Assim como em qualquer sistema de retries, garanta que a task seja idempotente.

## Tarefas Periódicas (Cron)

```python
@app.periodic(cron="0 3 * * *")  # todo dia às 3h
@app.task
def limpar_sessoes_expiradas(timestamp: int) -> None:
    ...
```

Tarefas periódicas só são agendadas enquanto houver um worker rodando — não existe um "beat" separado como no Celery.

## Admin e Observabilidade

Como o Procrastinate usa Django ORM/migrations, os models `ProcrastinateJob`, `ProcrastinateEvent`, `ProcrastinateWorker` e `ProcrastinatePeriodicDefer` ficam disponíveis no Django Admin para inspecionar e reprocessar jobs (inclusive um botão "Retry Failed Job"), sem precisar de uma ferramenta externa como o Flower do Celery.

## Testes

Em testes, prefira chamar a função da task diretamente (sem `.defer()`) ou usar um `InMemoryConnector` do Procrastinate para capturar jobs deferidos sem tocar o banco real:

```python
def test_confirmacao_dispara_email(self):
    enviar_email_confirmacao(pedido_pk=pedido.pk)
    self.assertEqual(len(mail.outbox), 1)
```
