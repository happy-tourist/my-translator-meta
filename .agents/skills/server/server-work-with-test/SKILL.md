---
name: server-work-with-test
description: >-
  Use when planning or writing Rust tests for my_translator_app (Translation
  Reader) core under `rust/`: unit tests via `#[cfg(test)]` next to parse /
  lemma / dict / translate modules, optional crate integration under `tests/`,
  thin FFI surface checks in `api/`, fixtures for subtitle snippets. Harness:
  `cargo test` from `{app}/rust`. Prefer internal-module coverage without
  Flutter. Flutter widget/integration tests are NOT this skill — use
  client-work-with-test. Core in SKILL.md; details in topics: unit.md,
  fixtures.md.
---

# Work With Test (Rust)

Use this skill to plan and write **Rust** tests for **my_translator_app**
(product: **Translation Reader**). Start with a test plan, then implement
tests from that plan.

**Paths:** this skill lives in **my_translator_meta** at
`.agents/skills/server/server-work-with-test/`. Runtime paths (`rust/src/…`,
`rust/tests/…`) are relative to the **my_translator_app** sibling
(`../my_translator_app` from meta; see `docs/projects-map.md`). Read
`{app}/AGENTS.md` before changing the crate.

Stack: **Rust** edition `2021`, crate `rust_lib_my_translator_app`
(`cdylib` + `staticlib`), **`cargo test`** from `{app}/rust`. Bridge:
**flutter_rust_bridge** `2.13.0` — public FFI in `rust/src/api/`; generated
`rust/src/frb_generated.rs` is never hand-edited. Rust tests use
`cargo test` only.

Flutter UI tests (`flutter test`, `integration_test/`) belong to
**client-work-with-test** — do **not** mix stacks here.

Core principle: test **core algorithms and contracts** (parse cues, lemmas,
dictionary rules, offline translate, SQLite when present) at the **internal
module** boundary. Keep `api/` tests thin (wiring / arg mapping). Do not
reimplement rules in Dart “for realism,” and do not require Flutter to prove
Rust correctness.

## Specialized Topics

Read the matching file in this folder when the unit under test involves that
area (do not load every file at once):

| Topic | File |
|-------|------|
| Unit modules, `#[cfg(test)]`, layout as core grows | [unit.md](unit.md) |
| Subtitle / lemma / dict fixtures and golden strings | [fixtures.md](fixtures.md) |

## Bootstrap (scaffold today)

Today `{app}/rust` is mostly FRB scaffold (`api/simple.rs`: `process_text`,
`add_numbers`). **Few or no product tests yet** — prescribe patterns below so
new modules ship with tests in the same change.

Harness:

| Piece | Role |
|-------|------|
| Command | From `{app}/rust`: `cargo test` (optionally `cargo test <filter>`) |
| Unit | `#[cfg(test)] mod tests { … }` in the same `.rs` as the SUT (preferred) |
| Integration | Optional `rust/tests/*.rs` for multi-module / public-API flows |
| Assert | `assert_eq!` / `assert!` / `assert!(matches!(…))` — std Rust asserts |

Do **not** put Rust logic tests under Flutter `test/` or `integration_test/`.

Agent **runs** `cargo test` from `{app}/rust` after writing or changing Rust
tests; fix failures before claiming done. Flutter analyze/test is out of scope
unless the same change also touches Dart (then follow client skills separately).

OpenSpec: new parse / lemma / dict / translate / persistence scenarios **must**
include `cargo test` task items mapped to `SC-…` IDs in Traceability — not
`cargo check` alone.

## What Belongs Where

| Concern | Test layer |
|---------|------------|
| SRT / VTT / ASS parse, cue ordering, edge formats | Rust unit → [unit.md](unit.md), fixtures → [fixtures.md](fixtures.md) |
| Tokenize / lemmatize / wordform match | Rust unit on internal modules |
| Dictionary CRUD, lemma lookup, context attach | Rust unit (+ SQLite temp DB when persistence lands) |
| Offline translate rules | Rust unit |
| Thin `api/` export of already-tested core | Optional thin `#[cfg(test)]` or `tests/` smoke |
| Button / loading / FRB fake UI | **client-work-with-test** |
| `RustLib.init` + UI smoke | **client-work-with-test** (`integration_test/`) |

## Target Module Layout (grow into this)

As product core lands (per `{app}/AGENTS.md`), expect modules **outside**
`api/` with thin FFI export:

```text
my_translator_app/rust/
  Cargo.toml
  src/
    lib.rs              # api + (later) parse / lemma / dict / …
    api/                # public FRB surface only
    parse/              # SRT/VTT/ASS — unit tests here
    lemma/              # tokenize + lemmatize — unit tests here
    dict/               # vocab rules / SQLite — unit tests here
    translate/          # offline translate — unit tests here
    frb_generated.rs    # GENERATED — never edit
  tests/                # optional crate-level integration
  fixtures/             # optional shared .srt / .vtt snippets (or under tests/)
```

Until those folders exist, add the module + its `#[cfg(test)]` together. Prefer
**private** modules with `pub(crate)` surfaces tested from the same crate over
testing only through FRB.

## Workflow

1. Read the SUT (`rust/src/…`): pure functions, error enums, public vs
   `pub(crate)` API. Note whether logic lives in `api/` (thin) or a core
   module (preferred home for tests).
