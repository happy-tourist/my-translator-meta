---
name: server-align-code
description: >-
  Audits Rust rust/ branch and working-tree changes in my_translator_app against
  the active OpenSpec change (primary), optional pasted clarifications, in-repo
  analogues, test readiness, and behavior preservation. Use when aligning Rust
  core code, checking FRB surface races (init_app / #[frb(init)], stream
  first-event loss, blocking sync work on the FFI thread), parse / tokenize /
  lemmatize / dictionary / translate / db contracts, or when align-code runs the
  Rust-layer align pass. Not for Flutter UI or meta-doc quality audits.
---

# Server Align Code — Rust (`rust/`)

Read-only **Rust-core** alignment audit for Translation Reader (`my_translator_app` → `rust/`). Report findings in **Russian** across three tiers: hard gaps, warnings, recommendations.

Stack context: Rust edition 2021 crate `rust_lib_my_translator_app` in `rust/`; public FFI in `rust/src/api/**`; FRB 2.13 glue (native FFI / WASM). Generated never hand-edit: `rust/src/frb_generated.rs`, `lib/src/rust/**`. Target core modules: parse / tokenize / lemmatize / dictionary / translate / db — scaffold today mostly `api/simple.rs` (`process_text`, `add_numbers`, `init_app`). Judge code vs **OpenSpec change**, not imaginary full product.

**Paths:** this skill lives in **my_translator_meta** at `.agents/skills/server/server-align-code/`. Resolve app via [`docs/projects-map.md`](../../../../docs/projects-map.md) key `app` (default `../my_translator_app`). Runtime paths below are relative to **`{app}`**. OpenSpec lives in **meta** (`.`). Before deep reads — `{app}/AGENTS.md`.

Orchestrator: [`align-code`](../../align-code/SKILL.md) may call this skill as the Rust pass. Axes A–D here map to align-code «постановка / код / тесты / регрессии» + Align C races. Do **not** audit Flutter `lib/` UI here (Flutter pass: [`client-align-code`](../../client/client-align-code/SKILL.md)). Sibling server skills: [`.agents/skills/server/`](../) (e.g. verify / locate / structure / test when present).

## Four Required Axes

Always complete all four:

1. **Requirements fit** — code vs **активный OpenSpec change** (primary); paste/диалог только как явные уточнения.
2. **Codebase fit** — changed behavior vs strong repository analogues in `rust/`.
3. **Test readiness** — deterministic defects in reachable states before deriving tests.
4. **Behavior preservation** — unjustified regressions vs base in refactors, shared modules, and neighboring edits.

Primary subject: working tree + branch diff in **app** (`rust/` and related Rust tests). Active OpenSpec change in **meta** is the primary requirements source.

## Hard Boundary

Do not edit, create, delete, format, or commit files. Do not run tests for preservation analysis. Do not run FRB codegen. Deliver the complete result in chat.

(When invoked under `align-code` / `implement-change`, follow the orchestrator’s verify/fix overrides; this skill itself stays read-only for align.)

## Input And OpenSpec

**Primary:** resolve and load one **active OpenSpec change**. Accept an optional change name; otherwise auto-select the single active change. Paste/dialog may refine AC only when it does not contradict the change (or the user explicitly overrides).

Ask when the change cannot be resolved (0 or >1 active without a clear pick, and no name given). Do not treat paste alone as a substitute unless the user confirms `OpenSpec: none`.

### Resolve OpenSpec change

Resolve **one** change name before Axes A–D:

1. Use the name the user passed.
2. Else infer from conversation if unambiguous.
3. Else from meta root: `openspec list --json` — auto-select if exactly **one** active change.
4. Else if several — ask the user to pick one.
5. Else if none — ask for a change name or explicit `OpenSpec: none`.

Announce: `Using OpenSpec change: <name>` (or `OpenSpec: none`).

That change is **primary requirement evidence** for atomic checklists, Axis A, Axis C edge cases / scenarios, and docs↔code contradictions.

## Requirements Sources

Load **in this order**:

