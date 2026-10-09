---
name: work-with-frb-api
description: >-
  Designs the public flutter_rust_bridge API in my_translator_app rust/src/api/
  — modules and FFI-friendly types for Flutter, thin wrappers over parse /
  tokenize / lemmatize / dictionary / translate / init, and the Rust-side
  codegen workflow after an API change. Use when adding or changing public
  Rust FRB exports, Result error shapes on the bridge, or scaffolding product
  API areas while api/ stays thin. Complements client work-with-frb (Flutter
  bind / init / call sites); not app settings or flutter_rust_bridge.yaml
  (work-with-config).
---

# Work With FRB API (Rust public surface)

How agents design and evolve the **public FRB API** in **my_translator_app**
(Translation Reader). Canon: `{app}/AGENTS.md`.

**Authority:** `rust/src/api/**/*.rs` is the source of truth for what Flutter
may call. Generated Dart (`lib/src/rust/**`) and `rust/src/frb_generated.rs`
are outputs — never hand-edit.

**Version:** `flutter_rust_bridge` **2.13.0** (pin in `pubspec.yaml` and
`rust/Cargo.toml`). Codegen CLI ≈ `2.13.0`.

This skill owns **Rust-side API design** (modules, signatures, export types,
thin wrappers). Flutter bind / `RustLib.init` / call-site async → client
**`work-with-frb`**.

## When to Use

- Add, rename, split, or remove a public function/type under `rust/src/api/`
- Choose FFI-friendly shapes for cues, tokens, lemmas, dictionary entries
- Keep `api/` thin over new parse / NLP / dict / translate modules
- Run or reason about codegen **because the Rust API changed**
- Decide how `Result` / errors cross the bridge (see error section)

**Not this skill:** Flutter screens, `RustLib.init` wiring, leaf-widget FRB
calls, theme/i18n, or inventing parse/lemma algorithms inside `api/`.

## Layout (truth vs generated)

| Role | Path | Edit? |
|------|------|-------|
| Public FFI API | `rust/src/api/**/*.rs` | **Yes** — this skill |
| API module tree | `rust/src/api/mod.rs` | **Yes** (`pub mod …`) |
| Core logic | `rust/src/<domain>/…` (parse, nlp, dict, …) | **Yes** — not in `api/` |
| Generated Rust glue | `rust/src/frb_generated.rs` | **Never** |
| Generated Dart | `lib/src/rust/**` | **Never** |
| FRB config | `flutter_rust_bridge.yaml` | Rarely → **`work-with-config`** |
| Platform glue | `rust_builder/` | Glue only — no API surface |

Config today (`flutter_rust_bridge.yaml`):

```yaml
rust_input: crate::api
rust_root: rust/
dart_output: lib/src/rust
```

Crate root: `rust/src/lib.rs` exposes `api` + `frb_generated`.

## Scaffold Honesty (current)

Today the only public module is `simple`:

| Rust (`api/simple.rs`) | Behavior now | Product role |
|------------------------|--------------|--------------|
| `process_text(input: String) -> String` | `to_uppercase` | Demo only — **not** a parse/translate API |
| `add_numbers(a: i32, b: i32) -> i32` | `a + b` | Demo only |
| `init_app` (`#[frb(init)]`) | `setup_default_user_utils()` | Real init hook — keep / extend carefully |

`api/mod.rs` is only `pub mod simple;`.

**Do not** grow product features by stuffing logic into `process_text`. Add
named modules/functions for parse / tokenize / dictionary / translate, then
retire or leave demo helpers until UI no longer needs them.

## Core Rules

1. **`api/` is thin.** Public fns map args → call domain modules → return
   FFI-safe values. No parsers, SQLite, or NLP loops inside `api/*.rs`.
2. **Heavy logic lives outside `api/`.** Target: `parse`, tokenize/lemmatize,
   dictionary CRUD, translate, db — under `rust/src/…` (exact folder names
   follow existing crate layout when present).
