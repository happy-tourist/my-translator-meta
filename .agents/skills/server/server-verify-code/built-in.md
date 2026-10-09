# Built-in checks — `server-verify-code`

Read from [`SKILL.md`](SKILL.md) when applying the Rust Verify pass. Report tags
use `(skill: server-verify-code / <section>)`.

## Built-in: Rust style

Sources: Rust API guidelines / idiomatic edition 2021 practice, `rustfmt`,
project `Cargo.toml` lints / Clippy when configured. Soft guidance on
changed/new `rust/src/**/*.rs` (non-generated). Report as
`(skill: server-verify-code / Rust style)`.

**Soft recommendations only — tidy order, do not enforce rigidly.**

- Goal: keep modules readable with consistent structure.
- **Not** a hard lint beyond what `cargo check` / Clippy already enforces.
- Prefer clear, safe tidy-ups over micro-nits.
- Severity: always **Recommendations** — unless verify-and-fix and reorder is
  clearly safe.

### Module / import order

Prefer:

1. `std` / `core` / `alloc`
2. External crates
3. `crate::` / `super::` / `self::` internals

Group with a blank line between groups when the file already does. Suggest only
obvious inversions. Prefer `rustfmt`-friendly layout; do not fight project
formatting.

### Item order in a module

Prefer a skim-friendly order:

1. Module docs / `//!`
2. `use` / `mod`
3. Constants / type aliases
4. Public types and inherent impls
5. Trait impls
6. Free functions (public then private)
7. Tests (`#[cfg(test)]`) at the bottom of the same file when co-located

**Suggest** when public API is buried under large private helpers, or tests
interrupt the public surface. Do not demand a full reorder if it risks breakage.

### Naming and idioms

- Prefer idiomatic Rust names (`snake_case` fns/modules, `PascalCase` types).
- Prefer `Result` + `?` over nested match pyramids when the change set already
  uses that style.
- Prefer borrowing / iterators over needless clones when the fix is local and
  obvious.

### Empty lines between multi-line items

When adjacent multi-line items become hard to skim, prefer a blank line between
them. Do not flag readable dense single-line blocks.

## Built-in: Rust conventions

Grounded in `{app}/AGENTS.md`. Report as
`(skill: server-verify-code / Rust conventions)`.
Apply always; sibling skills win when they exist and conflict on a detail.

### Language and architecture

- **Rust** edition **2021** in crate **`rust_lib_my_translator_app`**.
- **Heavy compute in Rust** (parse, tokenize, lemmatize, offline translate).
  Flutter is UI/orchestration + **Drift user persistence** — do **not** move
  NLP/parse into Dart or `rust_builder/`, and do **not** add user SQLite in Rust.
- Dependency direction:

  ```
  lib/** (UI + Drift)  →  lib/src/rust/api/* (generated)  →  rust/src/api/*  →  rust compute modules
  ```

- Optional AI providers later only when explicitly in scope.
- Prefer growing domain modules (parse / tokenize / translate)
  behind a **thin** `rust/src/api/` export over stuffing all logic into `api/`
  or leaving only scaffold demos once product APIs land.

### FRB / generated bindings

- Source of truth for FFI API: `rust/src/api/**/*.rs`.
- **Never hand-edit** `rust/src/frb_generated.rs` or `lib/src/rust/**`.
- After public API change in Rust `api/` → run FRB codegen
  (`flutter_rust_bridge_codegen generate`, CLI ≈ **2.13.0**) → update Flutter
  call sites and tests (Flutter side is owned by client skills; still flag if
  this change set hand-patches generated output instead).
- `#[frb(init)]` / `init_app` path must remain the init hook Flutter awaits via
  `RustLib.init()` before `runApp`.
- `rust_builder/` — **glue / cargokit only**; **no** business logic.
- Keep `api/` thin: validate / map types / call domain modules; do not embed
  full parsers or NLP pipelines inside `api/*.rs` when modules exist (or should).

### Errors and fallibility

- Prefer typed `Result` on fallible parse / IO / DB / translate paths over
  panicking on expected user/input failures.
- Do not swallow errors with empty `catch`-equivalents (`let _ = …`, ignored
  `Result`) on user-facing FRB entry points — propagate a clear error to the
  FFI boundary so UI can surface it.