1. **Active OpenSpec change**: `proposal`, delta `specs/**/spec.md`, `design`, `tasks` — SC-*, acceptance, design decisions, task scope.
2. Optional paste / dialog clarifications that explicitly update expected behavior.
3. `{app}/AGENTS.md` / strong `rust/` analogues — conventions only, never replacements for change evidence.

Treat explicit prohibitions and exceptions as atomic: «не возвращать», «не добавлять», «только для…», «кроме…», «без сети», «stub».

Heavy **compute** (parse, lemma, offline translate) belongs in **Rust**; Flutter must bind FRB, not reimplement. User persistence (library / dictionary) is **Drift in Flutter** — not `rusqlite` / `rust/src/db`. Flag Rust leaking UI concerns or missing FFI exports when AC puts rules in core as `code-only` / `extra` as appropriate. Authoritative subtitle/NLP truth lives in Rust — do not treat Flutter local string munging as fulfilment of compute AC.

A tester's guess is only a lead. Dialog wording is clarification only when it explicitly updates expected behavior relative to the active change.

## Atomic Requirement Checklist

### FFI types / DTOs

For every stated public type / struct / enum exported via FRB (`rust/src/api/**` or re-exported for Dart), independently verify:

- present vs intentionally omitted on the FFI surface;
- field source/path and transformation (internal module → api DTO → generated Dart);
- requiredness for Flutter contract;
- type and format (String, i32/u32, bool, Vec, Option, nested structs, enums);
- allowed range, length, empty vs null (`Option` vs empty `Vec`/`String`);
- default / empty / initial values after parse or DB load;
- who owns mutation (Rust core — Flutter must not invent lemma/dict rules);
- naming/shape match expected generated Dart (`lib/src/rust/api/…` — do not hand-edit).

A synced/exported field is not covered until all explicit properties are covered. Scaffold today: `process_text` / `add_numbers` in `api/simple.rs` — do not treat scaffold alone as fulfilment of AC that demands parse/lemma/dict/translate.

### FFI functions / streams

For every public FRB function / stream / sink, independently verify:

1. triggering Flutter call site intent and when core is ready (`RustLib.init` / `#[frb(init)]` done);
2. exact exported name and module path under `rust/src/api/`;
3. every argument: requiredness, type, value domain, source (path bytes, lang codes, lemma id, cue index);
4. argument order / named params as FRB will expose;
5. sync vs async (`async fn` / `Stream`) and whether work is offloaded off the calling FFI thread when AC says «не блокировать UI»;
6. Result / error contract (`Result`, panic boundaries, typed errors vs opaque strings);
7. side effects: DB writes, file reads, in-memory caches;
8. stream lifecycle: create → first event → cancel/drop; no silent close without terminal event when AC requires progress;
9. idempotency / re-entrancy for repeated calls with the same inputs when AC demands it;
10. layering: thin `api/` export over parse / tokenize / lemmatize / dictionary / translate / db modules (logic not dumped into Flutter).

Trace the concrete chain:

```text
Flutter (generated api.*) → rust/src/api/*
  → core modules (parse | tokenize | lemmatize | dictionary | translate | db)
  → Result / Stream event | error
  → (Dart bind — out of scope here; note surface only)
```

Method presence or «looks compatible» alone is insufficient. Track each contract fact separately.

When Rust API and OpenSpec/Flutter bind disagree, report `code-only` / contradiction with both sides named (`rust/src/api/…` vs change artifacts / expected Dart usage). Silent `unwrap` / panic on user input is not a typed error contract.

Prefer aligning FFI names/types with Flutter call sites rather than forcing unilateral Dart patches — unless AC explicitly says otherwise. **Never** require hand-edits of `rust/src/frb_generated.rs` or `lib/src/rust/**`.

## Load Branch Work

Inspect read-only from **`{app}`**:

```bash
git status --short
git diff
git diff --cached
git branch -vv
```

When the branch has commits beyond base, determine merge-base and inspect:

```bash
git log --oneline <base>..HEAD
git diff --stat <base>...HEAD
git diff <base>...HEAD
```

