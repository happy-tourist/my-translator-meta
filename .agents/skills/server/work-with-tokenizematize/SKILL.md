---
name: work-with-tokenizematize
description: >-
  Guides Rust tokenization + lemmatization for Translation Reader: tokens,
  lemmas, char/byte spans, EN/RU (+extensible langs), morph-aware highlight
  keys for vocab paint and tap targets. Use when adding or changing tokenize /
  lemmatize / wordform-match in `{app}/rust`, designing FFI token DTOs for
  Flutter highlight/tap, or wiring cue text → tokens → lemma → dictionary
  lookup. Not for Flutter paint alone (work-with-reader) or FRB codegen
  mechanics (work-with-frb-api).
---

# Work With Tokenize + Lemmatize

Target design for **tokenization + lemmatization** in **my_translator_app**
(product: **Translation Reader**). Runtime: sibling `{app}/rust`
(`docs/projects-map.md` key `app`). Read `{app}/AGENTS.md` before changing the
crate.

**Status:** scaffold today — **not implemented**. `rust/Cargo.toml` has only
`flutter_rust_bridge`; no tokenize/lemma modules yet. Follow this skill when
introducing them; do not grow demo `process_text` into NLP.

**Core principle:** Rust owns NLP. Flutter paints spans and gestures from Rust
token data. **Do not** re-tokenize, stem, or lemmatize in Dart.

## When to Use

- Cue/line text → tokens with spans + lemmas
- Morph-aware vocab highlight keys (`run` / `running` / `ran`)
- Tap / phrase targets keyed by token spans
- FFI-friendly token DTOs exported via `rust/src/api/`
- EN / RU first; planning hooks for more languages

**Not this skill:** subtitle file parse (→ **work-with-parse**); dictionary CRUD
/ lookup persistence (→ **work-with-dictionary**); FRB codegen / generated Dart
(→ **work-with-frb-api**); Flutter `TextSpan` paint / gestures alone
(→ **work-with-reader**); Rust test harness shape alone
(→ **server-work-with-test**).

## Pipeline (target)

```text
parsed cue text (from parse)
  → tokenize  → Token { surface, char_start/end, byte_start/end?, … }
  → lemmatize → lemma / lemma_key (language-aware)
  → consumers:
        dictionary lookup / add-by-lemma
        Flutter highlight + tap hit-test (paint only)
```

| Stage | Owner | Output |
|-------|--------|--------|
| Parse SRT/VTT/ASS → cues | Rust parse | cue id + text (+ timing) |
| Tokenize cue text | Rust NLP | ordered tokens + spans in that cue |
| Lemmatize each token | Rust NLP | `lemma_key` (normalize for dict match) |
| Mark in-vocab | Rust dict ∩ lemmas | flag / set on tokens (or join at API) |
| Paint / hit-test | Flutter | `TextSpan` / geometry from Rust spans |

Input is **already-parsed cue text**, not raw subtitle files. Do not re-parse
inside tokenize. Language for morph rules comes from reader/settings learning
pair — not UI chrome locale alone.

## Ownership

| Concern | Layer |
|---------|--------|
| Split words, punctuation policy, spans | Rust |
| Lemma / stem / morph key | Rust |
| Same-lemma equality across wordforms | Rust |
| Dictionary membership by lemma | Rust (+ **work-with-dictionary**) |
| Highlight style / theme | Flutter (**work-with-styles**) |
| Gesture → token under pointer | Flutter hit-test using **Rust spans** |

Anti-pattern: Dart `RegExp` word split for product tap/highlight; Dart stemmer;
keeping a divergent “highlighted lemmas” set only in Flutter memory.

## Languages

| Lang | Priority | Notes |
|------|----------|--------|
| EN | First | Wordforms → shared lemma key for highlight/dict |
| RU | First | Cyrillic morph; same span + lemma DTO shape |
| Other | Extensible | Pluggable per-lang backend behind one trait/API |

Unknown / unsupported language: degrade gracefully (e.g. surface-as-lemma or
tokenize-only) — document the chosen fallback in the module; do not crash the
reader. Flag product policy if unclear: **нужно решение разработчика**.

## Library Choice — Do Not Hard-Commit

`{app}/rust/Cargo.toml` currently has **no** stemmer/lemmatizer/NLP crate.
**Do not** pick and pin a crate in docs or code as “the” solution without an
explicit developer decision.

**«Нужно решение разработчика»** before adopting any of: Snowball/stemmers,
`rust-stemmers`, spaCy-like bindings, dictionary-based lemmatizers, custom
finite-state morph, embedding models, or large on-device morph DBs.

### Selection criteria (when deciding)

1. **Mobile + WASM** — size, no mandatory network, acceptable cold start.
2. **EN + RU quality** for lemma-key stability (highlight must not drift).
3. **Offline** default; no cloud morph API as a hard dependency.
4. **License** compatible with app distribution.
5. **Determinism** — same surface + lang → same `lemma_key` across platforms.
6. **FFI cost** — CPU per cue; batch cues if needed; avoid per-keystroke full-doc
   re-lemma unless measured OK.
7. **Extensibility** — trait / `match lang` so a second backend can land later.

Until a crate is chosen: implement a **placeholder** module with a stable
internal API and stub/heuristic lemmatizer (e.g. lowercase identity) **behind**
that API — or leave functions `todo!` / `unimplemented!` only in clearly marked
stubs, not in the public FRB contract that Flutter already calls.

