---
name: work-with-parse
description: >-
  SRT / VTT / ASS(SSA) parsing, cue models, timestamps, encoding, and parse
  errors in my_translator_app Rust core. Use when implementing or changing
  subtitle parsers, format detection, FFI-friendly cue output, or thin FRB
  export of parse results. Not for Flutter cue painting (work-with-reader) or
  tokenization / lemmatization (work-with-tokenizematize).
---

# Work With Parse

## Overview

**Subtitle parse** for **Translation Reader** (`my_translator_app`) is
**landed** under `rust/src/parse/` with thin FRB export `rust/src/api/parse.rs`
(`parse_subtitles` → `ParsedSubtitles` / `ParseErrorKind`). Scaffold demo
(`process_text` / `add_numbers`) may still exist — do **not** grow it into the
parser.

Logic lives in the **Rust internal `parse/` module**; Flutter only displays
cues and orchestrates the FRB call. Path package for UI:
`package:my_translator_app/...`. Canon: `{app}/AGENTS.md`.

**Paths:** this skill lives in **my_translator_meta** at
`.agents/skills/server/work-with-parse/`. Runtime paths (`rust/src/…`) are
relative to the **my_translator_app** sibling (`../my_translator_app` from meta;
see `docs/projects-map.md`).

## When to Use

- Add or change SRT / WebVTT / ASS(SSA) parsers
- Define or evolve the cue / document output model (index, start/end, text)
- Timestamps, encoding (UTF-8 / BOM / CRLF), format sniffing
- Parse error kinds surfaced through FRB
- Thin `rust/src/api/` export that returns cues to Flutter

**Not this skill:** reader line paint / tap / highlight (`work-with-reader`);
tokenize + lemmatize (`work-with-tokenizematize`); FRB codegen mechanics /
Dart call-site wiring alone (`work-with-frb-api`, client `work-with-frb`);
Flutter upload chrome alone (`work-with-pages`).

## Rust vs Flutter

| Concern | Owner |
|---------|--------|
| Detect format; parse bytes/text → cues | Rust (`rust/src/parse/`) |
| Timestamps, encoding normalize | Rust |
| Stable parse error kinds | Rust → thin `api/` |
| Display cue list / empty / error UI | Flutter |
| Tokenize / lemma / highlight flags | **Next** pipeline — `work-with-tokenizematize` |
| Persist document bytes / library rows | Flutter Drift (`library/documents`) — not this skill’s grammar |

Do **not** reimplement SRT/VTT/ASS grammar in Dart. Do **not** put parse logic
in `rust_builder/` or inflate scaffold `api/simple.rs` into a product parser.

## Placement (current tree)

Keep grammar / detect / map **outside** `api/`; FFI stays thin:

```text
my_translator_app/rust/src/
  lib.rs                 # `mod parse;` + `pub mod api;`
  parse/
    mod.rs               # Format enum, `parse_subtitles` / `parse_text`, re-exports
    cue.rs               # Cue / ParsedDocument (internal; FRB DTO is api::ParsedSubtitles)
    detect.rs            # extension hint + content sniff
    srt.rs / vtt.rs / ass.rs   # map crate parsers → Cue (lossy where pinned)
    error.rs             # ParseError kinds
    tests.rs + fixtures/ # unit fixtures (.srt / .vtt / .ass)
  api/
    parse.rs             # Thin FRB: bytes + format_hint → ParsedSubtitles / ParseErrorKind
    mod.rs               # `pub mod parse;`
```

Timestamps: normalize via the pinned crate’s duration → `u64` ms (no separate
`time.rs` required). Current pin: **`fasrt` 0.5** (see OpenSpec / Cargo.toml).

Wiring:

1. Change grammar / map / tests under `parse/`.
2. Export only what Flutter needs from `api/parse.rs` (DTO map).
3. Codegen / Dart call sites → **`work-with-frb-api`** (and client
   `work-with-frb` for UI binding).

## Output Model (FFI-friendly)

Keep the public shape **flat and implementable** — no heavy graphs, no
format-specific trees crossing the bridge unless a later change requires them.

**Target cue fields (minimum):**

| Field | Role |
|-------|------|
| `index` | 0-based order in document (stable for split-view sync) |
| `start_ms` / `end_ms` | Inclusive timeline bounds as integers (ms) |
| `text_lines` | `Vec<String>` — one entry per visual line inside the cue |

Optional later (do not invent until needed): raw format tag, speaker/style
name, position overrides. Prefer stripping ASS override codes / VTT tags to
**plain text lines** for reader v1 unless OpenSpec says otherwise.

**Document wrapper:** detected `format` (`srt` \| `vtt` \| `ass`) + ordered
`Vec<Cue>`. Reader open-flow (`parse/subtitles` PS-06): **zero cues → hard fail
`Empty`** (not ok-empty). Distinguish **empty** vs **malformed** via
`ParseError` / FFI `ParseErrorKind` (`server-work-with-errors`).

