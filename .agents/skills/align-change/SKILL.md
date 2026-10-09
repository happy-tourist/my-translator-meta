---
name: align-change
description: >-
  Orchestrates align passes on an active OpenSpec change: align-implementability-back
  (Rust), align-implementability-front (Flutter), then align-meta — each in its own
  subagent, each followed by the fix prompt «Добавь и поправь что считаешь нужным».
  Use when the user asks to align-change, run implementability + meta aligns on a
  change, or back → front → meta without pausing for review. Meta artifacts only —
  does not propose, apply, or edit runtime code.
---

# Align Change — align-back (Rust) → align-front (Flutter) → align-meta

Оркестратор выравнивания **уже существующего** OpenSpec change: реализуемость Rust/Flutter и качество meta. Parent **не** правит артефакты сам: делегирует шаги субагентам и гонит пайплайн до конца.

**Граница:** только meta (`openspec/`, docs/skills indexes при правках align). **Не** запускать `openspec-propose`, `openspec-apply-change`; не трогать runtime в `../my_translator_app` (`lib/`, `rust/`).

## Когда применять

- Пользователь запускает `align-change` / просит прогнать align’ы по активному change
- Нужен end-to-end: implementability-back → implementability-front → align-meta без промежуточного review
- Артефакты change уже есть (после propose / update) — этот скилл их **не** создаёт

Не подменять одиночный `align-implementability-back` / `front` / `align-meta`, если пользователь явно ограничил одной фазой.

После прерывания продолжение — снова `align-change` (читает `.align-change-state.yaml`).

## Жёсткое правило прерывания

**Прервать можно ТОЛЬКО если есть задачи которые невозможно выполнить без решения человека.**

Не останавливаться ради «лучше спросить», мелкой неоднозначности с default, промежуточного отчёта.

Останавливаться при: нет/несколько active change без однозначности; артефакты отсутствуют; секрет; конфликт без default; субагент вернул `HUMAN_BLOCKER`.

При стопе — записать state, кратко blocker; после решения — снова `align-change`.

## State-файл (для resume)

Путь: `openspec/changes/<name>/.align-change-state.yaml` (от корня meta).

```yaml
change: <name>
phase: align-back | align-front | align-meta | done
blocker: null | "<short reason>"
updated: "<ISO-8601>"
```

- Старт → `phase: align-back`
- После back → `align-front`
- После front → `align-meta`
- После meta → `done`

## Делегируемые скиллы

| Фаза | Skill |
|------|--------|
| 1. Align back (Rust) | [`.agents/skills/align-implementability-back/SKILL.md`](../align-implementability-back/SKILL.md) |
| 2. Align front (Flutter) | [`.agents/skills/align-implementability-front/SKILL.md`](../align-implementability-front/SKILL.md) |
| 3. Align meta | [`.agents/skills/align-meta/SKILL.md`](../align-meta/SKILL.md) |

## Вход / выбор change

1. Имя от пользователя, иначе из контекста.
2. Иначе `openspec list --json` — автовыбор при ровно **одном** active.
3. Иначе 0 или >1 без однозначности — **прервать**.

Объявить: `Using change: <name>`.

Пути runtime — через [`docs/projects-map.md`](../../../docs/projects-map.md) (+ local YAML): ключ `app` → `../my_translator_app` (`lib/` Flutter, `rust/` Rust).

## Workflow

```
Align-Change Progress:
- [ ] 1. Select active change (parent)
- [ ] 2. Align-implementability-back (Rust) → субагент (+ правка)
- [ ] 3. Align-implementability-front (Flutter) → субагент (+ правка)
- [ ] 4. Align-meta → субагент (+ правка)
- [ ] 5. Short final report
```

### Фазы 2–4

Каждый субагент: (1) выполнить дочерний скилл; (2) затем дословно **«Добавь и поправь что считаешь нужным»** — правки только **meta**; не runtime; не ждать подтверждения. Вернуть verdict + что поправлено или `HUMAN_BLOCKER`.

Порядок **фиксирован**: back → front → meta.

### Финальный отчёт

```markdown
# Align Change — итог

**Change:** <name>
**Align-back (Rust):** OK | HUMAN_BLOCKER | skipped
**Align-front (Flutter):** OK | HUMAN_BLOCKER | skipped
**Align-meta:** OK | HUMAN_BLOCKER | skipped
**Stopped:** нет | <причина>
**Resume:** снова `align-change` (если Stopped ≠ нет)
**Next:** review → `/openspec-apply-change` или `implement-change`
```

## Не делать

- Не пропускать фазы «чтобы быстрее».
- Не запускать propose / apply / implement-change / commit / archive.
- Не править runtime `lib/` / `rust/`.
- Не ослаблять safety (секреты, force push).