2. Produce a test plan with two sections: **What needs to be faked/stubbed**
   and **What to verify**.
3. Place tests:
   - Unit → `#[cfg(test)]` in the same file / `mod tests` sibling →
     [unit.md](unit.md).
   - Multi-module or fixture-heavy → `rust/tests/<area>_test.rs` + fixtures →
     [fixtures.md](fixtures.md).
4. Cover success, error/reject, and representative edge cases from the plan.
5. Run `cargo test` from `{app}/rust`; fix failures before claiming done.

## Test Plan

### What Needs To Be Faked / Stubbed

Prefer **real** pure functions with in-memory or temp inputs. Stub only when
the SUT would hit something you must not run:

- **Filesystem / SQLite** — use temp dirs (`tempfile` if adopted) or in-memory
  DB; never developer production DB paths.
- **Optional network / AI providers** (later) — stub at a trait / function
  boundary; do not call real providers in unit tests.
- **Clocks** — only if the SUT uses time you must control; document the hook.

Do **not** mock FRB codegen. Do **not** require Flutter/`RustLib.init` for
core coverage. Do **not** hand-edit `frb_generated.rs` or Dart
`lib/src/rust/**` to make Rust tests pass.

### What To Verify

Use these categories only when the SUT has relevant behavior:

- Parse: cue count, text, timestamps, ordering; reject malformed input with
  stable error kind/message shape.
- Lemma / tokenize: surface form → lemma; same-lemma match across wordforms
  (`run` / `running` / `ran`); language pair when applicable.
- Dictionary: add / list / remove / lookup-by-lemma; context string retained;
  duplicate lemma policy as designed.
- Translate (offline): expected gloss for fixture phrases; error on empty /
  unsupported.
- API thin layer: `process_text` / future exports map args and return shapes
  without re-testing deep algorithms already covered below.

Formulation rules:

- Name the condition under test (`parse_srt_rejects_missing_arrow`, …).
- Assert outputs and error kinds, not private helper call graphs.
- Cover success **and** failure when the SUT branches.
- Use tables / loops for isomorphic cases (several formats, several lemmas).

Examples:

```text
Verify SRT parse via each fixture:
- Minimal two-cue file: cue count 2; texts and start/end ms match
- Missing timestamp arrow: Err(ParseError::…)
- UTF-8 BOM / CRLF: still parses

Verify lemma match via each wordform of "run":
- running, ran, runs → lemma "run"; highlight key equals dict entry

Verify dict add then lookup:
- add lemma+translation+context → list contains entry; lookup by lemma hits
- remove → lookup misses

Verify api::process_text (scaffold thin):
- "hello" → "HELLO" (until replaced by real pipeline)
```

## Do Not Test Through Flutter

- Do not write `flutter test` here to prove parse/lemma/dict.
- Do not assert UI Keys, SnackBars, or widget trees.
- Cross-stack smoke (`RustLib.init` + button) → **client-work-with-test**.

## Test File Structure

Preferred unit style (grow modules this way):

```rust
// rust/src/parse/srt.rs (illustrative — add when implementing parse)

pub fn parse_srt(input: &str) -> Result<Vec<Cue>, ParseError> {
    // …
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_two_cues() {
        let cues = parse_srt(include_str!("../../fixtures/minimal.srt")).unwrap();
        assert_eq!(cues.len(), 2);
        assert_eq!(cues[0].text, "Hello");
    }

    #[test]
    fn rejects_missing_arrow() {
        let err = parse_srt("1\n00:00:01,000 00:00:02,000\nHi\n").unwrap_err();
        assert!(matches!(err, ParseError::BadTimestamp { .. }));
    }
}
```

Scaffold API thin smoke (optional until core exists):

```rust
// rust/src/api/simple.rs — extend existing module

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn process_text_uppercases() {
        assert_eq!(process_text("hello".into()), "HELLO");
    }

    #[test]
    fn add_numbers_sums() {
        assert_eq!(add_numbers(2, 3), 5);
    }
}
```

Optional crate integration (`rust/tests/parse_roundtrip.rs`) when several
modules must compose — keep rare; prefer unit modules → [unit.md](unit.md).

## Checklist Before Finishing

- Test plan covered; success + failure for branching SUT paths.
- Tests live under `{app}/rust` (`#[cfg(test)]` and/or `tests/`); not Flutter
  `test/`.
- Core logic tested on internal modules; `api/` only thin if needed.
- Fixtures small and deterministic → [fixtures.md](fixtures.md).
- No hand-edits to `frb_generated.rs` or `lib/src/rust/**`.
- Flutter widget/integration deferred to **client-work-with-test**.
- Ran `cargo test` from `{app}/rust`; failures fixed.

## Anti-Patterns

```rust
// Testing Rust rules only via Flutter widget tests
// → use client-work-with-test for UI; keep cargo test for core

// Hand-editing generated FRB for test hooks
// rust/src/frb_generated.rs  ← FORBIDDEN

// Reimplementing parser in the test instead of calling SUT
#[test]
fn fake_parse() {
    let cues = input.lines().count(); // not the real parser
    assert_eq!(cues, 3);
}

// Giant binary fixtures committed without need
// fixtures/feature_film.srt  // prefer minimal snippets

// Skipping cargo test after adding tests
```

Topic-specific anti-patterns live in [unit.md](unit.md) and
[fixtures.md](fixtures.md).
