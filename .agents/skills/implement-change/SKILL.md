---
name: implement-change
description: >-
  Orchestrates full implementation of an active OpenSpec change: apply task
  blocks via subagents, then align-code, optional verify-mock (only if a mock
  is attached and the change is mock-driven), check-changes, and commit. Use
  when the user asks to implement-change, fully implement an active change, or
  run apply → align → [mock] → check → commit end-to-end without pausing for
  review.
---

# Implement Change — apply → align → [mock] → check → commit

Оркестратор полного цикла по **активному** OpenSpec change. Parent **не** пишет runtime-код сам: делегирует субагентам.

Runtime: **один** app `../my_translator_app` (Flutter `lib/` + Rust `rust/`) + meta.

## Когда применять

- `implement-change` / полная реализация active change
- end-to-end без промежуточного review

Не подменять одиночный `openspec-apply-change`. Resume — `.implement-change-state.yaml`.

## Жёсткое правило прерывания

Прерывать **только** при необходимости решения человека. verify-mock без макета → `SKIPPED`, пайплайн продолжается.

## State-файл

`openspec/changes/<name>/.implement-change-state.yaml`:

```yaml
change: <name>
phase: apply | align | mock | check | commit | done
apply_block: "<N-or-null>"
blocker: null | "<short reason>"
updated: "<ISO-8601>"
```

## Делегируемые скиллы

| Фаза | Skill |
|------|--------|
| Apply | [`openspec-apply-change`](../openspec-apply-change/SKILL.md) |
| Align | [`align-code`](../align-code/SKILL.md) |
| Mock | [`verify-mock`](../verify-mock/SKILL.md) — если gate |
| Check | [`check-changes`](../check-changes/SKILL.md) |
| Commit | [`commit`](../commit/SKILL.md) |

Пути — [`docs/projects-map.md`](../../../docs/projects-map.md).

## Выбор change

Имя → контекст → единственный active → иначе HUMAN_BLOCKER. `Using change: <name>`.

## Gate verify-mock

G1: макет прикреплён (файл/вложение). G2: change про реализацию макета (Visual Spec / UI по скрину). Оба → mock; иначе SKIPPED → check.

## Workflow

```
Implement-Change Progress:
- [ ] 1. Select change + read apply skill
- [ ] 2. Status + apply instructions + contextFiles
- [ ] 3. Apply: каждый ##-блок tasks.md → отдельный субагент
- [ ] 4. Align-code → субагент (+ «Добавь и поправь…»)
- [ ] 5. Verify-mock → если gate; иначе SKIPPED
- [ ] 6. Check-changes → субагент (+ правка)
- [ ] 7. Commit → субагент (meta + app)
- [ ] 8. Short final report
```

### Apply blocks

Блок = `## N. …` в tasks.md. Субагент: следовать apply; только этот блок; `- [x]` сразу; не спрашивать; `HUMAN_BLOCKER` или `OK`. Последовательно (часто rust → flutter → bridge). Parent не пишет продукт-код.

### Align / Mock / Check

После отчёта — дословно **«Добавь и поправь что считаешь нужным»** (meta/runtime/skills по findings). Mock: пробросить абсолютные пути + `file_attachments`; правки UI в `lib/`.

### Commit

[`commit`](../commit/SKILL.md): stage → commit → push **meta + app**.

### Final report

```markdown
# Implement Change — итог
**Change:** <name>
**Apply:** N/M …
**Align:** OK | HUMAN_BLOCKER | skipped
**Verify-mock:** OK | SKIPPED (…) | HUMAN_BLOCKER
**Check-changes:** …
**Commit:** …
**Stopped:** нет | <причина>
**Resume:** снова `implement-change`
```

## Субагенты

Новый субагент на фазу; `run_in_background: false`; контекст: change, пути meta/app (`lib/`, `rust/`).

## Не делать

- Не пропускать align/check/commit.
- Не verify-mock без gate.
- Не archive/PR в этом скилле.
- Не ослаблять safety commit.
