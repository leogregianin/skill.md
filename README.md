# skill.md

A collection of generic `SKILL.md` files for use with Claude Code. Each skill documents best practices, conventions, and code patterns for a specific technology: frontmatter with `name`/`description`, a "Quick Reference" section, and sections with code examples, with deeper dives in `references/` when a topic warrants it.

> Leia em [Português](README.pt-br.md).

## Available skills

| Skill | Description | File |
|---|---|---|
| [Delphi](delphi/SKILL.md) | Object Pascal (VCL/FMX): memory management, naming, data access with FireDAC, unit organization, testing with DUnitX. | [`delphi/SKILL.md`](delphi/SKILL.md) · [`delphi/SKILL.pt-br.md`](delphi/SKILL.pt-br.md) |
| [Django](python-django/SKILL.md) | Models, advanced ORM, views, forms, APIs with plain Django (DRF only as a last resort), security, migrations, testing, task queues with Procrastinate. | [`python-django/SKILL.md`](python-django/SKILL.md) · [`python-django/SKILL.pt-br.md`](python-django/SKILL.pt-br.md) |
| [FastAPI](python-fastapi/SKILL.md) | Routers, Pydantic schemas, dependency injection, async, error handling, testing. | [`python-fastapi/SKILL.md`](python-fastapi/SKILL.md) · [`python-fastapi/SKILL.pt-br.md`](python-fastapi/SKILL.pt-br.md) |

Each skill has an English version (`SKILL.md`, canonical) and a Portuguese version (`SKILL.pt-br.md`), with the same content and section structure.

## Skill structure

```
<skill>/
├── SKILL.md            # English version (canonical)
├── SKILL.pt-br.md       # Portuguese version
└── references/          # optional deep dives, one file per topic
    ├── <topic>.md
    └── <topic>.pt-br.md
```

`SKILL.md` should stay lean and cover the essentials via "Quick Reference" + short sections with a code example. Topics that need more depth (e.g. FireDAC, DUnitX, advanced ORM, task queues) go into `references/`, linked from the corresponding `SKILL.md`.

## Using a skill in a project

Copy (or symlink) the desired skill folder into the target project's Claude Code skills directory, for example:

```bash
cp -r delphi /path/to/project/.claude/skills/delphi
# or
cp -r python-django /path/to/project/.claude/skills/python-django
# or
cp -r python-fastapi /path/to/project/.claude/skills/python-fastapi
```

Claude Code automatically loads each folder's `SKILL.md` and uses the frontmatter `description` to decide when to apply it.

## Adding a new skill

1. Create a folder named after the technology (`<technology>/`).
2. Write `SKILL.md` (English, canonical) and `SKILL.pt-br.md` (Portuguese) with the same frontmatter (`name`, `description`) and the same section structure.
3. If a topic needs more depth, create `references/<topic>.md` + `references/<topic>.pt-br.md` and link them from `SKILL.md`.
4. Update the available-skills table in this README (and in [README.pt-br.md](README.pt-br.md)).