## Placeholder Module Layout (target)

Prefer core **outside** `api/`; thin FRB export in `api/`:

```text
my_translator_app/rust/src/
  lib.rs
  api/                    # thin FRB: tokenize_cue / tokens_for_cues, …
  tokenize/               # or lemma/ — pick one name and stick to it
    mod.rs                # public(crate) pipeline entry
    token.rs              # Token, Span, LemmaKey types
    langs/
      mod.rs              # LangBackend trait + dispatch
      en.rs               # EN backend (crate TBD)
      ru.rs               # RU backend (crate TBD)
    normalize.rs          # optional: NFC, case fold policy for keys
  frb_generated.rs        # GENERATED — never edit
```

Align folder name with **server-work-with-test** target (`lemma/` there) or
rename both when implementing — one tree, not duplicate `tokenize/` + `lemma/`
pipelines. **нужно решение разработчика** for final module name if OpenSpec /
other skills already assume `lemma/`.

## FFI-Friendly Token DTO

Export **plain, owned, FRB-mappable** types from `rust/src/api/` (or re-export
from core). Prefer:

```rust
// Illustrative target shape — not in repo yet; adjust names to FRB idioms.

pub struct TextSpan {
    pub char_start: u32, // inclusive, Unicode scalar index in cue text
    pub char_end: u32,   // exclusive
    // Optional if Flutter/layout needs UTF-8 offsets:
    // pub byte_start: u32,
    // pub byte_end: u32,
}

pub struct TokenDto {
    pub surface: String,     // exact substring of cue text
    pub span: TextSpan,      // must slice cue[char_start..char_end] == surface
    pub lemma_key: String,   // morph-aware highlight / dict key
    pub lang: String,        // e.g. "en" / "ru"
    pub is_word: bool,       // false for punctuation / whitespace tokens if kept
    pub in_vocab: bool,      // filled when joined with dictionary; else false
}

pub struct CueTokensDto {
    pub cue_id: String,      // stable id from parse layer
    pub tokens: Vec<TokenDto>,
}
```

### DTO rules

- **Spans are authoritative** for tap hit-test and highlight paint.
- `surface` must equal the cue substring for `span` (invariant; test it).
- Prefer **char** indices for Flutter `Text`/`TextSpan`; add **byte** indices only
  if a consumer needs them — document which is canonical for hit-test.
- `lemma_key` is the morph-aware highlight key (same string dictionary uses).
- Avoid raw pointers, lifetimes, or non-`'static` borrows across FRB.
- Prefer `String` / `u32` / `bool` / `Vec` over clever enums until FRB mapping is
  proven; if enums are needed, keep them simple and codegen-friendly.
- Batch API (`tokens_for_document` / per visible cue range) beats N calls per
  line when scrolling — design for batch even if v1 is per-cue.
- Do **not** hand-edit `frb_generated.rs` or `lib/src/rust/**`; change `api/` →
  codegen (**work-with-frb-api**).

## Contracts With Neighbors

| Neighbor | Boundary |
|----------|----------|
| **work-with-parse** | Supplies cue id + text; tokenize does not parse files |
| **work-with-dictionary** | Lookup/add by `lemma_key`; tokenize does not own SQLite |
| **work-with-frb-api** | Thin `api/` functions + DTO types; codegen after changes |
| **server-work-with-test** | Unit-test tokenize/lemma on internal modules (`cargo test`) |
| **work-with-reader** | Flutter consumes tokens for paint/tap; no Dart NLP |

After vocab add/remove, refresh `in_vocab` (or equivalent marks) from Rust — see
client `work-with-reader` highlight/tap topics.

## Edge Cases

| Case | Contract |
|------|----------|
| Empty cue / whitespace-only | Empty `tokens` or non-word tokens only; no fake words |
| Punctuation | Either separate non-word tokens or stripped — be consistent; taps ignore non-words |
| Apostrophes / hyphens (EN) | Policy explicit (split vs keep); test fixtures |
| Soft hyphens / RTL later | Out of scope until langs expand; don't break UTF-8 spans |
| Unsupported lang | Fallback documented; stable keys |
| Tokenize pending | API/UI may show plain text until tokens ready (reader skill) |

## Testing (pointers)

- Unit: surface → `lemma_key`; same lemma across wordforms; span invariants.
- Fixtures: small EN/RU strings, not full films.
- Harness: `cargo test` in `{app}/rust` — details in **server-work-with-test**.
- Do not prove lemma rules only via Flutter widget tests.

## Checklist

- [ ] Pipeline: cue text → tokens+spans → lemma_key → dict + Flutter
- [ ] NLP only in Rust; Flutter paints Rust spans
- [ ] EN/RU path designed; other langs behind dispatch
- [ ] No hard-pinned lemma crate without developer decision (**нужно решение разработчика**)
- [ ] Placeholder module layout; `api/` thin; DTO FFI-safe
- [ ] Span ↔ surface invariant covered by tests when implementing
- [ ] Linked: parse, dictionary, frb-api, server-work-with-test
- [ ] Not extending scaffold `process_text` uppercase demo into NLP

## Related Skills (by name)

- `work-with-parse`
- `work-with-dictionary`
- `work-with-frb-api`
- `server-work-with-test`
- `work-with-reader` (Flutter consumer; highlight.md / tap.md)