Scope for this skill: `rust/**`, Rust tests (`rust/tests/`, `#[cfg(test)]` modules), `rust/Cargo.toml` / FRB-related config if they affect the FFI contract (`flutter_rust_bridge.yaml` when API input roots change). Ignore pure Flutter UI hunks in `lib/**` except generated bind notes when the FFI surface changed. Do not ignore unstaged work.

Read relevant untracked files and full current Rust modules when surrounding behavior matters. Prefer `rust/src/api/`, `rust/src/lib.rs`, core modules as they appear, and `cargo test` targets under `rust/`.

## Load Planning Scope

After resolving the change name, from **meta**:

```bash
openspec status --change "<name>" --json
```

Read concrete artifact paths (`proposal`, `specs`, `design`, `tasks`). Use as **primary** requirements for **all four axes**, including Axis C edge cases and SC-* from delta specs. Report clear docs↔code contradictions. If no change resolved — only after `OpenSpec: none`, audit from paste/dialog and repository evidence; otherwise stop and ask.

## Axis A — Requirements

Judge implementation first, then planning artifacts.

Report hard gaps only as:

- `missing` — explicitly required and absent;
- `docs-only` — promised by complete artifacts but absent from implemented feature;
- `code-only` — clear docs/code contradiction;
- `extra` — clearly forbidden or unjustified behavior/data/side effect.

Ambiguous CR wording, unresolved Open Questions, or unproven leans → **Warnings**, not hard omissions.

Wrong DTO field/constraint, missing FFI export, wrong Result/error shape, parse/lemma/dict rules absent while AC demands them, or blocking sync work on the FFI thread when AC forbids UI jank are explicit omissions.

Score **Постановка: N/10** only from hard omissions (`missing` / `docs-only` / `code-only` / `extra`), not from Warnings or Recommendations.

Known scaffold fact: API is mostly `api/simple.rs`. Do not treat `to_uppercase` / `add_numbers` as fulfilment of AC that demands subtitle parse, lemmatize, dictionary, or offline translate.

## Axis B — Codebase

Find strong untouched analogues for the same domain/flow. Prefer same layer:

| Concern | Look in |
|---------|---------|
| Public FFI | `rust/src/api/**/*.rs`, `rust/src/api/mod.rs` |
| Crate root | `rust/src/lib.rs` |
| Core modules (target) | parse / tokenize / lemmatize / dictionary / translate / db under `rust/src/` as they appear |
| FRB init | `#[frb(init)]` / `init_app` in `api/` |
| Errors | `Result` types, `thiserror`/`anyhow` patterns if present |
| Persistence (user data) | **Drift in Flutter `lib/`** — not `rusqlite` / `rust/src/db` |
| Tests | `rust/**/tests`, `#[cfg(test)]`, `cargo test` targets |
| FRB config | `{app}/flutter_rust_bridge.yaml` (`rust_input: crate::api`) |
| Flutter bind (contrast only) | generated `lib/src/rust/api/*` — read-only evidence of surface, not edit target |

Compare error shapes, empty-input handling, module layering (thin `api/` over core), async vs blocking, and test coverage of SC-* only when the analogue proves the behavior.

Prefer authoritative rules inside Rust core modules with thin FFI — flag duplicated lemma/dict logic or business rules pushed to Dart as codebase-implied when analogues keep that boundary. **Never** require hand-edits of generated files.

Report `gap` only with:

- concrete analogue path;
- changed file lacking the behavior;
- why parity is required rather than optional polish.

Label analogue-driven findings as codebase-implied. Score **Код проекта: N/10** only from hard `gap` items.

## Axis C — Test Readiness

Use evidence in this order:

1. explicit requirements — **resolved OpenSpec change** (SC-*, design, tasks); do not invent generic edge cases instead of change artifacts;
2. public FFI signatures / DTO contracts / Result types;
3. strong analogues;
4. deterministic runtime semantics (`init_app` before any API use, stream first event, no blocking storm on FFI thread).

