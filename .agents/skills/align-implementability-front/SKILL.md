---
name: align-implementability-front
description: >-
  Use when judging whether my_translator_meta Flutter/UI claims are implementable
  (possibility to build only — not doc quality, not that Rust already shipped).
  Against current my_translator_app/lib (+ FRB-facing API shapes via projects-map).
  For mock-driven UI also audits Visual Spec completeness (sizes, padding, colors
  hex/rgba, typography, alignment, dividers, icons, hover) so FE can implement
  without inventing tokens (`[visual-spec-incomplete]`). Treat missing Rust sibling
  changes as expected lag if meta may add them; warn when FE-consumable Rust/API
  claims lack UI (`[be-without-fe]`). UI with no Rust/API in meta
  (`[fe-only-without-be]`) → align-implementability-back. Verdict
  FEASIBLE|PARTIAL|INFEASIBLE|UNKNOWN. Finding card: file + lines + issue + fix.
  Not Rust audit, meta quality, or pixel verify (verify-mock). No file edits.
---

# Align Implementability (Frontend / Flutter)

Read-only **Flutter implementability** audit vs **my_translator_app/lib**. Report in Russian.

**Scope — only possibility to implement.** Meta prose quality → **`align-meta`**. Deep Rust realizability / `[fe-only-without-be]` → **`align-implementability-back`**. Extract tokens from mock → **`prepare-mock`**. Pixel verify shipped UI → **`verify-mock`**.

Tiers: (1) Flutter implementability, (2) Rust→Flutter coverage `[be-without-fe]`, (3) Visual Spec completeness when mock-driven.

## Rust lag and BE-without-FE

| Ситуация | Суждение |
|----------|----------|
| UI field/action отсутствует в rust HEAD, но meta планирует Rust claim | Не hard; **extends** / Warning `[be-not-yet-in-siblings]` |
| UI без Rust/API claim в meta | Pointer → `align-implementability-back` `[fe-only-without-be]` |
| FE-потребляемый Rust/API claim без UI surface | Warning `[be-without-fe]` — не hard |
| UI path/API contradicts proven binding и meta не планирует сдвиг | Hard `conflicts` |

**FE-потребляемое:** FRB methods Flutter calls, result/error types for UI, streams for progress, file pick outcomes, vocab list DTOs. **Не** internals crates без UI surface.

## Runtime roots

- **Claims:** meta asserting Flutter: screens/widgets, navigation, state, Material/Cupertino UX, i18n, permissions UX, FRB calls as UI must bind them, Visual Spec when mock-driven.
- **Evidence:** `app` → `../my_translator_app/`, primary `lib/`. Visual Spec: `design.md` → `## Visual Spec (from mock)`.
- Skills: `.agents/skills/` (flutter/app topics when present).

## Scope filter

**Include:** screen/widget/modal placement; UI flows empty/error/loading; FE validation; auth/gates as UI; FRB path/DTO as Flutter binds; mock-driven visual tokens.

**Exclude:** Rust-only algorithms without FE consumption; `[fe-only-without-be]` owned by back.

If rust-only branch with no Flutter claims → **`UNKNOWN`**; skip Visual Spec mass checklist.

## Hard Boundary

No file edits. Do not invent Visual Spec numbers.

## Load Branch Work / Evidence

Same git collect in meta as back skill; filter Flutter claims; then Rust→UI scan; mock-driven Visual Spec axis if triggered.

Mock-driven if: mock asset referenced; Visual Spec section; pixel-faithful claim; user attached mock.

Evidence in `lib/`: widgets, routes, state, tests. Before deep reads — `{app}/AGENTS.md`.

## Axis — Flutter implementability

Surfaces, placement, state patterns, FRB binding, forms, edge UX → **aligns|extends|conflicts|unproven**. Server/rust lag ≠ hard INFEASIBLE.

## Axis — Visual Spec completeness (if mock-driven)

Canon: `design.md` Visual Spec. Checklist (sizes, pads, colors L+D, typography, alignment, icons, hover) — see [`prepare-mock`](../prepare-mock/SKILL.md). Missing critical tokens → Hard `[visual-spec-incomplete]`. Soft (hover absent) → Warning.

| Вердикт | Когда |
|---------|--------|
| `FEASIBLE` | No hard conflicts; Visual Spec OK or n/a |
| `PARTIAL` | Mixed / incomplete Visual Spec |
| `INFEASIBLE` | Hard conflicts or empty Visual Spec with pixel-faithful claim |
| `UNKNOWN` | Not enough Flutter evidence |

## Finding card

```markdown
### <N>. <заголовок>
- Метка: `[gap|code-only|warning|recommendation|be-without-fe|be-not-yet-in-siblings|visual-spec-incomplete]` (+ aligns|…)
- Файл: [`path`](path) — **L…**
- Связанные: [`../my_translator_app/lib/...`](../my_translator_app/lib/...) — **L…**
- Что не так: …
- Предложение: … / `prepare-mock` / Неясно: A/B
```

## Report (Russian)

```markdown
## Вердикт
- Код проекта (flutter): N/10 — …
- Реализуемость flutter: FEASIBLE | PARTIAL | INFEASIBLE | UNKNOWN
- Visual Spec (mock): n/a | complete | incomplete
## Claims / Evidence / Visual Spec checklist
## Замечания (hard / warnings / recommendations)
## Что делать дальше
```

Never apply fixes.

## Not This Skill

| Skill | Difference |
|-------|------------|
| `align-implementability-back` | Rust; owns `[fe-only-without-be]` |
| `prepare-mock` | Extracts tokens |
| `verify-mock` | Shipped UI vs mock pixels |
| `align-meta` | Meta quality |
