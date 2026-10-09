---
name: client-verify-code
description: >-
  Use when the user asks to verify code, check skill compliance, audit a branch
  diff vs master/main/merge-base, audit local diffs, or after implementing a
  change in my_translator_app Flutter lib/. Also when checking Effective Dart /
  Flutter soft style, Material/FRB conventions, or DRY / KISS / YAGNI balance
  on changed files. Reports three tiers: Violations, Warnings, Recommendations.
  Feeds the Flutter Verify axis of align-code.
---

# Verify Code — Flutter (`lib/`)

Use this skill to check **all production Flutter/Dart code changed on the current
branch** in **`my_translator_app`** (`lib/`) against **all code-related** project
skills under `.agents/skills/client/`, plus the **built-in** checks in this file
(Effective Dart / Flutter soft style, Flutter+FRB conventions, and DRY / KISS /
YAGNI with conflict-aware judgment).

Default scope is the **full branch diff**: commits vs merge-base **and**
uncommitted working-tree changes. Do not stop at staged/unstaged/untracked.

This skill is primarily an **orchestrator**: domain rules live in the code
skills listed below — **read those skill files** when present and apply them; do
not restate or invent parallel domain rules here. Built-in sections below fill
gaps when a listed skill file is missing, and own Flutter soft style guidance
plus DRY / KISS / YAGNI balance.

**Paths:** skills live in **meta** at `.agents/skills/client/` (this repo =
`my_translator_meta`). Runtime code and `lib/…` paths are relative to the
**app** package root (`my_translator_app`). Resolve app via
[`docs/projects-map.md`](../../../../docs/projects-map.md) (default
`../my_translator_app`). Before work, read `{app}/AGENTS.md` when present.

**Stack assumptions:** Flutter / Dart (`sdk` per `pubspec.yaml`), Material,
`flutter_lints` + `analysis_options.yaml`, **flutter_rust_bridge** (FRB) —
generated Dart under `lib/src/rust/**` (never hand-edit). Heavy logic lives in
Rust (`rust/`); this skill verifies the **Flutter UI layer only**.

**Out of scope for this skill:** Rust crate compliance (`rust/`), `cargo`
tooling, and `rust_builder/` glue — those belong to the Rust Verify axis of
[`align-code`](../../align-code/SKILL.md). Flag only Flutter-side contract
misuse (e.g. hand-edited generated bindings, duplicated NLP/parse/dict rules in
Dart).

## When To Use

- User asks to verify code / skill compliance / convention check for Flutter
- After implementing a feature or refactor in `lib/`, before commit
- When reviewing the current branch (or a named path) for Flutter convention drift
- As the **Flutter Verify** pass inside [`align-code`](../../align-code/SKILL.md)
  (report-only unless orchestrator override asks to fix)

## Scope: code skills only

Skills live under **meta** `.agents/skills/client/<name>/SKILL.md`.

### Always include (read and apply each when the file exists)

Verify against **every** skill in this set for the checked code — not a subset
guessed from the path. Path routing below only helps prioritize deeper reading;
it does **not** allow skipping skills from this list.

| Skill | Concern (when present) |
|-------|------------------------|
| `client-work-with-structure` | `lib/` layout, screens / widgets / theme split |
| `client-work-with-errors` | async/FRB error surfacing in UI |
| `work-with-forms` | form / input validation patterns |
| `work-with-pages` | navigation / screen shell |
| `work-with-stores` | app state (Provider / Riverpod / Bloc / … if adopted) |
| `work-with-styles` | ThemeData / Material tokens / light+dark |
| `work-with-localization` | Flutter l10n / ARB |
| `client-work-with-frb` / bridge skills | FRB init, API call patterns, streams |

If a new code skill appears under `.agents/skills/client/` (same kind: how to
write app code), include it too. Prefer reading one extra skill over missing a
rule. Do **not** auto-include `client-work-with-test` here — it is test-writing
only (see Always exclude).

If a listed skill file is **missing**, do not invent a parallel rulebook — apply
**Built-in: Flutter conventions** for that concern and continue.

### Always exclude

Do **not** use these for client-verify-code (unless the user explicitly asks):

| Skill | Why excluded |
|-------|----------------|
| `client-locate-change-points` | planning where to edit |
| `client-verify-code` | this orchestrator |
| `client-align-code` / `align-code` | alignment / постановка (separate axis) |
| `client-work-with-test` | test-writing conventions (not used to audit prod code) |
| `commit` | commit messages |
| `openspec` / opsx skills | change workflow / specs |
| `verify-mock` / `prepare-mock` | pixel/mock audit (separate skill) |
| `align-implementability-*` / `align-meta` | meta / implementability, not prod Flutter audit |