Timestamps: normalize all formats to the same integer ms (or a single shared
type). Do not expose format-specific timestamp strings on the FFI surface.

## Format Differences (high level)

| Format | Hallmarks | Parse notes |
|--------|-----------|-------------|
| **SRT** | Numeric index blocks; `HH:MM:SS,mmm --> HH:MM:SS,mmm`; blank-line separated | Comma decimal; ignore or tolerate missing index consistency; multi-line text until blank |
| **VTT** | Optional `WEBVTT` header; `HH:MM:SS.mmm --> …`; cue ids / notes / styles | Dot decimal; skip NOTE/STYLE/REGION blocks for v1 text cues; may omit hours |
| **ASS/SSA** | `[Script Info]` / `[Events]`; `Dialogue:` lines | Comma-separated fields after start/end; strip `{\…}` override codes for plain text; SSA ≈ older dialect of same family |

Detection: extension **hint** + content sniff (`WEBVTT`, `[Script Info]`, SRT-like blocks). Prefer content when they disagree; reject unknown with a stable error kind.

**Crates:** current product pin is **`fasrt` `=0.5.0`** (map layer in
`srt`/`vtt`/`ass`). Do not casually swap crates. If evaluating a replacement:
license, WASM/FFI friendliness, MSRV, and maintenance — prefer small surface
over a full media stack. Follow OpenSpec when it re-pins.

## Encoding & Input

- Prefer **UTF-8**; accept UTF-8 BOM; normalize `\r\n` / `\r` → `\n` before
  line logic.
- Input to core: `&str` / owned `String` after decode, or `&[u8]` with an
  explicit decode step at the boundary.
- Non-UTF-8: Reader open-flow uses **strict reject** (`str::from_utf8` →
  `Encoding`; not `from_utf8_lossy`). Do not silently corrupt text.
- Flutter passes **bytes** (+ optional `format_hint`); **decode + parse stay in
  Rust**.

## Errors

Classify failures for UI mapping (Library stays on the folder; no Reader
navigation on hard fail):

| Kind (illustrative) | When |
|---------------------|------|
| `UnsupportedFormat` | Unknown / ambiguous sniff |
| `Encoding` | Bytes not decodable under policy |
| `BadTimestamp` | Unparseable or `end < start` (map to `Malformed` on FFI) |
| `Malformed` | Truncated block, missing separator, broken ASS field count |
| `Empty` | Zero cues after a successful format path — **hard fail** for Reader open |

Use a typed `ParseError` (or equivalent) in `parse/`; map to FRB-safe messages
in `api/`. Full error-contract rules → **`server-work-with-errors`**. Flutter
display of those failures → `client-work-with-errors`.

## Pipeline Position

```text
file bytes/path
  → parse (this skill) → ordered cues
      → tokenize / lemmatize (work-with-tokenizematize)
          → reader paint / tap / vocab highlight (Flutter + later APIs)
```

Do not tokenize inside the parser. Cues are plain text lines + times; NLP is
the next step.

## Tests

- Unit-test parsers next to modules: `#[cfg(test)]` under `parse/`.
- Fixtures: minimal `.srt` / `.vtt` / `.ass` snippets (BOM, CRLF, multi-line,
  bad timestamp).
- Run from `{app}/rust`: `cargo test`.
- Patterns, layout, fixtures → **`server-work-with-test`**.
- Do **not** prove parse correctness only via `flutter test`.

## Related Skills (by name only)

- `server-work-with-test` — `cargo test`, fixtures, unit placement
- `server-work-with-errors` — Rust error kinds / FRB mapping
- `work-with-frb-api` — thin `api/` export + codegen contract
- `work-with-tokenizematize` — next pipeline step after cues
- `work-with-reader` — Flutter cue display (not parse)
- `work-with-frb` / `client-work-with-structure` — Dart binding / upload handoff

Canon product scenarios: `{app}/AGENTS.md`.

## Checklist

- [ ] Logic under `rust/src/parse/` (or agreed structure map); not Dart /
      `rust_builder/`
- [ ] Thin `api/` export only; regenerate FRB after surface changes
- [ ] Cues: index + start/end ms + text lines; FFI-flat
- [ ] SRT + VTT + ASS/SSA covered or explicitly scoped in the change
- [ ] Encoding / CRLF / BOM policy tested
- [ ] Typed parse errors → `server-work-with-errors`
- [ ] `cargo test` coverage → `server-work-with-test`
- [ ] No tokenize/lemma inside parse; hand off to `work-with-tokenizematize`
- [ ] Not growing scaffold `process_text` into a subtitle parser
