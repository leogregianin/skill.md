---
name: django
description: Best practices and conventions for Django projects. Use when writing or reviewing Django code — models, views, ORM, forms, security, migrations, and testing.
---

# Django

Generic skill for writing Django code following best practices, keeping consistency and avoiding common pitfalls (N+1 queries, security issues, poorly managed migrations).

General principle: prioritize the Python standard library and Django's built-in features whenever they solve the problem well; only add a third-party dependency when the payoff is clear and the built-in option falls short.

## Quick Reference

* Dependencies: always prefer the standard library/Django's built-in features; add a third-party package only when necessary.
* Settings: split configuration per environment and use `django-environ` for environment variables; see [Project Structure](#project-structure).
* Models: always define `__str__` and explicit `related_name`; see [Models](#models).
* ORM: use `select_related`/`prefetch_related` to avoid N+1 queries; see [Optimized Queries](#optimized-queries-avoiding-n1) and the [advanced ORM reference](references/orm.md) for `F()`/`Q()`, batch operations, transactions, and custom managers.
* Views: prefer Class-Based Views for standard CRUD; use `get_object_or_404`; see [Views](#views).
* Forms: use `ModelForm` when the form maps to a model; see [Forms](#forms).
* API: use plain Django (`JsonResponse`, `View`, `forms`) as the first option; reach for Django REST Framework only as a last resort; see [APIs](#apis).
* Security: don't disable CSRF without justification; `DEBUG = False` in production; see [Security](#security).
* Testing: use `TestCase`/`pytest-django`; see [Testing](#testing).
* Async tasks: use Procrastinate (a PostgreSQL-native queue, no Celery/Redis) for work outside the request/response cycle; see [Async Tasks](#async-tasks) and the [Procrastinate task queue reference](references/tasks.md) for Django integration, retries, and periodic tasks.

## Project Structure

```
myproject/
├── manage.py
├── myproject/
│   ├── settings/
│   │   ├── base.py
│   │   ├── dev.py
│   │   └── prod.py
│   ├── urls.py
│   └── wsgi.py / asgi.py
└── apps/
    └── clientes/
        ├── models.py
        ├── views.py
        ├── urls.py
        ├── forms.py
        ├── admin.py
        ├── tests.py
        └── migrations/
```

Use `django-environ` (or similar) to load secrets and settings from environment variables instead of hardcoding them in `settings.py`.

## Models

```python
class Cliente(models.Model):
    nome = models.CharField(max_length=255)
    email = models.EmailField(unique=True)
    criado_em = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["nome"]

    def __str__(self):
        return self.nome
```

* Use `related_name` on relationships to avoid generic names like `cliente_set`.
* Prefer `TextChoices`/`IntegerChoices` for enums instead of loose choice tuples.
* Avoid heavy business logic in model methods when it spans multiple models — consider a service layer.

## Optimized Queries (avoiding N+1)

```python
# Bad: N+1 queries
for pedido in Pedido.objects.all():
    print(pedido.cliente.nome)

# Good: a single query with JOIN
for pedido in Pedido.objects.select_related("cliente"):
    print(pedido.cliente.nome)

# For M2M or reverse FK
clientes = Cliente.objects.prefetch_related("pedidos")
```

Use `only()`/`defer()` to limit loaded columns on wide tables, and `annotate()`/`aggregate()` for database-side calculations instead of doing them in Python.

See the [advanced ORM reference](references/orm.md) for `F()`/`Q()` expressions, `bulk_create`/`bulk_update`, transactions (`atomic`, `on_commit`), custom managers, and indexes/constraints.

## Views

Prefer Class-Based Views for standard CRUD:

```python
class ClienteListView(ListView):
    model = Cliente
    paginate_by = 20


class ClienteDetailView(DetailView):
    model = Cliente
```

Use Function-Based Views for simple or heavily customized logic. Use `get_object_or_404` instead of manually catching `DoesNotExist`:

```python
def detalhe_cliente(request, pk):
    cliente = get_object_or_404(Cliente, pk=pk)
    return render(request, "clientes/detalhe.html", {"cliente": cliente})
```

## Forms

```python
class ClienteForm(forms.ModelForm):
    class Meta:
        model = Cliente
        fields = ["nome", "email"]
```

Always validate data in the form's `clean()`/`clean_<field>()`, not just in the template or frontend.

## APIs

Prioritize Django's built-in tools (`JsonResponse`, `View`, `forms` for validation) to expose JSON endpoints. This avoids an extra dependency and is enough for most simple CRUD cases:

```python
import json

from django.core.exceptions import ObjectDoesNotExist
from django.http import HttpRequest, JsonResponse
from django.views import View
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator


@method_decorator(csrf_exempt, name="dispatch")
class ClienteApiView(View):
    def get(self, request: HttpRequest, pk: int | None = None):
        if pk is None:
            clientes = list(Cliente.objects.values("id", "nome", "email"))
            return JsonResponse(clientes, safe=False)
        try:
            cliente = Cliente.objects.values("id", "nome", "email").get(pk=pk)
        except ObjectDoesNotExist:
            return JsonResponse({"detail": "not found"}, status=404)
        return JsonResponse(cliente)

    def post(self, request: HttpRequest):
        form = ClienteForm(json.loads(request.body))
        if not form.is_valid():
            return JsonResponse(form.errors, status=400)
        cliente = form.save()
        return JsonResponse({"id": cliente.id}, status=201)
```

Use `django.forms.Form`/`ModelForm` to validate the request body (the same form can back both HTML pages and the API), `QuerySet.values()`/`django.core.serializers` to serialize, and `JsonResponse`/`HttpResponseNotAllowed` to respond.

Reach for **Django REST Framework only as a last resort**, when the project genuinely needs features DRF handles well and that would take real effort to hand-roll: standardized pagination/filtering/ordering across many endpoints, complex nested serializers, token/OAuth authentication with fine-grained permissions, or a large public API with many endpoints following the same pattern.

```python
class ClienteSerializer(serializers.ModelSerializer):
    class Meta:
        model = Cliente
        fields = ["id", "nome", "email"]


class ClienteViewSet(viewsets.ModelViewSet):
    queryset = Cliente.objects.all()
    serializer_class = ClienteSerializer
```

If you do choose DRF, use explicit `permission_classes` and `authentication_classes` on each ViewSet; don't rely solely on global settings when an endpoint needs different rules.

## Security

* Don't disable CSRF (`@csrf_exempt`) without strong justification and review.
* Use the ORM to avoid SQL Injection; if raw SQL is needed, use `params` in `cursor.execute()`, never f-strings.
* Keep `DEBUG = False` in production and configure `ALLOWED_HOSTS` correctly.
* Don't use `{% autoescape off %}` without real need — avoid XSS.
* Validate file uploads (type, size) before saving.

## Migrations

```bash
python manage.py makemigrations
python manage.py migrate
```

* One migration per logical change; never edit migrations already applied in production.
* Review the generated SQL (`sqlmigrate`) before applying sensitive changes in production.
* Name migrations manually when the auto-generated name isn't clear (`--name`).

## Testing

```python
class ClienteTests(TestCase):
    def test_criacao_cliente(self):
        cliente = Cliente.objects.create(nome="João", email="joao@example.com")
        self.assertEqual(str(cliente), "João")
```

Use `pytest-django` for faster tests and reusable fixtures; test views and JSON endpoints with Django's built-in `Client` (`response.json()`), reaching for `APIClient` only if the project already uses DRF.

## Async Tasks

* Use [Procrastinate](https://github.com/procrastinate-org/procrastinate) for long-running tasks outside the request/response cycle — the queue runs on PostgreSQL itself (`LISTEN`/`NOTIFY` + `SKIP LOCKED`), with no need for Celery, Redis, or RabbitMQ.
* Install with `pip install 'procrastinate[django]'`, add `"procrastinate.contrib.django"` to `INSTALLED_APPS`, and use Django's native migrations (`python manage.py migrate`) — don't use `procrastinate schema`.
* `async def` views are supported since Django 4.1+, but use them only when the internal logic is truly async-safe (the synchronous ORM still blocks).
* See the [Procrastinate task queue reference](references/tasks.md) for task definitions, deferring, the worker, retries with `RetryStrategy`, and periodic tasks (`@app.periodic`).

## Recommended Tooling

* `django-environ` for environment variables
* `pytest-django` for testing
* `django-debug-toolbar` in development
* Ruff/Black for linting and formatting
* Django REST Framework — only as a last resort for APIs, when Django's built-in tools (`JsonResponse`, `View`, `forms`) aren't enough; see [APIs](#apis)
