---
name: align-code
description: >-
  Runs Flutter (lib/) and Rust (rust/) align + verify passes for my_translator_app
  against the active OpenSpec change, then prints a combined report. Use when the
  user asks for align-code, full-app align, or to audit Flutter and Rust against
  requirements and skill compliance in one pass (includes FRB binding races,
  async gaps before first UI bind, and layout/overlay risks that hide required chrome).
---

# Align Code — Flutter + Rust (один app)

Оркестратор: для слоёв **Flutter (`lib/`)** и **Rust (`rust/`)** внутри **одного** `my_translator_app` выполнить **align** (соответствие постановке) и **verify** (compliance / tooling), затем **единый отчёт**.

Скилл **только анализирует и отчитывается** — не правит runtime и не коммитит (кроме явной просьбы после отчёта).

В [`implement-change`](../implement-change/SKILL.md) оси **разведены**: фаза Align вызывает только `*-align-code`, фаза Verify — только `*-verify-code` (каждая со своим «Добавь и поправь…»). Этот скилл (`align-code`) по-прежнему делает **оба** прохода в одном запуске для standalone `/align-code`.

## Child skills (Flutter)

Для слоя **Flutter (`lib/`)** читать и следовать (только client skills этого meta):

| Ось | Skill |
|-----|--------|
| Align | [`client-align-code`](../client/client-align-code/SKILL.md) |
| Verify | [`client-verify-code`](../client/client-verify-code/SKILL.md) |

Topic skills (structure / errors / pages / styles / FRB / reader / test) — по релевантности диффа; индекс: [`.agents/AGENTS.md`](../../AGENTS.md).

## Child skills (Rust)

Для слоя **Rust (`rust/`)** читать и следовать (только server skills этого meta):

| Ось | Skill |
|-----|--------|
| Align | [`server-align-code`](../server/server-align-code/SKILL.md) |
| Verify | [`server-verify-code`](../server/server-verify-code/SKILL.md) |

Topic skills (structure / errors / test / config / frb-api / parse / tokenizematize / dictionary / translate) — по релевантности диффа; индекс: [`.agents/AGENTS.md`](../../AGENTS.md).

## Когда применять

- `align-code` / полный align Flutter+Rust
- Один проход: постановка + compliance на обоих слоях

Опционально сузить: `только flutter` / `только rust`.

## Репозитории

| Ключ | Path | Слой |
|------|------|------|
| meta | `.` | OpenSpec change (primary requirements) |
| `app` | `../my_translator_app` | `lib/` Flutter, `rust/` Rust |

Resolve via [`docs/projects-map.md`](../../../docs/projects-map.md). Перед работой — `{app}/AGENTS.md` если есть.

## Вход / OpenSpec change

Primary = активный OpenSpec change. Resolve: user name → dialog → single active → ask / `OpenSpec: none`.

Объявить: `Using OpenSpec change: <name>`.

## Hard Boundary

- Не править runtime/docs в standalone режиме.
- Align: read-only. Verify: запускать tooling (`flutter analyze`, `flutter test`, `cargo test` в `rust/`) и включать результаты; auto-fix только по явной просьбе / orchestrator override.

## Axes

**Flutter:** полный проход [`client-align-code`](../client/client-align-code/SKILL.md) (A–D) + [`client-verify-code`](../client/client-verify-code/SKILL.md); scores/tiers из их отчётов.

**Rust:** полный проход [`server-align-code`](../server/server-align-code/SKILL.md) (A–D) + [`server-verify-code`](../server/server-verify-code/SKILL.md); scores/tiers из их отчётов.

Severity: hard / warnings / recommendations. Score 0–10 per layer for постановка и код.

## Workflow

```
Align-Code Progress:
- [ ] 1. Resolve paths
- [ ] 2. Resolve OpenSpec change
- [ ] 3. Flutter: client-align-code + client-verify-code
- [ ] 4. Rust: server-align-code + server-verify-code
- [ ] 5. Combined report
```

Scope diff: branch vs merge-base + working tree в **app** (не meta как продукт). Meta — источник требований.

## Combined report (Russian)

```markdown
# Align Code — отчёт

Источник требований: OpenSpec change <name | none>
Scope: flutter + rust | flutter-only | rust-only

## Сводка
| Слой | Постановка | Код проекта | Тесты | Регрессии | Verify Violations | Verify Warnings |
|------|------------|-------------|-------|-----------|-------------------|-----------------|
| Flutter | N/10 | N/10 | READY\|BLOCKED | … | N | N |
| Rust | N/10 | N/10 | READY\|BLOCKED | … | N | N |

## Flutter (`lib/`)
### Align
…
### Verify
…

## Rust (`rust/`)
### Align
…
### Verify
…

## Что делать дальше
- hard / Violations → Warnings → Recommendations
- выполненные flutter/cargo команды и результаты
```

## Не делать

- Не подтягивать skills из чужих продуктов/репо — только `.agents/skills/client/` и `.agents/skills/server/` этого meta.
- Не смешивать findings слоёв без меток Flutter/Rust.
- Не запускать `commit` из этого скилла.
