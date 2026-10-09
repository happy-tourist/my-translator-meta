---
name: server-work-with-errors
description: >-
  Use when adding, changing, reviewing, or debugging Rust error handling in
  my_translator_app (Translation Reader) — Result / anyhow / thiserror at the
  FRB API boundary, parse / lemmatize / translate failures (user persist errors are Flutter/Drift),
  stable error codes or messages for Dart catch, or avoiding panics for expected
  user errors. Not Flutter UI presentation (client-work-with-errors).
---

# Work With Errors (Rust / FRB)

Use this skill when working with failure creation and propagation under `rust/`
in **my_translator_app** (product: **Translation Reader**).

Stack (from `{app}/AGENTS.md`): Rust edition 2021 crate
`rust_lib_my_translator_app`, public FFI in `rust/src/api/`, heavy modules
outside `api/` (parse, tokenize/lemmatize, dictionary, translate, db). Bridge:
**flutter_rust_bridge** 2.13 (in-process).

Paths resolve via `docs/projects-map.md` key `app` → `../my_translator_app`.
Runtime paths below are relative to that app root.

**Ownership split:**

| Layer | Owns |
|-------|------|
| This skill (`server-work-with-errors`) | Rust `Result` / error types, mapping at `api/`, logging, no panic for expected failures |
| `client-work-with-errors` | Dart catch, user-safe copy, SnackBar / inline / fatal UI |
| `work-with-frb` | Codegen, init, generated file rules — not product error taxonomy |

Related (by name — load when in scope): `work-with-frb`,
`client-work-with-errors`, `align-implementability-back`,
`server-work-with-structure` / `server-work-with-test` when present.
App canon: `{app}/AGENTS.md`.

## Core Model

Failures are **in-process**: Rust function returns `Result` (or typed error)
→ FRB turns `Err` into a Dart-catchable exception → Flutter orchestration maps
to UI (`client-work-with-errors`).

```
Domain module:  Result<T, DomainError> or Result<T, anyhow::Error>
     ↓ map / ?
api/ (FRB):     Result<T, anyhow::Error>  (or FRB-exported typed error)
     ↓ codegen
Dart:           throws AnyhowException (message)  / typed exception when API adds it
     ↓
UI skill:       mapFrbUserMessage → SnackBar / inline / fatal
```

Prefer **clear, stable failure signals** the client can map — not panics, not
raw `Display` of nested crate chains for expected user mistakes.

## Cargo Dependencies (current vs target)

`rust/Cargo.toml` today pins only `flutter_rust_bridge = "=2.13.0"`. Scaffold
APIs (`process_text`, `add_numbers`) return plain values and do not yet model
errors.

When product APIs need fallible paths, **add crates deliberately** (do not
assume they exist):

| Crate | Role | When |
|-------|------|------|
| `anyhow` | Ergonomic `Result` + context in modules and thin `api/` | Default for FRB `Result<T, anyhow::Error>` → Dart `AnyhowException` |
| `thiserror` | Typed domain enums (`ParseError`, `DictError`, …) inside Rust | Preferred for module boundaries; map to anyhow or FRB type at `api/` |
| Neither yet | Scaffold | OK until first fallible product API |

Do **not** invent other error frameworks (`eyre` unless chosen) without an
explicit product/stack decision. Prefer the smallest set: **thiserror in
domain + anyhow (or typed) at FRB surface**.

## Exception Sources (Rust → Dart)

| Rust outcome | Typical Dart type | Rule on Rust side |
|--------------|-------------------|-------------------|
| `Err(anyhow::Error)` from FRB fn | `AnyhowException` (`message`) | Stable, mappable message or code prefix; no backtrace in user-facing string |
| FRB-exported typed error enum | Generated Dart exception / enum | Prefer when client needs structured branches |
| `panic!` / unwrap on bug | `PanicException` | **Only** for programming bugs; never for bad user files or missing lemmas |
| Sync success with empty data | normal `Ok(empty)` | Empty list / zero cues is **not** `Err` — UI empty state |

Import/codegen details → `work-with-frb`. UI mapping → `client-work-with-errors`.

## Domain Failure Channels

| Domain | Expected user/ops failures | Pattern |
|--------|---------------------------|---------|
| Parse (SRT/VTT/ASS) | Unsupported format, corrupt file, unreadable bytes | `Err` with stable code/prefix (`unsupported_format`, `parse_failed`, `empty_cues` only if product treats empty as error — prefer `Ok` + empty) |
| Tokenize / lemmatize | Unsupported language, empty input, engine failure | `Err` for hard fail; empty token list as `Ok(vec![])` when input was valid but blank |
| Dictionary (NLP handoff later) | Missing lemma key / bad input to compute | Typed or coded `Err`; not panic — **persist CRUD errors live in Flutter/Drift** |
| Translate (offline) | Unknown word, missing dict data, engine unavailable | `Err` or structured “not found” — pick one contract and keep it; document in API |
| User persistence | I/O, migration, constraint | **Flutter Drift** (not Rust SQLite) — map in client error skill |
| Settings load/save | Corrupt prefs / I/O | Flutter Drift later; caller keeps prior in-memory defaults |
| `init_app` | Setup failure | Fail loudly at init (Dart fatal UI); do not half-init |

