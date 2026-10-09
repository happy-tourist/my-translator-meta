# Fixtures

Use with the [core test skill](SKILL.md) when Rust tests need deterministic
subtitle snippets, lemma tables, or dictionary seed rows. Keep fixtures
**small**, **UTF-8**, and **checked in** next to the crate.

## When to use

| Use fixtures | Inline the string instead |
|--------------|---------------------------|
| Multi-line SRT/VTT/ASS samples reused across tests | One-liners / single cue |
| Encoding edge cases (BOM, CRLF, non-ASCII) | Trivial `"hello"` API smoke |
| Shared golden expected JSON/text | One-off assert in a single test |

## Layout

Prefer one of:

```text
my_translator_app/rust/fixtures/
  srt/minimal.srt
  srt/missing_arrow.srt
  vtt/minimal.vtt
  lemma/en_run_forms.txt

my_translator_app/rust/src/parse/fixtures/   # colocated with parse module
  minimal.srt
```

Or embed tiny samples with `include_str!` from the test module:

```rust
#[test]
fn parses_minimal_srt() {
    let input = include_str!("../../fixtures/srt/minimal.srt");
    let cues = parse_srt(input).unwrap();
    assert_eq!(cues.len(), 2);
}
```

Paths are relative to the **source file** that contains `include_str!`, not
the crate root. Adjust `../` carefully when moving files.

## Minimal SRT example

```text
1
00:00:01,000 --> 00:00:02,500
Hello

2
00:00:03,000 --> 00:00:04,000
World
```

Save as `fixtures/srt/minimal.srt`. Pair with expected structs in the test —
do not rely on floating “approx” timestamps unless the SUT defines rounding.

## What to cover with fixtures

- Happy: minimal valid SRT / VTT / ASS.
- Reject: missing `-->`, empty body, truncated file (assert error kind).
- Normalize: CRLF vs LF, leading UTF-8 BOM, extra blank lines between cues.
- Multilingual: at least one non-ASCII cue (e.g. Cyrillic) when RU path exists.
- Lemma: small table of surface → lemma pairs for EN (and RU when implemented).

## Determinism rules

1. No network fetches for fixtures.
2. No timestamps from “now”; fixed cue times only.
3. No full movie/episode files — trim to the smallest reproducing case.
4. Prefer text fixtures over binary; if binary is required, document why.
5. Golden expected output: either assert field-by-field in Rust or a small
   `.expected.json` next to the input — keep both in the same change.

## Dictionary / SQLite seeds

When persistence exists, seed via API in the test (insert rows) or a tiny SQL
fixture applied to a temp DB. Do not commit a large live DB file for
unit tests.

```rust
#[test]
fn lookup_by_lemma_after_seed() {
    let db = open_temp_dict(); // helper when dict lands
    db.add(entry("run", "бежать", "I run daily")).unwrap();
    assert!(db.lookup_lemma("run").is_some());
}
```

## Checklist

- Fixture is minimal and named by scenario (`minimal`, `bad_timestamp`, …).
- `include_str!` / path resolves from the test file location.
- Assertions check structured fields, not only “non-empty”.
- No Flutter/`assets/` coupling required for Rust unit tests.

## Anti-Patterns

```text
# Full-length subtitle dump as the only fixture
fixtures/entire_season.srt

# Fixture that requires Flutter asset bundle loading
# → Rust tests read files / include_str! only

# Undocumented encoding
# binary .srt with unknown codepage — use UTF-8
```
