---
name: client-align-code
description: >-
  Audits Flutter lib/ branch and working-tree changes in my_translator_app against
  the active OpenSpec change (primary), optional pasted clarifications, in-repo
  analogues, test readiness, and behavior preservation. Use when aligning Flutter
  UI code, checking FRB binding races (await gap before first UI bind, stream
  first-event loss), Material/layout overlays that hide required reader chrome,
  or when align-code runs the Flutter-layer align pass. Not for Rust-only or
  meta-doc quality audits.
---

# Client Align Code — Flutter (`lib/`)

Read-only **Flutter-layer** alignment audit for Translation Reader (`my_translator_app` → `lib/`). Report findings in **Russian** across three tiers: hard gaps, warnings, recommendations.

Stack context: Flutter/Dart UI in `lib/`; FRB 2.x glue to Rust core (`lib/src/rust/**` generated — do not treat as hand-edited source of truth); no backend. Product target UX: subtitle reader, tap/highlight, split scroll, vocab, upload, settings (app may still be scaffold — judge code vs **OpenSpec change**, not imaginary full product).

**Paths:** this skill lives in **my_translator_meta** at `.agents/skills/client/client-align-code/`. Resolve app via [`docs/projects-map.md`](../../../../docs/projects-map.md) key `app` (default `../my_translator_app`). Runtime paths below are relative to **`{app}`**. OpenSpec lives in **meta** (`.`). Before deep reads — `{app}/AGENTS.md`.

Orchestrator: [`align-code`](../../align-code/SKILL.md) may call this skill as the Flutter pass. Axes A–D here map to align-code «постановка / код / тесты / регрессии» + Align C races. Do **not** audit `rust/` here (Rust pass is separate).

## Four Required Axes

Always complete all four:

1. **Requirements fit** — code vs **активный OpenSpec change** (primary); paste/диалог только как явные уточнения.
2. **Codebase fit** — changed behavior vs strong repository analogues in `lib/`.
3. **Test readiness** — deterministic defects in reachable states before deriving tests.
4. **Behavior preservation** — unjustified regressions vs base in refactors, shared widgets, and neighboring edits.

Primary subject: working tree + branch diff in **app** (`lib/` and related Flutter tests). Active OpenSpec change in **meta** is the primary requirements source.

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

1. **Active OpenSpec change**: `proposal`, delta `specs/**/spec.md`, `design`, `tasks` — SC-*, acceptance, design decisions, task scope. Visual Spec (from mock) when present is first-class for UI chrome.
2. Optional paste / dialog clarifications that explicitly update expected behavior.
3. `{app}/AGENTS.md` / strong `lib/` analogues — conventions only, never replacements for change evidence.

Treat explicit prohibitions and exceptions as atomic: «не отображать», «не добавлять», «скрыть», «только для…», «кроме…».

Heavy compute (parse, lemma; offline translate on **native**) belongs in **Rust**; user persistence (library / dictionary) belongs in **Drift** on Flutter. Web offline translate may use Transformers.js behind `lib/data/translate/` (same UX contract — not a Dart-invented gloss). Flutter must bind FRB for native compute, not reimplement NLP/parse. Flag Flutter inventing core compute rules when AC puts them in Rust as `code-only` / `extra` as appropriate.

A tester's guess is only a lead. Dialog wording is clarification only when it explicitly updates expected behavior relative to the active change.

## Atomic Requirement Checklist

For every stated UI field / chrome element, independently verify:

- displayed vs intentionally hidden;
- data source and transformation (FRB DTO field → widget; lemma highlight → token span);
- requiredness;
- type and format;
- allowed range / length;
- default / placeholder / empty state;
- enabled / disabled / loading;
- validation and conditional visibility.

A rendered control is not covered until all explicit properties are covered.

For every user action / FRB call, independently verify:

1. triggering gesture/UI action and timing;
2. required warning or blocking confirmation when AC demands it (`showDialog` / `AlertDialog` / platform confirm — not a silent SnackBar alone);
3. transport: which generated FRB API (`package:…/src/rust/api/…`) and whether the call matches `rust/src/api` intent (names/types; do not hand-edit generated Dart);
4. every argument: requiredness, value, source (widget state / controller / session);
5. argument order / named params into the FRB function;
6. wiring (`onPressed` / `GestureDetector` / notifier method);
7. result / stream / error handling → `setState` / notifier / UI bind;
8. required refresh, dispose/cancel, and post-action notice (banner / SnackBar / inline error per AC).

Trace the concrete chain:

```text
UI event → widget / notifier
  → FRB api.* (generated) → Rust api
  → Future/Stream result | error
  → setState / ValueNotifier / Riverpod|Provider (if present)
  → rebuild chrome (reader / card / vocab / split)
```

Method presence or «looks compatible» alone is insufficient. Track each contract fact separately.

When Flutter UI and OpenSpec/Rust API disagree, report `code-only` / contradiction with both sides named (`lib/…` vs change artifacts / expected FRB surface). SnackBar / empty `catch` is not blocking confirmation.

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

Scope for this skill: `lib/**`, Flutter tests (`test/`, `integration_test/`), `pubspec.yaml` / `analysis_options.yaml` if they affect UI contract. Ignore pure `rust/` hunks (note them only if they change the FRB surface the UI must bind).

Read relevant untracked files and full current Dart files when surrounding behavior matters. Do not ignore unstaged work.

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
- `extra` — clearly forbidden or unjustified behavior/data.

Ambiguous CR wording, unresolved Open Questions, or unproven leans → **Warnings**, not hard omissions.

Wrong field source/constraint, wrong FRB API/args, absent confirm, incorrect gesture wiring, missing required chrome (reader toolbar, vocab CTA, split panes), or Flutter inventing Rust-owned rules are explicit omissions.

Score **Постановка: N/10** only from hard omissions (`missing` / `docs-only` / `code-only` / `extra`), not from Warnings or Recommendations.

## Axis B — Codebase

Find strong untouched analogues for the same domain/flow. Prefer same layer:

| Concern | Look in |
|---------|---------|
| Screens / pages | `lib/**/*_page.dart`, `lib/**/screens/**`, `lib/main.dart` |
| Widgets | `lib/**/widgets/**`, feature folders under `lib/` |
| State | `ChangeNotifier` / Riverpod / Bloc / Provider — whatever the repo uses |
| FRB bind | imports of `package:my_translator_app/src/rust/…`; call sites after `RustLib.init()` |
| Navigation | `Navigator` / `GoRouter` / routes if present |
| Errors / loading | banners, `SnackBar`, inline error text, progress indicators |
| Tests | `test/`, `integration_test/` |

Compare notices, loading/empty/error/retry, dispose of controllers/subscriptions, and empty lists only when the analogue proves the behavior.

Prefer FRB I/O behind a thin UI/notifier boundary (not duplicated parse/lemma in widgets) — flag deviations as codebase-implied when analogues keep that boundary. **Never** require hand-edits of `lib/src/rust/**`.

Report `gap` only with:

- concrete analogue path;
- changed file lacking the behavior;
- why parity is required rather than optional polish.

Label analogue-driven findings as codebase-implied. Score **Код проекта: N/10** only from hard `gap` items.

## Axis C — Test Readiness

Use evidence in this order:

1. explicit requirements — **resolved OpenSpec change** (SC-*, design, tasks); do not invent generic edge cases instead of change artifacts;
2. widget constructors / public notifier APIs / FRB Future&Stream contracts;
3. strong analogues;
4. deterministic runtime semantics (`RustLib.init()` before `runApp`, stream listen timing, dispose).

For reachable behavior, examine empty/null/zero/false states, loading/success/empty/error/retry, repeated taps, async cleanup, conditional chrome, validation-vs-handler mismatches — **and** every edge/negative path named by the change.

### Rebuild / async feedback loops (обязательно)

Когда diff трогает `setState`, `notifyListeners`, Riverpod/Bloc updates, or FRB calls from build/listen paths — проверь, нет ли шторма вызовов.

Для каждой цепочки:

1. **Триггер** — build phase vs post-frame vs explicit user event.
2. **Side effect** — FRB Future/Stream, file pick, тяжёлая sync-работа на UI isolate.
3. **Мутация watched-состояния** — пишет ли ответ в то же состояние, что снова запускает вызов.
4. **Guard** — in-flight / generation / «уже загрузили для этого id»; generation, игнорирующий только stale **ответы**, не блокирует новые вызовы → петля не разорвана.
5. **Кратность** — на один логический event ожидается конечное малое число FRB-вызовов (обычно 1).

Типичные петли:

```text
build/listen → FRB call → setState/notifier
  → rebuild/listen again → FRB call again → …
```

- FRB call directly inside `build` without cache/guard;
- `addListener` / `ref.listen` without dispose + remount → duplicate subscriptions;
- error/retry path that re-sets the same condition without backoff/stop.

Hard `[defect]` when a reachable path deterministically storms FRB/I/O or spins rebuilds. Incomplete proof → **Warning**.

