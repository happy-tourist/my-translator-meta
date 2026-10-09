---
name: server-verify-code
description: >-
  Use when the user asks to verify code, check skill compliance, audit a branch
  diff vs master/main/merge-base, audit local diffs, or after implementing a
  change in my_translator_app Rust rust/. Also when checking Rust edition 2021 /
  FRB 2.13 api surface, crate layout, error handling, or DRY / KISS / YAGNI
  balance on changed files. Reports three tiers: Violations, Warnings,
  Recommendations. Feeds the Rust Verify axis of align-code.
---

# Verify Code — Rust (`rust/`)

Use this skill to check **all production Rust code changed on the current
branch** in **`my_translator_app`** (`rust/`) against **all code-related** project
skills under `.agents/skills/server/`, plus the **built-in** checks in this file
(Rust / FRB soft style, Rust+FRB conventions, and DRY / KISS / YAGNI with
conflict-aware judgment).

Default scope is the **full branch diff**: commits vs merge-base **and**
uncommitted working-tree changes. Do not stop at staged/unstaged/untracked.

This skill is primarily an **orchestrator**: domain rules live in the code
skills listed below — **read those skill files** when present and apply them; do
not restate or invent parallel domain rules here. Built-in sections below fill
gaps when a listed skill file is missing, and own Rust soft style guidance
plus DRY / KISS / YAGNI balance.

**Paths:** skills live in **meta** at `.agents/skills/server/` (this repo =
`my_translator_meta`). Runtime code and `rust/…` paths are relative to the
**app** package root (`my_translator_app`). Resolve app via
[`docs/projects-map.md`](../../../../docs/projects-map.md) (default
`../my_translator_app`). Before work, read `{app}/AGENTS.md` when present.

**Stack assumptions:** Rust edition **2021**, crate
**`rust_lib_my_translator_app`** (`cdylib` + `staticlib`),
**flutter_rust_bridge** **2.13** — public FFI in `rust/src/api/**`, generated
`rust/src/frb_generated.rs` (never hand-edit). Heavy logic (parse, tokenize,
lemmatize, dictionary, offline translate, SQLite) lives in Rust modules outside
thin `api/` exports. Verify with `cargo` in `{app}/rust`.

**Out of scope for this skill:** Flutter UI compliance (`lib/` widgets,
Material, l10n) and `flutter analyze` / `flutter test` — those belong to the
Flutter Verify axis of [`align-code`](../../align-code/SKILL.md) /
[`client-verify-code`](../../client/client-verify-code/SKILL.md). Flag only
Rust-side contract misuse and cross-layer authority breaks (hand-edited
generated bindings, business logic in `rust_builder/`, Dart reimplementation of
core rules visible in the change set).

## When To Use

- User asks to verify code / skill compliance / convention check for Rust
- After implementing a feature or refactor in `rust/`, before commit
- When reviewing the current branch (or a named path) for Rust convention drift
- As the **Rust Verify** pass inside [`align-code`](../../align-code/SKILL.md)
  (report-only unless orchestrator override asks to fix)

## Scope: code skills only

Skills live under **meta** `.agents/skills/server/<name>/SKILL.md`.

### Always include (read and apply each when the file exists)

Verify against **every** skill in this set for the checked code — not a subset
guessed from the path. Path routing below only helps prioritize deeper reading;
it does **not** allow skipping skills from this list.

| Skill | Concern (when present) |
|-------|------------------------|
| `server-work-with-structure` | `rust/` layout, `api/` vs domain modules, crate modules |
| `server-work-with-errors` | `Result` / error types, empty `unwrap`/`expect` on user paths, FRB error surfacing |
| `work-with-config` | `Cargo.toml`, FRB yaml, feature flags, env for optional providers |
| `work-with-frb-api` | public FFI surface in `rust/src/api/**`, thin exports, codegen contract |
| `work-with-parse` | subtitle parse (SRT / VTT / ASS) |
| `work-with-tokenizematize` | tokenize + lemmatize / wordform match |
| `work-with-dictionary` | personal dictionary CRUD + lemma lookup + SQLite |
| `work-with-translate` | offline word/phrase translate (and later optional AI providers) |

If a new code skill appears under `.agents/skills/server/` (same kind: how to
write Rust core code), include it too. Prefer reading one extra skill over
missing a rule. Do **not** auto-include `server-work-with-test` here — it is
test-writing only (see Always exclude).

If a listed skill file is **missing**, do not invent a parallel rulebook — apply
**Built-in: Rust conventions** for that concern and continue.

### Always exclude

Do **not** use these for server-verify-code (unless the user explicitly asks):

