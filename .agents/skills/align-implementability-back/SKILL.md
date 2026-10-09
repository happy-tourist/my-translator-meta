---
name: align-implementability-back
description: >-
  Use when judging whether my_translator_meta Rust/core claims are implementable
  (possibility to build only — not meta doc quality, not Flutter UI layout).
  Against current my_translator_app/rust (crates, FRB API, persistence, NLP/translate
  core via projects-map). Also warn when Flutter/UI docs in the change family
  describe data/actions with no Rust/API claim in meta (`[fe-only-without-be]`).
  Classifies aligns/extends/conflicts/unproven; verdict FEASIBLE|PARTIAL|INFEASIBLE|UNKNOWN.
  Finding card: file + lines + issue + fix. Not Flutter UI realizability
  (align-implementability-front) or meta quality (align-meta). Does not edit files.
---

# Align Implementability (Backend / Rust)

Read-only **Rust implementability** audit: can the **my_translator_meta** change be built **as written** against **my_translator_app/rust**? Report in Russian.

**Scope — only possibility to implement.** Completeness of meta → **`align-meta`**. Flutter screens fit → **`align-implementability-front`** (owns `[be-without-fe]`).

Новый API/тип/алгоритм, которого ещё нет в app HEAD, но есть ясный аналог для **extends** — нормальная реализуемость, не hard gap. Hard только при **conflicts** с доказанным контрактом/кодом.

Output **one scope** with tiers hard → warnings → recommendations:

1. **Rust implementability** — claims vs `rust/` evidence (`aligns` / `extends` / `conflicts` / `unproven`).
2. **Flutter→Rust coverage in meta** — UI claims without Rust/API counterpart (`[fe-only-without-be]`).

Every finding is a **card**: file + lines + issue + fix (or «неясно как делать»).

## FE-only-without-BE

| Ситуация | Суждение |
|----------|----------|
| UI/Flutter docs описывают данные/действие, а в meta нет Rust/API/FRB claim | Warning `[fe-only-without-be]` — не hard |
| В UI есть поле, нет в app HEAD, но meta Rust claim есть | Не `[fe-only-without-be]`; **extends** |
| Ветка явно rust-only без UI docs | Не сканируй UI массово |

**Потребность в Rust для метки:** поле/действие через FRB, парсинг/леммы/перевод/словарь, persistence, error codes с API. **Не** сюда: layout/widgets/i18n UX → front.

## Runtime roots

- **Claims:** meta artifacts asserting **Rust** runtime: crates, modules, FRB API signatures, SQLite/diesel, subtitle/NLP/translate pipelines, rust skills.
- **Evidence:** projects-map `app` → `../my_translator_app/`, primary `rust/`. Skills: `.agents/skills/` (rust/app topics when present).
- Do **not** treat app working tree as the primary diff to “fix”.

## Scope filter

**Include:** crate/module layout; FRB/`api` functions & types; persistence; core algorithms (parse, lemmatize, translate, vocab); error/result contracts; for coverage scan — UI actions implying Rust API.

**Exclude:** Flutter routes/widgets as UI implementability; `[be-without-fe]` (front skill).

## Hard Boundary

Do not edit/create/delete/commit files. Verdict evidence only.

## Input And Store

Optional OpenSpec change name. Else audit change artifacts + rust evidence. Store: `openspec store list --json` if named; else `my_translator_meta/openspec/`.

## Load Branch Work (claims)

В meta (read-only):

```bash
git status --short
git diff
git diff --cached
git branch -vv
```

Merge-base: `origin/main` / `origin/master` / `main` / `master`. Collect commits + staged + unstaged + untracked; filter to Rust-relevant claims. Then UI→Rust scan for `[fe-only-without-be]`.

## Load Sibling Evidence (Rust)

For each claim: resolve `app`, grep/read `rust/src`, Cargo.toml, tests, FRB api. Missing clone → Warning `[sibling-missing]`. Before deep reads — `{app}/AGENTS.md` if present.

## Axis — judgment

Compare: API surface; types/errors; persistence; algorithm fit on existing crates; edge paths. Classify **aligns|extends|conflicts|unproven**. Hard gap only with concrete evidence + misstatement + required parity. Absence in HEAD ≠ hard if clear **extends**.

Score **Код проекта (rust): N/10** only from hard items. `[fe-only-without-be]` не снижает.

| Вердикт | Когда |
|---------|--------|
| `FEASIBLE` | No hard conflicts; aligns/extends |
| `PARTIAL` | Mixed / critical unproven |
| `INFEASIBLE` | Hard conflicts/gap — не «ещё нет в HEAD» при extends |
| `UNKNOWN` | Not enough evidence — list what was searched |

## Finding card

```markdown
### <N>. <заголовок>
- Метка: `[gap|code-only|warning|recommendation|fe-only-without-be]` (+ `aligns|extends|conflicts|unproven`)
- Файл: [`path`](path) — **L12–L18**
- Связанные: [`../my_translator_app/rust/...`](../my_translator_app/rust/...) — **L…**
- Что не так: …
- Предложение: … **или** Неясно как делать: A/B
```

## Report (Russian)

```markdown
## Вердикт
- Код проекта (rust): N/10 — …
- Реализуемость rust: FEASIBLE | PARTIAL | INFEASIBLE | UNKNOWN
- Режим: docs/openspec | skills | mixed | rust-only
- Источник claims: …
- Работа в ветке meta: …
- Flutter↔Rust в meta: `[fe-only-without-be]` …; UI-fit → align-implementability-front

## Claims (meta, rust)
## Доказательная база (app/rust)
## Замечания (hard / warnings / recommendations)
## Карта аналогов (rust)
## Что делать дальше
```

Never apply fixes.

## Not This Skill

| Skill | Difference |
|-------|------------|
| `align-implementability-front` | Flutter UI; owns `[be-without-fe]` |
| `align-meta` | Meta quality |
| `align-code` | Runtime vs requirements after code exists |
| `openspec-verify-change` | Implementation vs OpenSpec after code |