## API Boundary (`rust/src/api/`)

Public FFI functions are the **contract**. Keep them thin: call domain modules,
map errors, return `Result`.

### Preferred shape (anyhow at FRB)

```rust
use anyhow::{anyhow, Context, Result};

/// Parse subtitle bytes. On failure returns Err for Dart AnyhowException.
pub fn parse_subtitles(path: String) -> Result<SubtitleDoc> {
    let bytes = std::fs::read(&path)
        .with_context(|| format!("read_failed:{path}"))?;
    crate::parse::parse_bytes(&bytes)
        .map_err(|e| anyhow!("parse_failed:{}", e))
}
```

Guidelines:

- Return `Result<T, E>` for fallible product work. Do not return “success with
  error string field” unless an OpenSpec change explicitly requires a soft
  result object.
- Put **stable tokens** early in the message (or in a typed code field) so
  Flutter `mapFrbUserMessage` can branch: `unsupported_format`, `parse_failed`,
  `empty_file`, `dict_not_found`, `db_failed`, `translate_unavailable`.
- Use `.context("code")` / `.with_context` for I/O chains; keep the **outer**
  message short and code-first.
- Do **not** include absolute paths with user home segments, SQL, or full
  anyhow chain dumps as the only message if a short code+hint suffices.
- After changing signatures → `flutter_rust_bridge_codegen generate` → update
  Dart call sites (`work-with-frb`). Never hand-edit `lib/src/rust/**` or
  `rust/src/frb_generated.rs`.

### Typed domain errors (thiserror) → map at api/

```rust
// in rust/src/parse/error.rs (module, not necessarily FRB-exported)
#[derive(Debug, thiserror::Error)]
pub enum ParseError {
    #[error("unsupported_format")]
    UnsupportedFormat,
    #[error("parse_failed:{0}")]
    Corrupt(String),
}

// in rust/src/api/parse.rs
pub fn parse_subtitles(path: String) -> Result<SubtitleDoc> {
    crate::parse::parse_path(&path).map_err(Into::into) // into anyhow::Error
}
```

Export a typed FRB error enum only when Flutter must `switch` on variants
without string scraping. Until then, anyhow + stable prefixes is enough and
matches `client-work-with-errors`.

### Scaffold today

`process_text` / `add_numbers` are infallible demos. When replacing them with
parse/lemma/dict/translate APIs, introduce `Result` at the new functions —
do not force demo APIs to return `Result` “for consistency” unless they become
fallible.

## Never Panic For Expected User Errors

| Situation | Do | Do not |
|-----------|----|--------|
| Bad subtitle file | `return Err(...)` | `unwrap` / `expect` / `panic!` |
| Unknown lemma / no translation | `Err` or structured miss | panic |
| DB constraint / missing row | `Err` | `unwrap` on query |
| Bug: invariant violated internally | `debug_assert!` / `unreachable!` only if truly unreachable; prefer `Err` if recoverable across FFI | `unwrap` on user input |
| FFI init | fail init path | catch-and-ignore |

Panics across FRB become `PanicException`. The UI skill **must not** show panic
bodies; Rust must not rely on panic as the user-facing channel for parse/dict
mistakes.

Avoid `unwrap`/`expect` on:

- File reads of user-chosen paths
- Parse results
- DB queries that may miss
- Optional dict/translate lookups

Acceptable: `unwrap` on mutex poison after documented abort, or in unit tests.

## Mapping And Propagation Inside Rust

```
parse::parse_bytes → Result<Doc, ParseError>
dict::add_entry    → Result<(), DictError>
api::parse_*       → Result<Doc, anyhow::Error>   // map_err / ?
api::dict_*        → Result<(), anyhow::Error>
```

- Use `?` in domain and api; map once at the boundary if types differ.
- Do not catch-all and return `Ok(Default::default())` for real failures —
  that forces Flutter into empty-state UX for broken files.
- Do not log-and-swallow in `api/` then return success.
- Preserve enough context for logs (`tracing`/`log`/`eprintln` — only if the
  crate already uses or product adds them); default scaffold has no logger —
  `eprintln`/`log` only when intentionally added.

## Logging (Rust)

Keep logging optional and quiet unless observability is an explicit add:

- Prefer returning `Err` over logging alone.
- If logging: include domain tag (`[parse]`, `[dict]`, `[db]`) and the stable
  code; avoid duplicating the entire file contents.