For reachable behavior, examine empty/null/zero/false inputs, invalid subtitle bytes, unknown language codes, missing lemma, empty dictionary, DB open failure, repeated calls, cancel mid-stream, Option vs empty collections — **and** every edge/negative path named by the change.

### Call / feedback loops (обязательно)

Когда diff трогает FFI entrypoints, streams/sinks, timers, or internal callbacks that re-enter the same API — проверь, нет ли шторма вызовов/событий.

Для каждой цепочки:

1. **Триггер** — inbound FRB call, stream poll, DB watcher, internal task.
2. **Side effect** — emit stream event, spawn task, write DB that re-triggers the same path, sync re-entry into the same `api` fn.
3. **Повторный вход** — может ли side effect снова попасть в тот же handler без нового внешнего вызова.
4. **Guard** — in-flight / generation / «уже посчитали для этого id»; generation, игнорирующий только stale **ответы**, не блокирует новые вызовы → петля не разорвана.
5. **Кратность** — на один логический Flutter call ожидается конечное малое число тяжёлых работ / stream events (обычно 1 результат или явный прогресс по AC).

Типичные петли:

```text
api call → mutate cache/DB → watcher/callback
  → same api path again → …
```

- stream handler emits an event that immediately re-invokes the producing function without external demand;
- retry without backoff/max on parse/DB failure inside the same call stack;
- progressive emit that never terminates when AC expects a finite sequence.

Hard `[defect]` when a reachable path deterministically storms work/events. Incomplete proof → **Warning**.

### Async races — FRB init / stream first event (обязательно)

Когда diff трогает `#[frb(init)]` / `init_app`, stream/sink APIs, or first emission of parse/progress/vocab updates — проверь гонки: первый inbound event уходит до подписки Dart, или API вызывается до init.

Контекст FRB (Rust side):

- `init_app` / `#[frb(init)]` must establish runtime utils **before** other API use; Flutter must `await RustLib.init()` before calls — flag Rust that assumes init state without documenting/enforcing it when AC requires ordered boot.
- Stream/progress: first event emitted **before** Dart `listen` (or after an await gap on the UI side) can miss the only emission → stuck loading. On Rust: prefer hold/buffer/replay or emit-after-subscribe semantics when AC needs a guaranteed first snapshot.
- `await` / yield between creating a stream handle and attaching the first producer that can fire is high risk when subsequent events are sparse.

Для каждой цепочки:

1. **Точка готовности** — `init_app` done; stream subscribed / handle live.
2. **Первый критичный outbound** — first stream event, first parse chunk, first list snapshot needed by UI.
3. **Регистрация consumer** — где Dart listen произойдёт (note only); есть ли immediate snapshot / current value on subscribe.
4. **Gap** — есть ли окно, где событие может уйти до подписки или API зовётся до init.
5. **Идемпотентность после пропуска** — если первый event пропущен, придёт ли повторный, или UI «застынет» → hard defect when Rust cannot recover.
6. **Порядок** — канон: init → create stream / start work → **consumer can attach before first mandatory emit** (or snapshot on subscribe) → потом прочий await. Обратный порядок — hard `[defect]` при доказуемом miss.

Типичные гонки:

```text
// Rust starts emitting before Dart listen
let stream = watch_vocab(); // first item pushed immediately
// Dart: await something(); stream.listen(...)  → miss
```

- API usable paths that panic if called before `init_app`;
- one-shot stream with no replay when AC requires initial snapshot;
- fire-and-forget internal task that drops errors the UI must see.

Hard `[defect]` when a reachable path can miss the first event/result and leave the contract wrong with no further update. Incomplete proof → **Warning**.

### Blocking sync work on FFI thread (обязательно)

Когда diff трогает sync `fn` на горячем пути (parse большого SRT, batch lemmatize, full-file translate) — проверь, не блокирует ли вызов UI/FFI thread вопреки AC.

Для каждого тяжёлого entrypoint:

1. **Объём работы** — O(file) / O(cues) sync на вызывающем потоке.
2. **Контракт AC** — «не блокировать UI», async FRB, chunked progress, isolate/worker expectations.
3. **Safe patterns** — `async fn` with real yield points; chunked processing; offload per project conventions when present.
4. **Unsafe patterns** — sync parse of multi‑MB input entirely inside a sync FFI fn while AC forbids jank; nested sync re-entry that holds the thread across DB + NLP.

Hard `[defect]` when a reachable call deterministically performs heavy sync work on the FFI/UI-facing thread against an explicit non-blocking requirement. Incomplete proof → **Warning**.

Report hard `defect` only when a reachable state deterministically causes wrong Result/stream payload, panic/runtime failure, invalid DTO, stuck stream, unsafe side effect (including call/event storms, **missed first event**, **FFI-thread block**), or contract violation.

Each hard defect includes condition, current behavior, expected invariant/evidence, file/symbol, and minimum scenario. Verdict:

- `READY` — no proven deterministic hard defect;
- `BLOCKED` — at least one hard defect must be resolved or explicitly characterized before tests.

Unresolved Open Questions that do not prove wrong API → **Warnings**, not `BLOCKED`.

Cite missing `cargo test` coverage as **Recommendations** unless the missing test is the only way to prove a hard defect already evidenced in code. Do not run `cargo test` here for preservation analysis.

## Axis D — Behavior Preservation

Classify every changed hunk:

- `task` — directly justified by an atomic requirement;
- `incidental` — refactor/rename/cleanup/shared-helper change;
- `neighbor` — unrelated changed behavior in a task file.

Compare before vs after using diff/base, contracts, and callers. Report `regression` only for proven prior behavior changed without task justification: public FFI names/signatures, DTO fields, Result/error shapes, stream semantics, `init_app` ordering, shared helper results used by multiple API paths, or Rust tests that encode the contract.

Do not run tests. Do not double-count the same issue as both defect and regression unless evidence and impact differ.

### Changed FFI public surface

When a changed file touches `rust/src/api/**`, public structs/enums, or exported fn signatures, audit the contract separately from internal helpers.

For every removed/renamed/reshaped export:

1. **Old public surface** — fn/type/field names and Result shapes Flutter/tests used.
2. **New resolution path** — which module/fn now owns the value.
3. **Caller migration** — grep generated Dart imports and `rust/` tests for old names; untouched call sites after a rename are strong regression evidence (note Flutter regen need — do not hand-edit generated Dart).
4. **Effective behavior** — same inputs still yield the AC-required outputs.

Typical regressions: required arg added without updating call sites; DTO field renamed; stream no longer emits initial snapshot; `init_app` side effects removed.

### Core modules, persistence, and FRB wiring

Treat core pipelines and init as public risk class:

1. **Init order** — `#[frb(init)]` / `init_app` still establishes required utils; no API path assumes uninitialized global state without AC.
2. **Generated artifacts** — no «fix» by hand-editing `frb_generated.rs` / `lib/src/rust/**`.
3. **Layering** — parse/lemma/dict/translate/db still authoritative in Rust; not reintroduced as string hacks in `api/simple` only when AC demands real modules.
4. **Streams / async** — first-event and cancel/drop still correct after refactor (see Axis C races).
5. **Errors** — failures still surface as Result/typed errors per prior pattern/AC, not silent `Ok(Default)` / panic on user data.
6. **Blocking** — hot-path sync work not newly introduced against non-blocking AC (see Axis C).

Typical regressions: listen/emit order flipped; sync parse moved onto FFI thread; dictionary lookup returns different empty semantics; scaffold tests left expecting old uppercase demo while API changed (Warning unless AC requires those tests green now).

## Severity

Every finding goes into exactly one tier.

| Tier | When | Align cues |
|------|------|------------|
| **Hard gaps** | Explicit requirement / forbidden behavior / required analogue parity / deterministic wrong state / proven unjustified regression | `missing`, `docs-only`, `code-only`, `extra`, required `gap`, `defect`, `regression`; scores and `BLOCKED` use **only** this tier |
| **Warnings** | Strong lean without full proof; ambiguous AC; desirable analogue parity not mandatory; Open Question that can change behavior | `[warning]`; does **not** lower scores or force `BLOCKED` alone |
| **Recommendations** | Optional polish; docs/OpenSpec sync; unchecked tasks; soft parity; test ideas that are not defects | `[recommendation]`; informational only |