### Async races — await gap before first UI bind / stream first event (обязательно)

Когда diff трогает `RustLib.init()`, FRB `Future`/`Stream`, listen/subscribe, first paint bind of reader/vocab/progress — проверь гонки: первый inbound event уходит в никуда, UI остаётся пустым/устаревшим.

Контекст FRB:

- `main()` must `await RustLib.init()` **before** `runApp` — UI calling API earlier is undefined.
- Stream/progress: listener attached **after** first event (or after an `await` gap) can miss the only emission → stuck loading / empty list.
- `await` between obtaining a stream/handle and `listen` / first synchronous snapshot read is high risk when subsequent events are sparse.

Для каждой цепочки:

1. **Точка готовности** — `RustLib.init()` done; Future resolved; Stream subscribed.
2. **Первый критичный inbound** — first stream event, first list snapshot, first parse result needed for chrome.
3. **Регистрация bind** — где `listen` / `StreamBuilder` / notifier attach; есть ли immediate snapshot / current value read.
4. **Await-gap** — есть ли `await` (navigation, dialog, лишний I/O) **между** ready и attach/bind.
5. **Идемпотентность после пропуска** — если первый event пропущен, придёт ли повторный, или UI «застынет» → hard defect.
6. **Порядок** — канон: init → create/obtain stream or start call → **сразу** subscribe/bind → потом прочий await/cleanup. Обратный порядок — hard `[defect]` при доказуемом miss.

Типичные гонки:

```text
final stream = api.watchVocab();
await Navigator.push(...);   // ← gap: first event already fired
stream.listen(updateUi);     // too late → empty vocab chrome
```

- UI binds FRB before `RustLib.init()` completes;
- `StreamBuilder` created after an await that missed the first event with no seed value;
- fire-and-forget Future without attaching error/result to UI state.

Hard `[defect]` when a reachable path can miss the first event/result and leave UI wrong with no further update. Incomplete proof → **Warning**.

### Overlay / Material layout hiding required chrome (обязательно при касании Stack / progress / barriers)

Когда diff трогает `Stack`, `Overlay`, `ModalBarrier`, full-screen `CircularProgressIndicator`, nested `AbsorbPointer`/`IgnorePointer`, split-view panes, or absolute-positioned layers over reader/vocab chrome — проверь, не прячет ли композиция обязательный видимый контент.

Для каждой композиции:

1. **Обязательный видимый якорь** — что AC/Visual Spec требует видеть (reader lines, tap target, vocab CTA, split panes, app bar actions).
2. **Hit-test / visibility** — перекрывает ли barrier/progress hit-testing или paints over chrome without a hole/`IgnorePointer` where needed.
3. **Safe patterns** — progress as sibling that does not cover mandated controls; `SafeArea`; split panes both reachable; loading inline near content rather than full-screen wipe unless AC says so.
4. **Unsafe patterns** — full-screen opaque barrier while AC requires visible toolbar/CTA; `Stack` child with expanding size covering the reader; progress parent that removes child from tree instead of overlaying safely.

Hard `[defect]` when a reachable state deterministically omits mandated chrome because of overlay/layout stacking. Incomplete proof → **Warning**.

### UI-thread / layout invariants (кратко)

- Тяжёлый sync-вызов на UI isolate (парсинг большого файла в Dart, огромный `setState` rebuild всего дерева) при AC «не блокировать UI» → hard или Warning по доказанности.
- Split sync-scroll: если AC требует построчный sync — проверь, что listeners/controllers disposed and not one-way broken after neighbor edits.
- `%` / unbounded constraints: `Expanded`/`Flexible` misuse → zero-height pane hiding content → same class as overlay hide when proven.

Report hard `defect` only when a reachable state deterministically causes wrong UI, runtime failure, invalid value, stuck state, unsafe side effect (including I/O storms, **missed first bind/event**, **overlay hide**), or contract violation.

Each hard defect includes condition, current behavior, expected invariant/evidence, file/symbol, and minimum scenario. Verdict:

- `READY` — no proven deterministic hard defect;
- `BLOCKED` — at least one hard defect must be resolved or explicitly characterized before tests.

Unresolved Open Questions that do not prove wrong UI → **Warnings**, not `BLOCKED`.

## Axis D — Behavior Preservation

Classify every changed hunk:

- `task` — directly justified by an atomic requirement;
- `incidental` — refactor/rename/cleanup/shared-helper change;
- `neighbor` — unrelated changed behavior in a task file.

