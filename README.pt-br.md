# skill.md

Coleção de `SKILL.md` genéricos para uso com o Claude Code. Cada skill documenta boas práticas, convenções e padrões de código para uma tecnologia específica: frontmatter com `name`/`description`, uma seção "Quick Reference" e seções com exemplos de código, com aprofundamentos em `references/` quando o tópico exige mais detalhe.

> Read this in [English](README.md).

## Skills disponíveis

| Skill | Descrição | Arquivo |
|---|---|---|
| [Delphi](delphi/SKILL.pt-br.md) | Object Pascal (VCL/FMX): gerenciamento de memória, nomenclatura, acesso a dados com FireDAC, organização de units, testes com DUnitX. | [`delphi/SKILL.md`](delphi/SKILL.md) · [`delphi/SKILL.pt-br.md`](delphi/SKILL.pt-br.md) |
| [Django](python-django/SKILL.pt-br.md) | Models, ORM avançado, views, formulários, APIs com Django puro (DRF só como último recurso), segurança, migrations, testes, filas de tarefas com Procrastinate. | [`python-django/SKILL.md`](python-django/SKILL.md) · [`python-django/SKILL.pt-br.md`](python-django/SKILL.pt-br.md) |
| [FastAPI](python-fastapi/SKILL.pt-br.md) | Routers, schemas Pydantic, injeção de dependência, async, tratamento de erros, testes. | [`python-fastapi/SKILL.md`](python-fastapi/SKILL.md) · [`python-fastapi/SKILL.pt-br.md`](python-fastapi/SKILL.pt-br.md) |

Cada skill tem uma versão em inglês (`SKILL.md`, canônica) e uma em português (`SKILL.pt-br.md`), com o mesmo conteúdo e estrutura de seções.

## Estrutura de um skill

```
<skill>/
├── SKILL.md            # versão em inglês (canônica)
├── SKILL.pt-br.md       # versão em português
└── references/          # aprofundamentos opcionais, um arquivo por tópico
    ├── <topico>.md
    └── <topico>.pt-br.md
```

O `SKILL.md` deve ficar enxuto e cobrir o essencial via "Quick Reference" + seções curtas com exemplo de código. Tópicos que exigem mais profundidade (ex.: FireDAC, DUnitX, ORM avançado, filas de tarefas) vão para `references/`, linkados a partir do `SKILL.md` correspondente.

## Como usar em um projeto

Copie (ou crie um link simbólico para) a pasta do skill desejado dentro da estrutura de skills do Claude Code no projeto de destino, por exemplo:

```bash
cp -r delphi /caminho/do/projeto/.claude/skills/delphi
# ou
cp -r python-django /caminho/do/projeto/.claude/skills/python-django
# ou
cp -r python-fastapi /caminho/do/projeto/.claude/skills/python-fastapi
```

O Claude Code carrega o `SKILL.md` de cada pasta automaticamente e usa a `description` do frontmatter para decidir quando aplicá-lo.

## Adicionando um novo skill

1. Crie uma pasta com o nome da tecnologia (`<tecnologia>/`).
2. Escreva `SKILL.md` (inglês, canônico) e `SKILL.pt-br.md` (português) com o mesmo frontmatter (`name`, `description`) e a mesma estrutura de seções.
3. Se algum tópico precisar de mais profundidade, crie `references/<topico>.md` + `references/<topico>.pt-br.md` e linke a partir do `SKILL.md`.
4. Atualize a tabela de skills disponíveis neste README (e no [README.md](README.md)).
