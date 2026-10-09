# Unit Tests (`#[cfg(test)]`)

Use with the [core test skill](SKILL.md) for Rust unit tests inside
`{app}/rust`. Prefer these for parse, lemma, dict, translate, and other
**internal** modules. Run with `cargo test` from `{app}/rust`.

Flutter widget/unit tests are **not** covered here →
`client-work-with-test`.

## When to use

| Use unit (`#[cfg(test)]`) | Prefer crate `tests/` instead | Prefer Flutter instead |
|---------------------------|-------------------------------|------------------------|
| Pure functions, error enums, lemma rules | Multi-module public composition | UI Keys, loading banners, FRB fakes |
| Same-file access to `pub(crate)` / private helpers under test | Black-box only through crate root | `RustLib.init` smoke |
| Fast TDD while growing a module | Shared large fixtures across crates | End-to-end reader UX |

Default for a growing core: **colocate** tests with the module. Add
`rust/tests/` only when the scenario needs several public modules or shared
binary fixtures without pulling private items.

## File layout

```text
my_translator_app/rust/src/
  parse/
    mod.rs
    srt.rs          # + #[cfg(test)] mod tests
    vtt.rs
  lemma/
    mod.rs          # + #[cfg(test)]
  dict/
    mod.rs          # + #[cfg(test)]
  api/
    simple.rs       # thin smoke only
  lib.rs            # pub mod api; mod parse; …

my_translator_app/rust/tests/   # optional
  parse_srt_integration.rs
```

Naming: `#[test] fn verb_condition_expected()` — stable, searchable filters
for `cargo test verb_condition`.

## Module pattern

```rust
//! Subtitle SRT parser (illustrative).

#[derive(Debug, PartialEq, Eq)]
pub struct Cue {
    pub index: u32,
    pub start_ms: u64,
    pub end_ms: u64,
    pub text: String,
}

#[derive(Debug, PartialEq, Eq)]
pub enum ParseError {
    Empty,
    BadTimestamp { line: usize },
}

pub fn parse_srt(input: &str) -> Result<Vec<Cue>, ParseError> {
    if input.trim().is_empty() {
        return Err(ParseError::Empty);
    }
    // …
    Ok(Vec::new())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn empty_input_is_error() {
        assert_eq!(parse_srt("").unwrap_err(), ParseError::Empty);
    }

    #[test]
    fn parses_single_cue_text() {
        let input = "1\n00:00:01,000 --> 00:00:02,000\nHello\n";
        let cues = parse_srt(input).expect("parse");
        assert_eq!(cues.len(), 1);
        assert_eq!(cues[0].text, "Hello");
    }
}
```

Conventions:

- `use super::*;` inside `mod tests`.
- Keep production code free of test-only deps; put `dev-dependencies` in
  `Cargo.toml` only when needed (`tempfile`, etc.).
- Prefer `assert_eq!` on owned values; use `matches!` for error variants with
  payloads you do not need to fully construct.
- Table-driven cases for isomorphic inputs:

```rust
#[test]
fn lemmas_normalize_wordforms() {
    let cases = [("running", "run"), ("ran", "run"), ("runs", "run")];
    for (surface, lemma) in cases {
        assert_eq!(lemmatize(surface, Lang::En), lemma);
    }
}
```

## Visibility

| Need | Approach |
|------|----------|
| Test public API | Call `pub` / `pub(crate)` from `#[cfg(test)]` |
| Test private helper | Move helper to `pub(crate)` in a submodule, or test via public wrapper |
| Force black-box | `rust/tests/*.rs` imports only crate root |

Do not `include!` production sources into tests. Do not weaken `pub` solely
for Dart/FRB — keep FFI surface in `api/`.

## Persistence (when SQLite lands)

- Use in-memory or temp-file DB per test; clean up in `Drop` or `defer`-style
  helper.
- Do not point tests at a developer device path.
- Cover migrate / CRUD / lookup-by-lemma; leave UI list rendering to Flutter
  tests.

## Thin `api/` tests

`api/` should stay a thin FRB export. A few smoke tests are OK for scaffold
(`process_text`, `add_numbers`) and later for “DTO in → DTO out” mapping.
Deep algorithm coverage belongs in `parse` / `lemma` / `dict`, not duplicated
in `api`.

Never edit `frb_generated.rs` for tests. After changing `api/` signatures, run
FRB codegen for Dart separately — Rust `cargo test` does not replace that.

## Running

```bash
cd ../my_translator_app/rust   # from meta; or projects-map `app` + /rust
cargo test
cargo test parse_              # filter by name
cargo test -- --nocapture     # show println! when debugging
```

Agent runs these after writing or changing tests; fix failures before done.

## Checklist

- Tests colocated with SUT unless integration file is justified.
- Success + error paths for branching logic.
- No Flutter harness; no generated-file edits.
- `cargo test` green from `{app}/rust`.

## Anti-Patterns

```rust
// Private field peeking via unsafe / mem tricks
// → expose pub(crate) test seam or assert via public result

// Copy-paste of production parser into the test module
fn parse_srt_for_test(s: &str) -> Vec<Cue> { /* duplicate */ }

// Only testing through FRB-generated Dart
// → core stays in cargo test; UI in client-work-with-test

// Ignoring Result in tests
let _ = parse_srt(bad); // should assert Err
```
