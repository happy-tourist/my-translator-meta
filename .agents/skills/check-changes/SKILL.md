---
name: check-changes
description: >-
  Scans unstaged (and untracked) changes in my_translator_app (Flutter lib/ and
  Rust rust/), then recommends whether to add, update, delete, or split bloated
  skills using a scored bloat checklist (topic files preferred; score ≥5 →
  mandatory split plan), and whether to fix projects-map / project-map, docs, or
  AGENTS.md. Use when the user runs check-changes, asks to review unstaged app
  diffs for skill/docs/agent drift, wants a «что добавить / изменить / удалить /
  разбить» list, or asks which skills have grown too large to split.
---

# Check Changes — навыки, maps, docs, AGENTS

Проверить **незастейдженные** (и untracked) изменения в **одном** runtime-репо `my_translator_app` (`lib/` + `rust/`), сопоставить с каноном skills / projects-map / docs / `AGENTS.md`, выдать: **что добавить**, **изменить**, **удалить**, **разбить**.

Скилл **только анализирует и предлагает** — не правит файлы (для коммита — [`commit`](../commit/SKILL.md)).

## Когда применять

- `check-changes` / drift skills/docs после локальной разработки
- Явно спрашивают про раздувшиеся skills

## Репозитории

| Ключ | Path | Сканировать |
|------|------|-------------|
| `app` | `../my_translator_app` | **да** — unstaged + untracked (`lib/`, `rust/`, tests, pubspec, Cargo.toml, …) |
| meta | `.` | **нет** как источник diff; **да** как канон |

Resolve via [`docs/projects-map.md`](../../../docs/projects-map.md).

### Канон

| Область | Где |
|---------|-----|
| Skills index | `.agents/AGENTS.md` |
| Workspace skills | `.agents/skills/` |
| Flutter client skills | `.agents/skills/client/` |
| Projects map | `docs/projects-map.md`, local YAML |
| Child map | `{app}/project-map.md` |
| AGENTS | meta + `.agents/AGENTS.md` + `{app}/AGENTS.md` |

## Scope diff

Только unstaged/untracked в **app**:

```bash
git status -sb
git status --porcelain
git diff
git ls-files --others --exclude-standard
```

Staged — кратко упомянуть, не разбирать. Если app чист — «незастейдженных изменений нет».

## Workflow

```
Check-Changes Progress:
- [ ] 1. Resolve paths
- [ ] 2. Collect unstaged/untracked (app)
- [ ] 3. Summarize themes (Flutter vs Rust vs bridge)
- [ ] 4. Map → skills / maps / docs / AGENTS
- [ ] 5. Bloat scoring
- [ ] 6. Decide add/change/delete/split
- [ ] 7. Report
```

### Themes examples

- новый экран/виджет / route
- новый FRB API / rust module
- смена контракта Flutter↔Rust
- persistence / NLP / subtitle pipeline
- только стиль / тесты без нового паттерна

Игнорировать форматирование, lockfile-only, generated FRB noise unless API surface changed.

### Bloat scoring

Как в исходном скилле: +2/+3 за размер ядра, description dump, ≥3 ортогональные темы, отсутствие Specialized Topics, дубли. Порог ≥5 → обязательно «Что разбить» (A topic files / B sibling skill). Process-skills (`align-*`, `openspec-*`) — только сигналы dump/дубль, не длина alone.

Скан: `.agents/skills/*/SKILL.md` и `.agents/skills/client/*/SKILL.md` (+ topic `*.md` при bloat).

### Decide

- **Add skill** — устойчивый новый паттерн агента (flutter-*/rust-*/work-with-*).
- **Change** — устаревший API/пути относительно diff.
- **Split** — score ≥3 (обязательно ≥5).
- **Delete** — домен убран и кода нет; почистить AGENTS.
- **Maps/docs/AGENTS** — пути, стек Flutter+Rust, индексы skills.

Не предлагать runtime-правки — только agent/docs/map.

## Report

```markdown
## Сводка изменений
| Repo | Unstaged / untracked | Темы |
|------|----------------------|------|
| app (lib/) | … | … |
| app (rust/) | … | … |

## Что добавить
## Что изменить
## Что удалить
## Что разбить
## Вне scope / заметки
```

## Не делать

- Не редактировать skills в этом прогоне без отдельной просьбы.
- Не анализировать staged/branch commits как основной scope.
- Не сканировать два sibling-репо client/server — только **app**.
- Не путать с `align-code` / `commit`.