| Skill | Why excluded |
|-------|----------------|
| `server-locate-change-points` | planning where to edit |
| `server-verify-code` | this orchestrator |
| `server-align-code` / `align-code` | alignment / постановка (separate axis) |
| `server-work-with-test` | test-writing conventions (not used to audit prod code) |
| `commit` | commit messages |
| `openspec` / opsx skills | change workflow / specs |
| `verify-mock` / `prepare-mock` | pixel/mock audit (Flutter UI) |
| `align-implementability-*` / `align-meta` | meta / implementability, not prod Rust audit |
| client `*` skills under `.agents/skills/client/` | Flutter UI axis — use `client-verify-code` |

Skip verifying files that are **only** tests (`rust/**/tests/**`,
`rust/**/*_test.rs`, `#[cfg(test)]` modules that are test-only files) unless the
user explicitly asks to include them. Focus on production/source code under
`rust/src/` (excluding generated FRB output — see below). Include
`rust/Cargo.toml` / lockfile only when product-relevant (deps, features, lints).

**Never treat as editable production sources:**

- `rust/src/frb_generated.rs` (FRB-generated Rust glue)
- `lib/src/rust/**` (FRB-generated Dart — Flutter side; do not “fix” from here)

If the change set **touches** generated bindings, report a **Violation** when
they were hand-edited or when public API was changed without going through
`rust/src/api/` + FRB codegen. Do not invent clippy/lint findings inside
generated files.

**Always flag in the change set (hard Violations):**

- Hand-edited `rust/src/frb_generated.rs` or `lib/src/rust/**`
- Business logic (parse / tokenize / lemma / dictionary / translate / SQLite
  rules) placed in `rust_builder/` (glue / cargokit only)
- Dart reimplementation of those core rules (authority forked out of Rust) when
  visible in the same change set

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

Union of 2–5 is the file set (filter to Rust production sources under `rust/`
plus `rust_builder/` when touched for glue vs logic; flag Dart core-rule forks
if present). Never skip step 2 because the working tree looks small.

Narrow only when the user **explicitly** asks for working-tree-only / only staged,
or names a **folder or path**.

Skip unrelated noise (lockfiles unless dep policy changed, `rust/target/`,
`build/`, `.dart_tool/`, platform runner boilerplate with no product change,
binary assets) unless the user asked to include them. Do not expand into Flutter
UI audits for this skill (except the authority-fork flags above).

## Commands (before done)

Run tooling from **`{app}/rust`** (crate root: `Cargo.toml` for
`rust_lib_my_translator_app`). Fix failures before claiming done when
verify-and-fix / implementation owns the change. Do not skip check / test after
changes unless the user explicitly asked for a **report-only** pass (as in
standalone `align-code` Verify — include command results in the report, do not
auto-fix unless override).

| Command | Purpose |
|---------|---------|
| `cargo check` | compile / typecheck the crate |
| `cargo test` | unit / integration tests in the crate |
| `cargo clippy` | lints when Clippy is available (toolchain or project config) |

Run from `{app}/rust`:

```bash
cd rust && cargo check && cargo test
```

If Clippy is present (`cargo clippy --version` succeeds, or the crate /
workspace defines clippy lints / `.clippy.toml`), also run:

```bash
cd rust && cargo clippy -- -D warnings
```

Prefer the project’s documented clippy invocation when one exists; otherwise
`cargo clippy` without inventing extra `-W` groups beyond what `Cargo.toml` /
clippy config already sets. Do **not** run `flutter analyze` / `flutter test`
from this skill. Do not treat passing `cargo check` as a substitute for skill
checks or missing tests from OpenSpec tasks.

Report failed check/test/clippy output under **Violations** (tooling) with the
command and a short failure summary. Do not invent lint rules beyond the crate’s
`Cargo.toml` / clippy config and skill text.

### align-code Verify expectations (Rust)

When invoked as the Rust Verify axis of `align-code`:

- Include tooling results (`cargo check`, `cargo test`, clippy if present) in
  the Verify block
- Classify skill/convention findings as Violations / Warnings / Recommendations
- Stay report-only unless the orchestrator override asks to fix
- Do not mix Flutter UI findings into this skill’s output; label paths under
  `rust/` (and `rust_builder/` / generated when flagging)

## Workflow

1. Resolve **app** path via projects-map; collect the file set: merge-base…HEAD
   **plus** staged, unstaged, untracked (or the path the user named). Exclude
   test-only and generated FRB files per Scope (still flag hand-edits).
2. **Read all skills from the Always include table** that exist on disk (do not
   rely on memory; do not skip because the path “looks unrelated”).
3. For each source file, apply every code skill whose rules can touch that file.
   When unsure, apply the skill. If missing, use Built-in Rust conventions.
4. For every checked `.rs` production file under `rust/src/` (non-generated),
   apply **Built-in: Rust style**.
5. Apply **Built-in: Rust conventions** to all checked production files.
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
9. When implementing / verify-and-fix: **run** `cargo check` and `cargo test`
   from `{app}/rust`; run `cargo clippy` when present; fix failures before
   claiming done.

