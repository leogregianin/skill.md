---
name: django
description: Boas práticas e convenções para projetos Django. Use ao escrever ou revisar código Django — models, views, ORM, formulários, segurança, migrations e testes.
---

# Django

Skill genérico para escrever código Django seguindo boas práticas, mantendo consistência e evitando armadilhas comuns (N+1 queries, falhas de segurança, migrations mal geridas).

Princípio geral: priorize a biblioteca padrão do Python e os recursos nativos do Django sempre que resolverem bem o problema; só adicione uma dependência de terceiros quando o ganho for claro e o nativo for insuficiente.

## Referência Rápida

* Dependências: prefira sempre a biblioteca padrão/recursos nativos do Django; adicione um pacote de terceiros só quando necessário.
* Settings: separe configurações por ambiente e use `django-environ` para variáveis de ambiente; veja [Estrutura de Projeto](#estrutura-de-projeto).
* Models: defina sempre `__str__` e `related_name` explícito; veja [Models](#models).
* ORM: use `select_related`/`prefetch_related` para evitar N+1; veja [Consultas Otimizadas](#consultas-otimizadas-evitando-n1) e a [referência de ORM avançado](references/orm.pt-br.md) para `F()`/`Q()`, operações em lote, transações e managers customizados.
* Views: prefira Class-Based Views para CRUD padrão; use `get_object_or_404`; veja [Views](#views).
* Formulários: use `ModelForm` quando o form mapear para um model; veja [Forms](#forms).
* API: use o próprio Django (`JsonResponse`, `View`, `forms`) como primeira opção; recorra ao Django REST Framework apenas em último caso; veja [APIs](#apis).
* Segurança: não desative CSRF sem justificativa; `DEBUG = False` em produção; veja [Segurança](#segurança).
* Testes: use `TestCase`/`pytest-django`; veja [Testes](#testes).
* Tarefas assíncronas: use Procrastinate (fila baseada no próprio PostgreSQL, sem Celery/Redis) para trabalho fora do ciclo request/response; veja [Tarefas Assíncronas](#tarefas-assíncronas) e a [referência de filas com Procrastinate](references/tasks.pt-br.md) para integração com Django, retries e tarefas periódicas.

## Estrutura de Projeto

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

Use `django-environ` (ou similar) para carregar segredos e configurações de variáveis de ambiente em vez de hardcoded em `settings.py`.

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

* Use `related_name` em relacionamentos para evitar nomes genéricos como `cliente_set`.
* Prefira `TextChoices`/`IntegerChoices` para enums em vez de tuplas soltas de choices.
* Evite lógica de negócio pesada em métodos de model quando envolver múltiplos models — considere uma camada de serviço.

## Consultas Otimizadas (evitando N+1)

```python
# Ruim: N+1 queries
for pedido in Pedido.objects.all():
    print(pedido.cliente.nome)

# Bom: uma query com JOIN
for pedido in Pedido.objects.select_related("cliente"):
    print(pedido.cliente.nome)

# Para M2M ou reverse FK
clientes = Cliente.objects.prefetch_related("pedidos")
```

Use `only()`/`defer()` para limitar colunas carregadas em tabelas largas, e `annotate()`/`aggregate()` para cálculos no banco em vez de em Python.

Veja a [referência de ORM avançado](references/orm.pt-br.md) para expressões `F()`/`Q()`, `bulk_create`/`bulk_update`, transações (`atomic`, `on_commit`), managers customizados e índices/constraints.

## Views

Prefira Class-Based Views para CRUD padrão:

```python
class ClienteListView(ListView):
    model = Cliente
    paginate_by = 20


class ClienteDetailView(DetailView):
    model = Cliente
```

Use Function-Based Views para lógica simples ou muito customizada. Use `get_object_or_404` em vez de capturar `DoesNotExist` manualmente:

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

Sempre valide dados no `clean()`/`clean_<campo>()` do form, não apenas no template ou no frontend.

## APIs

Priorize os recursos nativos do Django (`JsonResponse`, `View`/`View` genérica, `forms` para validação) para expor endpoints JSON. Isso evita uma dependência extra e é suficiente para a maioria dos casos de CRUD simples:

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
            return JsonResponse({"detail": "não encontrado"}, status=404)
        return JsonResponse(cliente)

    def post(self, request: HttpRequest):
        form = ClienteForm(json.loads(request.body))
        if not form.is_valid():
            return JsonResponse(form.errors, status=400)
        cliente = form.save()
        return JsonResponse({"id": cliente.id}, status=201)
```

Use `django.forms.Form`/`ModelForm` para validar o corpo da requisição (o mesmo form pode servir para telas HTML e para a API), `QuerySet.values()`/`django.core.serializers` para serializar, e `JsonResponse`/`HttpResponseNotAllowed` para responder.

Recorra ao **Django REST Framework somente em último caso**, quando o projeto realmente precisar de recursos que o DRF resolve bem e que dariam trabalho relevante para reimplementar à mão: paginação/filtragem/ordenação padronizadas em muitos endpoints, serializers aninhados complexos, autenticação por token/OAuth com permissões refinadas, ou API pública grande com muitos endpoints seguindo o mesmo padrão.

```python
class ClienteSerializer(serializers.ModelSerializer):
    class Meta:
        model = Cliente
        fields = ["id", "nome", "email"]


class ClienteViewSet(viewsets.ModelViewSet):
    queryset = Cliente.objects.all()
    serializer_class = ClienteSerializer
```

Se optar por DRF, use `permission_classes` e `authentication_classes` explícitos em cada ViewSet; não confie apenas nas configurações globais quando o endpoint exigir regras diferentes.

## Segurança

* Não desative CSRF (`@csrf_exempt`) sem justificativa forte e revisão.
* Use o ORM para evitar SQL Injection; se precisar de SQL raw, use `params` em `cursor.execute()`, nunca f-strings.
* Mantenha `DEBUG = False` em produção e configure `ALLOWED_HOSTS` corretamente.
* Não use `{% autoescape off %}` sem necessidade real — evite XSS.
* Valide uploads de arquivo (tipo, tamanho) antes de salvar.

## Migrations

```bash
python manage.py makemigrations
python manage.py migrate
```

* Uma migration por mudança lógica; nunca edite migrations já aplicadas em produção.
* Revise o SQL gerado (`sqlmigrate`) antes de aplicar mudanças sensíveis em produção.
* Nomeie migrations manualmente quando o nome automático não for claro (`--name`).

## Testes

```python
class ClienteTests(TestCase):
    def test_criacao_cliente(self):
        cliente = Cliente.objects.create(nome="João", email="joao@example.com")
        self.assertEqual(str(cliente), "João")
```

Use `pytest-django` para testes mais rápidos e fixtures reutilizáveis; teste views e endpoints JSON com o `Client` nativo do Django (`response.json()`), recorrendo a `APIClient` apenas se o projeto já usar DRF.

## Tarefas Assíncronas

* Use [Procrastinate](https://github.com/procrastinate-org/procrastinate) para tarefas longas fora do ciclo request/response — a fila roda sobre o próprio PostgreSQL (`LISTEN`/`NOTIFY` + `SKIP LOCKED`), sem precisar de Celery, Redis ou RabbitMQ.
* Instale com `pip install 'procrastinate[django]'`, adicione `"procrastinate.contrib.django"` em `INSTALLED_APPS` e use as migrations nativas do Django (`python manage.py migrate`) — não use `procrastinate schema`.
* Views `async def` são suportadas desde o Django 4.1+, mas use apenas quando a lógica interna for realmente async-safe (ORM síncrono ainda bloqueia).
* Veja a [referência de filas com Procrastinate](references/tasks.pt-br.md) para definição de tasks, deferimento, worker, retries com `RetryStrategy` e tarefas periódicas (`@app.periodic`).

## Ferramentas Recomendadas

* `django-environ` para variáveis de ambiente
* `pytest-django` para testes
* `django-debug-toolbar` em desenvolvimento
* Ruff/Black para lint e formatação
* Django REST Framework — apenas como último recurso para APIs, quando os recursos nativos do Django (`JsonResponse`, `View`, `forms`) não forem suficientes; veja [APIs](#apis)