3. **Never edit generated files.** Fix `rust/src/api/` (or domain code), then
   regenerate.
4. **One change = API + codegen + Dart call sites + tests** (see workflow).
   Do not leave Flutter on stale generated names.
5. **Prefer stable, explicit names** over reusing demo `process_text` for
   product work.
6. **Export only what Flutter needs.** Internal Rust types stay private;
   define thin DTO structs in `api/` (or a shared export crate module) when
   domain types are not FRB-friendly.

## Target API Areas

Grow toward these areas (names illustrative — match OpenSpec / AGENTS when
they specify):

| Area | Responsibility | Notes |
|------|----------------|-------|
| **init** | `#[frb(init)]` bootstrap | Extend `init_app`; do not add a second init without reason |
| **parse** | SRT / VTT / ASS → cues (text, timecodes) | Bytes/path/string in → structured cues out |
| **tokenize / lemmatize** | Tokens + lemmas for tap & highlight | Pair with language codes as needed |
| **dictionary** | Add / list / remove / lookup by lemma | CRUD + context fields; persistence in domain/db |
| **translate** | Offline word/phrase translate | Keep provider/AI behind domain; thin export |
| **(later) AI** | Optional explain/translate | Only when product opts in; still thin `api/` |

Register each new file in `api/mod.rs` (`pub mod parse;`, …).

## Workflow: Edit API → Codegen → Call Sites

```text
1. Design signatures + FFI types (this skill)
2. Implement domain logic under rust/src/<domain>/ (not in api/)
3. Add thin wrappers in rust/src/api/<module>.rs + mod.rs
4. From app root: flutter_rust_bridge_codegen generate
5. Update Flutter call sites / orchestration / tests (client work-with-frb)
6. cargo check|test in rust/; flutter analyze / flutter test
```

Naming: Rust `snake_case` → Dart `camelCase` with named params
(`parse_subtitles` → `parseSubtitles(...)`). Prefer `async` Rust for work that
must not block the UI isolate; Dart will typically see a `Future`.

If codegen fails: fix **exportable** types or API shape in `api/` — do not
patch generated output.

## FFI-Friendly Types

Prefer types FRB 2.x exports cleanly:

| Prefer | Avoid on the public surface |
|--------|-----------------------------|
| `String`, `bool`, numeric primitives | Raw OS handles, unbound lifetimes |
| `Vec<T>`, simple nested structs | Complex graphs with cycles |
| `Option<T>`, plain enums | Trait objects (`Box<dyn …>`) as returns |
| Owned values | Non-`'static` references in signatures |
| Explicit DTO structs | Leaking `diesel` / internal ORM types |
| `Result<T, E>` with a clear `E` | Panics for expected failures |

Guidelines:

- Mirror UI needs (cue list, token spans, dictionary rows) — not full internal
  domain graphs.
- When domain types are rich, map to `api` DTOs in the thin wrapper.
- Stable field names matter: renaming a public struct field is a breaking
  Dart change after codegen.
- Streams / callbacks: only when the product needs push updates; otherwise
  prefer request/response `async fn` → `Future`.

## Errors Across the Bridge

Expected failures (bad file, unsupported format, missing entry) should use
**`Result`** (or FRB-supported error mapping), not `unwrap`/`panic` for
control flow.

- Shape and map Rust errors for the bridge → sibling
  **`server-work-with-errors`** (Result / anyhow / typed codes).
- Flutter mapping of `AnyhowException` / panic → user copy → client
  **`client-work-with-errors`**.

Do not invent ad-hoc stringly error protocols in `api/` without aligning those
skills. Prefer stable codes or typed enums Flutter can switch on once they
exist.

## Boundary vs Sibling Skills