Do not praise compliant code. Do not turn this into a product/bug review skill.
Do not hand-edit `rust/src/frb_generated.rs` or `lib/src/rust/**`.

## Built-in checks

Read and apply **[built-in.md](built-in.md)** for every verify pass:

- Rust style (soft Recommendations)
- Rust conventions / FRB contracts (Violations–Recommendations)
- DRY / KISS / YAGNI with conflict order

Do not invent parallel built-in rules in chat; skill + topic win.

## Path hints (optional prioritization only)

Use only to decide **where to look harder**, never to drop a skill from Always
include.

| Change / path signals | Look harder at |
|----------------------|----------------|
| `rust/src/api/**`, `api/mod.rs`, `#[frb(…)]` | `work-with-frb-api`, Rust conventions / FRB |
| `rust/src/frb_generated.rs`, `lib/src/rust/**` | **Violation** if hand-edited — generated |
| `rust_builder/**` | glue only; **Violation** if business logic |
| New parse / SRT / VTT / ASS modules | `work-with-parse` |
| tokenize / lemma / morphology | `work-with-tokenizematize` |
| dictionary / SQLite / lemma lookup | `work-with-dictionary` |
| offline / phrase translate | `work-with-translate` |
| `Cargo.toml`, features, FRB yaml | `work-with-config` |
| `Result` / error types / `unwrap` on `api/` | `server-work-with-errors` |
| `rust/src/lib.rs`, module tree | `server-work-with-structure` |
| Soft import / item order | Rust style |

## High-signal checks (examples, not a full rulebook)

Reminders to **open the skill** (or Built-in) — skill text wins.

- **FRB**: no hand-edits of `frb_generated.rs` / `lib/src/rust/**`; public surface
  only via `rust/src/api/` + codegen; FRB **2.13**.
- **Layering**: parse / lemma / dict / translate / SQLite stay in Rust modules;
  `api/` thin; **no** business logic in `rust_builder/`; **no** Dart forks of
  those rules.
- **Errors**: no ignored `Result` / casual `unwrap` on user-facing `api/` for
  expected failures.
- **Tooling**: `cargo check` + `cargo test` from `{app}/rust`; Clippy when
  present.
- **Style** (soft): import groups; item order; idiomatic `Result`/`?`.
- **DRY/KISS/YAGNI**: conflict order — skills/FRB → YAGNI → KISS → DRY; one fix
  per hunk.

## Severity

Map each skill finding by how the source skill phrases the rule. Skill text wins;
when wording mixes levels, pick the strongest that still applies.

| Tier | When | Skill wording cues (examples) |
|------|------|-------------------------------|
| **Violations** | Hard break of a required pattern / contract | Do / Don't, Must, Never, Always, Hard Rules, hand-edit generated FRB, logic in `rust_builder/`, Dart NLP/parse/dict authority, empty/`unwrap` on expected user failures |
| **Warnings** | Preferred pattern clearly fits; allowed exception does **not** apply | Prefer, Should, “use X instead of Y” when X fits |
| **Recommendations** | Soft tidy / style / optional polish | usually, Soft, Rust style, optional item reorder, mild DRY/KISS |

Do not invent findings outside the code skills and the built-in sections above.
Do not inflate Prefer into Violations. `cargo`/`clippy` failures already caught
by tooling → report under Violations (tooling), do not duplicate as style
Recommendations.

## Output

Report language: **English** for prose; keep paths, symbols, and code as-is.

Always include all three finding sections. If a section has no items, put a
single line `- none` (do not mix items and `none` in the same section).

When feeding [`align-code`](../../align-code/SKILL.md), paste this block under
the Rust **Verify** section (align-code’s combined wrapper may be Russian;
keep this skill’s finding bullets in English unless the orchestrator asks
otherwise).

Format:

```text
## Verify Code

Scope: branch vs <merge-base> (<base-ref>) + staged + unstaged + untracked [+ path if used]
Repo: my_translator_app (rust/)
Skills: all server code skills (excl. locate/align/openspec/commit/test-writing) + Rust style + Rust conventions + DRY/KISS/YAGNI
Check/test/clippy: <commands run and pass/fail summary, or skipped with reason>

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
- Verified only `lib/` or skipped `rust/` for a Rust verify request
- “Fixed” generated files under `rust/src/frb_generated.rs` or `lib/src/rust/**`
- Ran non-Cargo tooling instead of `cargo` in `{app}/rust`

## Related Skills

All skills in the Always include table. Built-in style / conventions /
DRY-KISS-YAGNI: [built-in.md](built-in.md). Orchestrator:
[`align-code`](../../align-code/SKILL.md) (Rust Verify axis). Flutter twin:
[`client-verify-code`](../../client/client-verify-code/SKILL.md).