Skip verifying files that are **only** tests (`test/**`, `integration_test/**`,
`**/*_test.dart`) unless the user explicitly asks to include them. Focus on
production/source code under `lib/` (excluding generated FRB output — see
below).

**Never treat as editable production sources:**

- `lib/src/rust/**` (FRB-generated Dart)
- `rust/src/frb_generated.rs` (out of Flutter scope; do not “fix” from here)

If the change set **touches** generated bindings, report a **Violation** when
they were hand-edited or when Dart call sites were updated without going through
Rust `api/` + codegen. Do not invent lint findings inside generated files.

## What To Check

**Always** inspect the full branch change set in the **app** repo. Working tree
alone is not enough.

Run git from the **app repository root** (`my_translator_app`).

1. Resolve merge-base: `git merge-base HEAD <base-ref>`. Prefer `origin/main`
   (this repo), else `origin/master`, `main`, `master`, `develop` — first that
   exists.
2. Committed on the branch: `git diff --name-only <merge-base>...HEAD` and
   `git diff <merge-base>...HEAD`
3. Staged: `git diff --cached --name-only` and `git diff --cached`
4. Unstaged: `git diff --name-only` and `git diff`
5. Untracked: `git ls-files --others --exclude-standard` (read those files fully)

Union of 2–5 is the file set (filter to Flutter production sources under `lib/`
unless the user widened scope). Never skip step 2 because the working tree looks
small.

Narrow only when the user **explicitly** asks for working-tree-only / only staged,
or names a **folder or path**.

Skip unrelated noise (lockfiles, `build/`, `.dart_tool/`, platform runner
boilerplate with no product change, binary assets) unless the user asked to
include them. Do not expand into `rust/` for this skill.

## Commands (before done)

Run tooling from the **app package root** (`my_translator_app`). Fix failures
before claiming done when verify-and-fix / implementation owns the change. Do
not skip analyze / test after changes unless the user explicitly asked for a
**report-only** pass (as in standalone `align-code` Verify — include command
results in the report, do not auto-fix unless override).

| Command | Purpose |
|---------|---------|
| `flutter pub get` | deps if needed before analyze/test |
| `flutter analyze` | analyzer + `flutter_lints` / `analysis_options.yaml` |
| `flutter test` | unit/widget tests |
| `flutter test integration_test` | integration tests when UI/FFI smoke is in scope |

Prefer `flutter analyze` and `flutter test` for touched areas. Run integration
tests when the user asks or when OpenSpec tasks / change scope require them. Do
**not** run `cargo` / Rust checks from this skill. Do not treat passing analyze
as a substitute for skill checks or missing tests from OpenSpec tasks.

Report failed analyze/test output under **Violations** (tooling) with the
command and a short failure summary. Do not invent lint rules beyond
`analysis_options.yaml` / `flutter_lints` and skill text.

### align-code Verify expectations (Flutter)

When invoked as the Flutter Verify axis of `align-code`:

- Include tooling results (`flutter analyze`, `flutter test`) in the Verify block
- Classify skill/convention findings as Violations / Warnings / Recommendations
- Stay report-only unless the orchestrator override asks to fix
- Do not mix Rust findings into this skill’s output; label paths under `lib/`

## Workflow

1. Resolve **app** path via projects-map; collect the file set: merge-base…HEAD
   **plus** staged, unstaged, untracked (or the path the user named). Exclude
   test-only and generated FRB files per Scope.
2. **Read all skills from the Always include table** that exist on disk (do not
   rely on memory; do not skip because the path “looks unrelated”).
3. For each source file, apply every code skill whose rules can touch that file.
   When unsure, apply the skill. If missing, use Built-in Flutter conventions.
4. For every checked `.dart` UI file under `lib/` (non-generated), apply
   **Built-in: Flutter / Effective Dart style**.
5. Apply **Built-in: Flutter conventions** to all checked production files.
6. Apply **Built-in: DRY / KISS / YAGNI** to all checked production files.
   Resolve principle conflicts with the order in that section **before**
   classifying or fixing — never emit opposing principle fixes for the same hunk.
7. Classify every finding into Violations / Warnings / Recommendations
   (see Severity). Each item: file path, what is wrong, which skill/rule, and
   the expected pattern. Soft Prefer guidance → Warnings or Recommendations,
   not silence.
