# Документация экосистемы My Translator

Канон для AI-агентов и разработчиков. Исходники живут в sibling `../my_translator_app` (Flutter + Rust + FRB) — не внутри meta.

## Как читать

0. [projects-map.md](projects-map.md) — карта пакет → sibling-путь; local YAML.
1. OpenSpec: [`openspec/config.yaml`](../openspec/config.yaml) — context, rules, apply/verify files.
2. Пакетный контекст runtime: `{app}/AGENTS.md` → `../my_translator_app/AGENTS.md`
3. Skills и индекс агентов: [`.agents/AGENTS.md`](../.agents/AGENTS.md) (`.agents/skills/client/`, `.agents/skills/server/`, `openspec-*`).

Пути вида `{ключ}/…` и `my_translator_meta/docs/...` резолвить через [projects-map.md](projects-map.md) (+ `projects-map.local.yaml`) и child `project-map.md` (ключ `my_translator_meta`).

## Каталог docs (meta)

| Документ | Содержание |
|----------|------------|
| [projects-map.md](projects-map.md) | Карта пакет → sibling-путь (+ local YAML); таблица «Пути для OpenSpec / агентов» |

Общие code/test rules живут в пакетном `AGENTS.md` и skills (отдельного `docs/client/` / `docs/server/` нет).

OpenSpec: [`openspec/config.yaml`](../openspec/config.yaml); changes/specs в [`openspec/`](../openspec/).

## Sibling runtime

| Ключ projects-map | Путь (от meta) | Документация / контекст |
|-------------------|----------------|-------------------------|
| `app` | `../my_translator_app/` | `{app}/AGENTS.md` — продукт, стек, Flutter + Rust домены |

OpenSpec-артефакты: [`openspec/`](../openspec/) в **my_translator_meta** (не в app). Конфиг: [`openspec/config.yaml`](../openspec/config.yaml).

Always-on инструкции meta: [`AGENTS.md`](../AGENTS.md).