Compare before vs after using diff/base, contracts, and callers. Report `regression` only for proven prior behavior changed without task justification: public widget constructor params, notifier/public state APIs, FRB call shapes the UI depends on, navigation routes, shared-helper results, `RustLib.init()` ordering.

Do not run tests. Do not double-count the same issue as both defect and regression unless evidence and impact differ.

### Changed widget / notifier public surface

When a changed file touches public constructors, named params, callbacks (`VoidCallback` / `ValueChanged`), inherited model fields, or exported helpers, audit the public contract separately from business logic.

For every removed/renamed param/callback/field:

1. **Old public surface** — which params/callbacks callers used.
2. **New resolution path** — where the value must come from now.
3. **Caller migration** — grep for old names; untouched call sites after a rename are strong regression evidence.
4. **Effective UI** — what the user still sees/can tap.

Typical regressions: required param added without updating call sites; callback renamed; key/scroll controller identity changed breaking split sync.

### FRB bind surface and app chrome wiring

Treat generated FRB usage and app chrome as public risk class:

1. **Init order** — `await RustLib.init()` still before `runApp`; no UI path calls API earlier.
2. **Generated API** — UI still imports generated API; no hand-patched `lib/src/rust/**` as «fix».
3. **Call sites** — renamed FRB wrappers / notifier methods still match screens.
4. **Streams / Futures** — listen/cancel/dispose still correct after refactor (see Axis C races).
5. **Error UX** — failures still surface per prior pattern/AC (inline / SnackBar / dialog), not silent catch.
6. **Chrome** — reader/vocab/settings affordances not removed by neighbor layout edits.

Typical regressions: listen moved after await gap; full-screen progress added that hides CTA; dispose dropped on `StreamSubscription`; demo scaffold asserts in `integration_test` left expecting old UI while production chrome changed (note as Warning unless AC requires those tests green now).

## Severity

Every finding goes into exactly one tier.

| Tier | When | Align cues |
|------|------|------------|
| **Hard gaps** | Explicit requirement / forbidden behavior / required analogue parity / deterministic wrong state / proven unjustified regression | `missing`, `docs-only`, `code-only`, `extra`, required `gap`, `defect`, `regression`; scores and `BLOCKED` use **only** this tier |
| **Warnings** | Strong lean without full proof; ambiguous AC; desirable analogue parity not mandatory; Open Question that can change behavior | `[warning]`; does **not** lower scores or force `BLOCKED` alone |
| **Recommendations** | Optional polish; docs/OpenSpec sync; unchecked tasks; soft UX parity; test ideas that are not defects | `[recommendation]`; informational only |

Examples:

- GUI requires vocab CTA and code omits it → hard `[missing]`.
- AC unclear whether empty dictionary shows placeholder; code shows blank with no wrong message proven → **Warning**.
- Analogue always shows inline error after FRB failure and CR says «как на экране загрузки» → hard `[gap]` if that screen is cited; if CR silent → **Recommendation**.
- Unchecked `tasks.md` while code exists → **Recommendation**.
- FRB call inside `build` storms parse API → hard `[defect]`.
- `await` between stream obtain and `listen` misses first event → hard `[defect]`.
- Full-screen barrier hides required reader app-bar action → hard `[defect]`.

Hard detail sections omit when empty. **Warnings** and **Recommendations** always appear; if empty, a single line `- нет`.

## Finding card (when listing items)

```markdown
### <N>. <заголовок>
- Метка: `[missing|docs-only|code-only|extra|gap|defect|regression|warning|recommendation]`
- Файл: [`lib/…`](../../../../my_translator_app/lib/…) — **L…**
- Связанные: OpenSpec `openspec/changes/<name>/…` — **L…** / аналог `lib/…`
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

## Изменившийся публичный контракт виджетов / notifier
- [regression|defect] <old param/callback → new path; call sites>

## FRB bind / chrome
- [regression|defect] <init order | api call | stream listen/dispose | error UX | chrome; before/after; consumers>

## Rebuild / async-петли
- [defect|warning] <trigger → FRB/side effect → mutated state → re-trigger; file/symbol>

## Async-гонки (await до bind / первый stream event)
- [defect|warning] <init/stream/future ready → await gap → missed first event/result; attach order; file/symbol>

## Overlay / layout (скрытие chrome)
- [defect|warning] <Stack/barrier/progress covers mandated chrome | zero-height pane; required visible anchor; file/symbol>

## Предупреждения
- [warning] <risk / ambiguity / likely gap; evidence; what would promote it to hard>
- нет

## Рекомендации
- [recommendation] <optional polish / docs sync / soft parity>
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
