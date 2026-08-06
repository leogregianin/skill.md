# ORM Avançado

Referência aprofundada do ORM do Django: expressões `F`/`Q`, operações em lote, transações, managers customizados e integridade no banco.

## Expressões F() e Q()

Use `F()` para referenciar o valor de um campo do próprio banco, evitando race conditions e round-trips desnecessários:

```python
from django.db.models import F

# Ruim: lê o valor em Python, incrementa, e salva (race condition)
produto = Produto.objects.get(pk=1)
produto.estoque = produto.estoque - 1
produto.save()

# Bom: o banco faz o decremento atomicamente
Produto.objects.filter(pk=1).update(estoque=F("estoque") - 1)
```

Use `Q()` para consultas com `OR`, negação ou combinações complexas que `filter()` sozinho não expressa:

```python
from django.db.models import Q

Cliente.objects.filter(Q(nome__icontains="silva") | Q(email__icontains="silva"))
Cliente.objects.filter(~Q(ativo=True))
```

## Operações em Lote

Use `bulk_create`/`bulk_update` para inserir ou atualizar muitos registros em poucas queries:

```python
Cliente.objects.bulk_create([
    Cliente(nome="Ana", email="ana@example.com"),
    Cliente(nome="Bruno", email="bruno@example.com"),
])

clientes = list(Cliente.objects.filter(ativo=False))
for c in clientes:
    c.ativo = True
Cliente.objects.bulk_update(clientes, ["ativo"])
```

`bulk_create`/`bulk_update` não disparam `save()` nem sinais `pre_save`/`post_save` — não use quando a lógica do model depender desses hooks.

## Transações

Envolva operações que precisam ser atômicas com `transaction.atomic()`:

```python
from django.db import transaction

with transaction.atomic():
    pedido = Pedido.objects.create(cliente=cliente)
    Item.objects.bulk_create([
        Item(pedido=pedido, produto=p, quantidade=q) for p, q in itens
    ])
```

Use `transaction.on_commit()` para agendar efeitos colaterais (envio de e-mail, disparo de task Celery) que só devem ocorrer se a transação for confirmada com sucesso:

```python
transaction.on_commit(lambda: enviar_email_confirmacao.delay(pedido.id))
```

## Managers e QuerySets Customizados

Encapsule filtros recorrentes em um `QuerySet`/`Manager` customizado em vez de repetir `filter()` em várias views:

```python
class ClienteQuerySet(models.QuerySet):
    def ativos(self):
        return self.filter(ativo=True)


class Cliente(models.Model):
    ativo = models.BooleanField(default=True)
    objects = ClienteQuerySet.as_manager()

# uso
Cliente.objects.ativos()
```

## Índices e Constraints

Declare índices e constraints no `Meta` do model, para que fiquem versionados junto com o código e sejam aplicados via migration:

```python
class Pedido(models.Model):
    numero = models.CharField(max_length=20)
    cliente = models.ForeignKey(Cliente, on_delete=models.CASCADE)

    class Meta:
        indexes = [models.Index(fields=["numero"])]
        constraints = [
            models.UniqueConstraint(fields=["cliente", "numero"], name="uniq_pedido_por_cliente"),
        ]
```

## SQL Raw (quando necessário)

Prefira sempre o ORM. Quando SQL raw for inevitável (consultas muito específicas de performance), use `params` para evitar SQL Injection — nunca formate a string manualmente:

```python
Cliente.objects.raw("SELECT * FROM clientes_cliente WHERE nome = %s", [nome])
```