| Skill | Owns | Does not own |
|-------|------|--------------|
| **`work-with-frb-api`** (this) | `rust/src/api/` design, export types, thin wrappers, Rust-triggered codegen | Flutter init/call sites, UI async, theme |
| **`work-with-frb`** (client) | `RustLib.init`, calling generated Dart, FFI vs WASM from UI, stream bind pitfalls | Inventing Rust signatures / domain logic |
| **`work-with-config`** | App settings / runtime config, rare `flutter_rust_bridge.yaml` / Cargo feature wiring | Product FFI function design in `api/` |
| **`server-work-with-errors`** | Rust Result / error types and mapping policy | Dart SnackBar / empty-state UX |
| **`client-work-with-structure`** | Where Flutter orchestration lives | Rust module layout |

**Split of labor on one change:** this skill designs/implements `api/` +
domain; client `work-with-frb` updates Dart callers after codegen. Do not
duplicate long Flutter init guidance here.

## Module Pattern (thin wrapper)

Illustrative — replace body with real domain calls when modules exist:

```rust
// rust/src/api/parse.rs  (target — not in scaffold yet)
use crate::parse; // domain module when added

pub struct CueDto {
    pub index: i32,
    pub start_ms: i64,
    pub end_ms: i64,
    pub text: String,
}

pub fn parse_subtitles(content: String, format_hint: Option<String>) -> Result<Vec<CueDto>, String> {
    let doc = parse::parse_string(&content, format_hint.as_deref())?;
    Ok(doc.cues.into_iter().map(CueDto::from).collect())
}
```

Rules of thumb for wrappers:

- Convert at the boundary; do not re-implement parse inside the `api` fn.
- Keep `init_app` free of long-running I/O; reserve heavy work for explicit
  API calls after UI is up.
- Document public fns briefly (what Flutter gets), not internals.

## Checklist (API change)

```text
- [ ] Domain logic lives outside api/
- [ ] New/changed exports are FFI-friendly
- [ ] api/mod.rs registers new modules
- [ ] Expected failures use Result (see server-work-with-errors)
- [ ] flutter_rust_bridge_codegen generate from app root
- [ ] No hand-edits under lib/src/rust/ or frb_generated.rs
- [ ] Flutter call sites + tests updated (work-with-frb)
- [ ] cargo check|test; flutter analyze / flutter test
- [ ] Demo process_text / add_numbers not overloaded for product
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Heavy parse/NLP/SQL inside `api/*.rs` | Move to domain module; thin wrap |
| Hand-edit `lib/src/rust/**` or `frb_generated.rs` | Change `api/` → codegen |
| Export non-FRB types (ORM, lifetimes, `dyn`) | DTO + owned fields |
| Panic on bad subtitle input | `Result` + server-work-with-errors |
| Skip codegen after `api/` edit | Regenerate; fix Dart callers |
| Grow `process_text` into the product API | New named modules/fns |
| Change only Rust and leave Flutter/tests stale | Same change updates call sites |
| Put settings/YAML redesign in this skill | `work-with-config` |
| Duplicate Flutter init docs here | `work-with-frb` |

## Quick Reference

| Task | Action |
|------|--------|
| New public fn | Domain module → thin `api/` → `mod.rs` → codegen → Dart |
| New export type | FFI-safe struct/enum in `api/` (or mapped DTO) |
| Rename API | Edit Rust → codegen → update all Dart/tests |
| Init hook | `#[frb(init)]` in `api/` (scaffold: `init_app`) |
| Codegen | `flutter_rust_bridge_codegen generate` (CLI ≈ 2.13.0) |
| Errors | `Result` + **`server-work-with-errors`** |
| Flutter bind | **`work-with-frb`** |

## Related

- App canon: `{app}/AGENTS.md`
- Client bind/init/codegen usage: `.agents/skills/client/work-with-frb/`
- Flutter FRB error UX: `.agents/skills/client/client-work-with-errors/`
- Rust error policy: `.agents/skills/server/server-work-with-errors/` (when present)
- Config / FRB yaml: `.agents/skills/server/work-with-config/` (when present)
