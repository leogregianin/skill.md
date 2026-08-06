# Advanced ORM

In-depth Django ORM reference: `F`/`Q` expressions, batch operations, transactions, custom managers, and database integrity.

## F() and Q() Expressions

Use `F()` to reference a field's value directly in the database, avoiding race conditions and unnecessary round-trips:

```python
from django.db.models import F

# Bad: reads the value in Python, decrements, then saves (race condition)
produto = Produto.objects.get(pk=1)
produto.estoque = produto.estoque - 1
produto.save()

# Good: the database performs the decrement atomically
Produto.objects.filter(pk=1).update(estoque=F("estoque") - 1)
```

Use `Q()` for `OR` queries, negation, or complex combinations that `filter()` alone can't express:

```python
from django.db.models import Q

Cliente.objects.filter(Q(nome__icontains="silva") | Q(email__icontains="silva"))
Cliente.objects.filter(~Q(ativo=True))
```

## Batch Operations

Use `bulk_create`/`bulk_update` to insert or update many records in few queries:

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

`bulk_create`/`bulk_update` don't call `save()` or trigger `pre_save`/`post_save` signals — don't use them when model logic depends on those hooks.

## Transactions

Wrap operations that must be atomic with `transaction.atomic()`:

```python
from django.db import transaction

with transaction.atomic():
    pedido = Pedido.objects.create(cliente=cliente)
    Item.objects.bulk_create([
        Item(pedido=pedido, produto=p, quantidade=q) for p, q in itens
    ])
```

Use `transaction.on_commit()` to schedule side effects (sending an email, triggering a Celery task) that should only happen if the transaction commits successfully:

```python
transaction.on_commit(lambda: enviar_email_confirmacao.delay(pedido.id))
```

## Custom Managers and QuerySets

Encapsulate recurring filters in a custom `QuerySet`/`Manager` instead of repeating `filter()` across multiple views:

```python
class ClienteQuerySet(models.QuerySet):
    def ativos(self):
        return self.filter(ativo=True)


class Cliente(models.Model):
    ativo = models.BooleanField(default=True)
    objects = ClienteQuerySet.as_manager()

# usage
Cliente.objects.ativos()
```

## Indexes and Constraints

Declare indexes and constraints in the model's `Meta`, so they're version-controlled alongside the code and applied via migration:

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

## Raw SQL (when necessary)

Always prefer the ORM. When raw SQL is unavoidable (very specific performance-critical queries), use `params` to avoid SQL Injection — never format the string manually:

```python
Cliente.objects.raw("SELECT * FROM clientes_cliente WHERE nome = %s", [nome])
```