Examples:

- AC requires `parse_subtitles` FFI and code omits it → hard `[missing]`.
- AC unclear whether empty dict returns `[]` or error; code picks one with no wrong consumer proven → **Warning**.
- Analogue always maps invalid SRT to `Err` and CR says «как parse_x» → hard `[gap]` if that file is cited; if CR silent → **Recommendation**.
- Unchecked `tasks.md` while code exists → **Recommendation**.
- Sync multi‑MB parse on FFI thread against «не блокировать UI» → hard `[defect]`.
- One-shot stream emits before subscribe with no replay → hard `[defect]`.
- Hand-edited `frb_generated.rs` as «fix» → hard `[extra]` / `[code-only]` vs AGENTS rules.

Hard detail sections omit when empty. **Warnings** and **Recommendations** always appear; if empty, a single line `- нет`.

## Finding card (when listing items)

```markdown
### <N>. <заголовок>
- Метка: `[missing|docs-only|code-only|extra|gap|defect|regression|warning|recommendation]`
- Файл: [`rust/…`](../../../../my_translator_app/rust/…) — **L…**
- Связанные: OpenSpec `openspec/changes/<name>/…` — **L…** / аналог `rust/…`
- Что не так: …
- Предложение: … / Неясно: A/B
```

(Paths in the report may be repo-relative from meta or app; keep them clickable when possible.)

## Report

Write in Russian.

```markdown
## Вердикт
- Постановка: N/10 — <основание; только hard>
- Код проекта: N/10 — <основание; только hard>
- Готовность к тестам: READY | BLOCKED — <основание; только hard defect>
- Регрессии: нет | N — <основание; только hard>
- Режим: requirements-only | post-propose | in-progress
- Источник: активный OpenSpec change <name> (+ уточнения | none + fallback)
- OpenSpec change: <name | none; passed | active | inferred>
- Работа в ветке: <staged / unstaged / untracked / commits в app>

## Просмотренные файлы
- `path` — <state>

## Пробелы относительно постановки
- [missing|docs-only|code-only|extra] <gap and evidence>

## Пробелы относительно кода проекта
- [gap] <gap> — аналог: `path` — нет в: `changed-file`

## Расхождения docs и кода
- <proven contradiction>

## Явные дефекты перед написанием тестов
- [defect] <condition/current/expected/evidence/file/scenario>

## Регрессии прежнего поведения (без прогона тестов)
- [regression] <hunk class/before/after/justification/file/caller>

## Изменившийся публичный FFI / DTO
- [regression|defect] <old fn/type/field → new path; Dart/test call sites>

## FRB init / core wiring
- [regression|defect] <init_app | layering | Result/error | stream cancel; before/after; consumers>

## Call / feedback-петли
- [defect|warning] <trigger → side effect → re-entry; expected 1 effect vs storm; file/symbol>

## Async-гонки (init / первый stream event)
- [defect|warning] <init/stream ready → gap → missed first event/result; attach/emit order; file/symbol>

## Blocking на FFI thread
- [defect|warning] <sync heavy work on FFI/UI-facing thread vs non-blocking AC; file/symbol>

## Предупреждения
- [warning] <risk / ambiguity / likely gap; evidence; what would promote it to hard>
- нет

## Рекомендации
- [recommendation] <optional polish / docs sync / soft parity / cargo test coverage>
- нет

## Карта аналогов
- `path` — <finding supported>

## Требуется решение перед тестами
- <only for BLOCKED from hard defects>

## Что делать дальше
- <hard first; then warnings; recommendations last>
```

Omit empty hard detail sections. Always keep **Предупреждения** and **Рекомендации** (use `- нет` when empty; do not mix items and `нет`).

Never apply fixes from align mode.