8. Fix **Violations** when the user asked to verify-and-fix, or when
   verification runs as part of an implementation task you own. Fix **Warnings**
   on verify-and-fix when the preferred pattern clearly fits. Fix
   **Recommendations** only if the user asked to tidy / apply soft order.
   On principle fixes, obey **Built-in: DRY / KISS / YAGNI** conflict order so
   DRY does not fight KISS/YAGNI (and vice versa). Otherwise list findings and
   wait. Under `align-code` standalone: **report only**.
9. When implementing / verify-and-fix: **run** `flutter analyze` and
   `flutter test` from the app root; fix failures before claiming done.

Do not praise compliant code. Do not turn this into a product/bug review skill.
Do not hand-edit `lib/src/rust/**`.

## Built-in: Flutter / Effective Dart style

Sources: [Effective Dart](https://dart.dev/effective-dart), Flutter style common
practice, project `analysis_options.yaml` / `flutter_lints`. Soft guidance on
changed/new `lib/**/*.dart` (non-generated). Report as
`(skill: client-verify-code / Flutter style)`.

**Soft recommendations only — tidy order, do not enforce rigidly.**

- Goal: keep widgets and libraries readable with consistent structure.
- **Not** a hard lint beyond what `flutter analyze` already enforces.
- Prefer clear, safe tidy-ups over micro-nits.
- Severity: always **Recommendations** — unless verify-and-fix and reorder is
  clearly safe.

### Library / import order

Prefer:

1. `dart:` imports
2. `package:` imports (Flutter / third-party / app package)
3. Relative imports (`../…`) — only when already used in the file; prefer
   `package:my_translator_app/...` for app code when that is the project norm

Group with a blank line between groups. Suggest only obvious inversions.

### Widget / class member order

Inside a `StatefulWidget` / `State` / large widget class, prefer a skim-friendly
order:

1. Fields / statics
2. Constructor
3. Lifecycle (`initState`, `dispose`, …)
4. Build-related helpers
5. `build`

**Suggest** when lifecycle is buried after a huge `build`, or unrelated helpers
interrupt constructors. Do not demand a full reorder if it risks breakage.

### Const and Material

- Prefer `const` constructors/widgets where the analyzer already hints and the
  change set touches those call sites.
- Prefer Material / existing `ThemeData` tokens over one-off hardcoded colors
  when a theme is already in use — soft Prefer, not a redesign mandate.

### Empty lines between multi-line declarations

When adjacent multi-line declarations become hard to skim, prefer a blank line
between them. Do not flag readable dense single-line blocks.

## Built-in: Flutter conventions

Grounded in `{app}/AGENTS.md`. Report as
`(skill: client-verify-code / Flutter conventions)`.
Apply always; sibling skills win when they exist and conflict on a detail.

### Language and architecture

- **Dart** + **Flutter** widgets in `lib/`; UI and orchestration only.
- **Heavy logic in Rust** (parse, tokenize, lemmatize, dictionary, offline
  translate, SQLite). Do **not** reimplement those rules in Dart.
- Dependency direction:

  ```
  lib/**  →  lib/src/rust/api/* (generated)  →  rust/src/api/*  →  rust modules
  ```

- Optional AI providers later only when explicitly in scope.
- Prefer product screens (reader / split / dictionary / load) over expanding
  scaffold-demo leftovers once those screens exist.

### FRB / generated bindings

- Source of truth for FFI API: `rust/src/api/**/*.rs`.
- **Never hand-edit** `lib/src/rust/**` or `rust/src/frb_generated.rs`.
- After public API change in Rust `api/` → run FRB codegen → update Flutter call
  sites and tests.
- `main()`: `await RustLib.init()` **before** `runApp`.
- Do not put business logic in `rust_builder/` (glue / cargokit only).
- Prefer calling generated API from `package:my_translator_app/src/rust/api/...`
  rather than duplicating FFI shapes by hand.

### Async, streams, UI binding

- Await FRB futures before binding results into first paint when the UI depends
  on them; do not leave an await-gap that drops the first stream event when a
  skill/OpenSpec requires listening immediately after connect/subscribe.
- Do not block the UI isolate with heavy sync work that belongs in Rust.
- Do not swallow errors with empty `catch` / ignored `Future` errors on
  user-facing actions — surface failure in UI (banner / snackbar / error text)
  once patterns exist; until then, at least do not silently empty-catch.

### State, navigation, theme

- No global DI / router is required while scaffold has none; when adding state
  management or routing, keep one approach (do not invent a second parallel
  global store).
- Theme: prefer `ThemeData` / `ColorScheme` consistency with existing app theme
  over scattered magic numbers when changing UI chrome.
- Target product platforms: **Android, iOS, Web**; do not grow desktop-runner
  product UX without an explicit request.

### Data and persistence

- User data (dictionary, subtitles, settings) — through Rust (target SQLite),
  not ad-hoc Flutter-only persistence that forks the source of truth.
- Do not hardcode production secrets or API keys in `lib/`.

### Severity cues for built-in conventions

| Tier | Examples |
|------|----------|
| **Violations** | Hand-edited `lib/src/rust/**`; Dart reimplementation of parse/lemma/dict rules; `runApp` without `RustLib.init()`; empty `catch` on user-facing FRB calls; inventing a parallel FFI client that bypasses generated FRB |
| **Warnings** | Business logic creeping into widgets; Flutter-only persistence that forks Rust truth; expanding scaffold demo instead of target reader flows; blocking UI with work that belongs in Rust; FRB await/subscribe ordering that risks missing first event |
| **Recommendations** | Import / member order tidy; `const` where analyzer already hints; mild ThemeData token use; mild DRY/KISS polish |

## Built-in: DRY / KISS / YAGNI

Report as `(skill: client-verify-code / DRY-KISS-YAGNI)`. Apply to all checked
production files. Principles are **judgment lenses**, not absolute mandates.

**Core:** Prefer the simplest correct code that meets the real requirement.
Project skills and Flutter conventions **win** over DRY, KISS, and YAGNI. When
principles pull opposite ways, choose **one** outcome using the conflict order
below — never “satisfy DRY” by violating KISS/YAGNI or a domain skill, and never
“simplify” by deleting a required shared contract (including FRB layering).

### Verify lens

| Principle | Flag when | Do not flag when |
|-----------|-----------|------------------|
| **DRY** | Same *knowledge/rule* is duplicated in the change set and will drift if only one side changes | Similar-looking UI with different screens; intentional parallel helpers a skill requires; trivial short copies that stay clearer separate; Dart mirror of Rust types that FRB already generates |
| **KISS** | New indirection, factory, or clever layer that obscures the change without payoff | Required layering (Flutter UI → generated API → Rust); Material / theme patterns |
| **YAGNI** | Abstraction, extension point, or helper built for hypothetical future call sites (0–1 real uses of that shape) | Small helper with 2+ real call sites of the *same* shape already in the change |

### Conflict resolution (required before classify / fix)

Apply **in order**:

1. **Domain skills / Flutter conventions / FRB contracts** — do not dedupe or
   simplify away a required pattern (e.g. keep UI↔Rust split; do not “DRY” by
   hand-copying generated bindings).
2. **YAGNI** — remove or do not introduce unused / future-only abstractions.
3. **KISS** — prefer direct code over shared machinery when duplication is small
   or meanings differ.
4. **DRY** — extract only when identical knowledge would otherwise drift; the
   extraction must stay simple.

**Anti-conflict rule:** Do not report both “extract for DRY” and “inline for
KISS/YAGNI” on the same hunk. Pick one using the order above. On
verify-and-fix, apply that single outcome. Prefer **one finding per hunk** that
names the winning principle (mention secondary principles only as supporting
reason in the same bullet).

### Severity cues

| Tier | When |
|------|------|
| **Violations** | Rare for pure principles. Only if a new premature abstraction **breaks** a required skill/contract (e.g. bypasses FRB), or critical contract knowledge is duplicated **and already diverges** in the same change set |
| **Warnings** | Identical-knowledge duplication likely to drift; unused/future-only abstraction; complexity that obscures a preferred skill pattern |
| **Recommendations** | Mild duplication or slightly overcomplicated local code where a simpler safe shape is obvious |

### Red flags — STOP and re-resolve

- “Merge for DRY” when a skill / FRB contract says keep the split
- Shared util with one call site “for later” (YAGNI)
- Inlining a module that multiple real callers already need (false KISS)
- Opposing principle findings on the same hunk without conflict resolution
- “Simplify” by editing generated `lib/src/rust/**`

## Path hints (optional prioritization only)

Use only to decide **where to look harder**, never to drop a skill from Always
include.

| Change / path signals | Look harder at |
|----------------------|----------------|
| `lib/main.dart`, `RustLib.init`, `lib/src/rust/api/**` imports | Flutter conventions / FRB |
| New screens under `lib/` (reader, split, dictionary, load) | structure / pages / styles |
| Theme / `ThemeData` / colors | `work-with-styles`, Flutter style |
| Forms / TextField / validation | `work-with-forms` |
| State management folders | `work-with-stores` |
| l10n / `.arb` | `work-with-localization` |
| Error banners / SnackBar / async catch | `client-work-with-errors` |
| `analysis_options.yaml`, `pubspec.yaml` | tooling + conventions |
| Hand-touched `lib/src/rust/**` | **Violation** — generated |
| Soft import / class member order | Flutter style |

## High-signal checks (examples, not a full rulebook)

Reminders to **open the skill** (or Built-in) — skill text wins.

- **FRB**: no hand-edits under `lib/src/rust/**`; init before `runApp`; call
  generated API only.
- **Layering**: parse / lemma / dict / translate / SQLite stay in Rust.
- **Async**: no empty catch on user-facing FRB; no UI-thread heavy work that
  belongs in Rust; watch stream subscribe-before-await gaps when relevant.
- **Tooling**: `flutter analyze` + `flutter test` from app root.
- **Style** (soft): import groups; widget member order; `const` where hinted.
- **DRY/KISS/YAGNI**: conflict order — skills/FRB → YAGNI → KISS → DRY; one fix
  per hunk.

## Severity

Map each skill finding by how the source skill phrases the rule. Skill text wins;
when wording mixes levels, pick the strongest that still applies.

| Tier | When | Skill wording cues (examples) |
|------|------|-------------------------------|
| **Violations** | Hard break of a required pattern / contract | Do / Don't, Must, Never, Always, Hard Rules, hand-edit generated FRB, empty `catch`, Dart NLP/parse/dict authority |
| **Warnings** | Preferred pattern clearly fits; allowed exception does **not** apply | Prefer, Should, “use X instead of Y” when X fits |
| **Recommendations** | Soft tidy / style / optional polish | usually, Soft, Effective Dart / Flutter style, optional member reorder, mild DRY/KISS |

Do not invent findings outside the code skills and the built-in sections above.
Do not inflate Prefer into Violations. Analyzer/`flutter_lints` failures already
caught by `flutter analyze` → report under Violations (tooling), do not
duplicate as style Recommendations.

## Output

Report language: **English** for prose; keep paths, symbols, and code as-is.

Always include all three finding sections. If a section has no items, put a
single line `- none` (do not mix items and `none` in the same section).

When feeding [`align-code`](../../align-code/SKILL.md), paste this block under
the Flutter **Verify** section (align-code’s combined wrapper may be Russian;
keep this skill’s finding bullets in English unless the orchestrator asks
otherwise).

Format:

```text
## Verify Code

Scope: branch vs <merge-base> (<base-ref>) + staged + unstaged + untracked [+ path if used]
Repo: my_translator_app (lib/)
Skills: all client code skills (excl. locate/align/openspec/commit/mock) + Flutter style + Flutter conventions + DRY/KISS/YAGNI
Analyze/test: <commands run and pass/fail summary, or skipped with reason>

### Violations
- `path` — <what is wrong> (skill: `<name>`)
  Expected: <short correct pattern or cite skill section>

### Warnings
- `path` — <what is wrong> (skill: `<name>`)
  Expected: <short correct pattern or cite skill section>

### Recommendations
- none
```

Empty-section example: only `- none` under that heading. Item shape is the same
in every tier. Do not collapse Warnings/Recommendations into Violations or omit
soft Prefer findings.

## Red Flags — collect the branch first

These mean STOP and re-collect files vs merge-base before reporting:

- Ran only `git diff` / `--cached` / untracked
- “User didn’t ask for the whole branch”
- “Working tree has the real work”
- “I’ll check working tree first, branch later”
- Report Scope without `branch vs <merge-base>`
- Verified only `rust/` or skipped `lib/` for a Flutter verify request
- “Fixed” generated files under `lib/src/rust/**`

## Related Skills

All skills in the Always include table. Flutter soft style, Flutter conventions,
and DRY / KISS / YAGNI balance are owned by this skill when sibling files are
absent. Orchestrator: [`align-code`](../../align-code/SKILL.md) (Flutter Verify
axis). Mock pixel audit: [`verify-mock`](../../verify-mock/SKILL.md) — separate.