- Never log secrets (future API keys). Paths may be sensitive — prefer
  filename / error code in info logs.

## Checklist For New Failures

1. Identify domain: parse / lemma / dict / translate / db / settings / init.
2. Choose `Err` vs empty `Ok` (empty data ≠ error unless product says so).
3. Use typed `thiserror` in the module **or** anyhow with a **stable code**.
4. Map at `rust/src/api/` to FRB-friendly `Result` (no panic).
5. Message/code is something `client-work-with-errors` can map; coordinate
   prefixes with Flutter helpers.
6. Codegen after API signature change; update Rust unit tests for reject paths.
7. Do not implement SnackBar/dialog text in Rust — return codes/messages only.

## Patterns And Examples

### 1. Parse reject (expected)

```rust
pub fn parse_bytes(bytes: &[u8]) -> Result<SubtitleDoc, ParseError> {
    if bytes.is_empty() {
        return Err(ParseError::Empty);
    }
    let kind = detect_format(bytes).ok_or(ParseError::UnsupportedFormat)?;
    parse_kind(kind, bytes).map_err(|e| ParseError::Corrupt(e.to_string()))
}
```

### 2. Dictionary miss vs hard failure

```rust
pub fn lookup_lemma(lemma: &str) -> Result<Option<DictEntry>> {
    let conn = db::conn().context("db_failed")?;
    Ok(dict::find(&conn, lemma).context("db_failed")?)
}
```

`Ok(None)` = not in dictionary (UI: no highlight / “not saved”).  
`Err` = DB broken (UI: error + retry). Do not conflate.

### 3. Translate unavailable

```rust
pub fn translate_word(lemma: String) -> Result<String> {
    crate::translate::offline(&lemma)
        .ok_or_else(|| anyhow!("translate_unavailable"))
}
```

### 4. Anti-pattern (do not)

```rust
pub fn parse_subtitles(path: String) -> SubtitleDoc {
    // BAD: panics on bad files → PanicException, unusable UX contract
    crate::parse::parse_path(&path).unwrap()
}
```

## Common Mistakes

- `unwrap` / `expect` / `panic!` on user files, missing rows, or unknown lemmas.
- Returning `Ok` with garbage/default on parse failure so UI shows a blank reader.
- Free-form anyhow messages with no stable prefix — Flutter cannot map reliably.
- Putting user-facing English/Russian essays only in Rust while Dart still
  scrapes `e.toString()` — keep Rust messages **code-first**; UI owns polished copy.
- Hand-editing generated Dart/Rust FRB files to “add” error types.
- Leaking SQL/`Debug` dumps as the FRB error string for expected failures.
- Treating empty vocab / zero cues as `Err` when product wants empty state.
- Catching errors in `api/` and returning success after `eprintln` only.
- Introducing `anyhow`/`thiserror` usage in docs/examples without adding them to
  `Cargo.toml` when implementing (skill may recommend; implementation must add).

## Key Files (target)

| Path | Role |
|------|------|
| `rust/Cargo.toml` | Add `anyhow` / `thiserror` when fallible APIs land |
| `rust/src/api/**/*.rs` | FRB surface: `Result` mapping, stable codes |
| `rust/src/api/mod.rs` | Module exports |
| `rust/src/parse/` (target) | Parse errors — unsupported / corrupt |
| `rust/src/` lemma/tokenize modules (target) | NLP failures |
| Flutter Drift repositories | User persist errors (not Rust) |
| `rust/src/` translate (target) | Offline miss / unavailable |
| `rust/src/frb_generated.rs` | GENERATED — never hand-edit |
| `lib/src/rust/**` | GENERATED Dart — never hand-edit |
| `{app}/AGENTS.md` | Layering, init order |

Scaffold today has **no** fallible product API — add patterns above with the
first parse/translate surface; do not grow demo `simple.rs` into an
error framework without real fallible behavior. User persist errors → Flutter.

## Related Skills

- Dart presentation / `AnyhowException` mapping → `client-work-with-errors`
- Codegen / init / generated paths → `work-with-frb`
- Whether meta error-contract claims are buildable → `align-implementability-back`
- App canon → `{app}/AGENTS.md` via `docs/projects-map.md`

## Verification

When touching Rust errors, confirm:

- [ ] Expected user/domain failures return `Err` (or typed miss), not panic
- [ ] FRB `api/` functions expose `Result` (or typed error) for fallible work
- [ ] Stable code/prefix present for Flutter mapping
- [ ] Empty successful collections use `Ok`, not `Err`, when product wants empty UI
- [ ] `Cargo.toml` lists any new `anyhow`/`thiserror` dependency used
- [ ] Codegen run after API error-signature changes; generated files not hand-edited
- [ ] Unit tests cover at least one reject path per new fallible API
- [ ] User-facing polish left to `client-work-with-errors`, not duplicated as only channel