- Reserve `unwrap` / `expect` for truly invariant cases; prefer `?` or mapped
  errors on public `api/` functions.

### Modules, persistence, config

- Expected growth: compute modules outside `api/` (parse, lemma/tokenize,
  translate) with thin export through `api/`.
- User data (dictionary, library folders/documents, settings) — **Drift in
  Flutter `lib/`**; do **not** add `rusqlite` / `diesel` / `rust/src/db` for
  user rows, and do not invent a second persistence authority in `rust_builder/`.
- Do not hardcode production secrets or API keys in `rust/` (optional AI later
  must use explicit config / secrets handling when in scope).
- Target product platforms via FRB: **Android, iOS, Web (WASM)**; do not grow
  desktop-only core assumptions without an explicit request.

### Severity cues for built-in conventions

| Tier | Examples |
|------|----------|
| **Violations** | Hand-edited `rust/src/frb_generated.rs` or `lib/src/rust/**`; business logic in `rust_builder/`; Dart reimplementation of parse/lemma/dict/translate rules; public API changed without `api/` + codegen path; empty/`unwrap` on user-facing FRB entry for expected failures; inventing a parallel FFI client that bypasses generated FRB |
| **Warnings** | Fat `api/` with domain logic that belongs in modules; persistence authority forking; expanding scaffold demo instead of target parse/NLP/dict APIs; panics on recoverable input; FRB types that force UI to re-derive lemma/match rules |
| **Recommendations** | Import / item order tidy; mild idiomatic `Result`/`?` polish; mild DRY/KISS polish; soft module split hints |

## Built-in: DRY / KISS / YAGNI

Report as `(skill: server-verify-code / DRY-KISS-YAGNI)`. Apply to all checked
production files. Principles are **judgment lenses**, not absolute mandates.

**Core:** Prefer the simplest correct code that meets the real requirement.
Project skills and Rust conventions **win** over DRY, KISS, and YAGNI. When
principles pull opposite ways, choose **one** outcome using the conflict order
below — never “satisfy DRY” by violating KISS/YAGNI or a domain skill, and never
“simplify” by deleting a required shared contract (including FRB `api/` thin
export + domain modules).

### Verify lens

| Principle | Flag when | Do not flag when |
|-----------|-----------|------------------|
| **DRY** | Same *knowledge/rule* is duplicated in the change set and will drift if only one side changes | Similar-looking helpers with different domain meaning; intentional thin `api/` wrappers a skill requires; trivial short copies that stay clearer separate; Dart mirrors of Rust types that FRB already generates |
| **KISS** | New indirection, factory, or clever layer that obscures the change without payoff | Required layering (Flutter UI → generated API → `rust/src/api` → domain modules); FRB init/codegen steps |
| **YAGNI** | Abstraction, extension point, or helper built for hypothetical future call sites (0–1 real uses of that shape) | Small helper with 2+ real call sites of the *same* shape already in the change |

### Conflict resolution (required before classify / fix)

Apply **in order**:

1. **Domain skills / Rust conventions / FRB contracts** — do not dedupe or
   simplify away a required pattern (e.g. keep thin `api/` vs domain split; do
   not “DRY” by hand-copying generated bindings).
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
| **Violations** | Rare for pure principles. Only if a new premature abstraction **breaks** a required skill/contract (e.g. bypasses FRB / puts logic in `rust_builder/`), or critical contract knowledge is duplicated **and already diverges** in the same change set |
| **Warnings** | Identical-knowledge duplication likely to drift; unused/future-only abstraction; complexity that obscures a preferred skill pattern |
| **Recommendations** | Mild duplication or slightly overcomplicated local code where a simpler safe shape is obvious |

### Red flags — STOP and re-resolve

- “Merge for DRY” when a skill / FRB contract says keep the split
- Shared util with one call site “for later” (YAGNI)
- Inlining a module that multiple real callers already need (false KISS)
- Opposing principle findings on the same hunk without conflict resolution
- “Simplify” by editing generated `frb_generated.rs` / `lib/src/rust/**`
- “Share” by moving NLP/parse/dict into `rust_builder/` or Dart
