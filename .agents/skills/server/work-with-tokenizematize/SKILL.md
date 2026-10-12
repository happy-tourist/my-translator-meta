---
name: work-with-tokenizematize
description: >-
  Guides Rust tokenization + lemmatization for Translation Reader: panko
  tokenize_cues, char spans, EN WordNet morph keys (`add-reader-lemma-highlight`),
  mark_vocab_tokens / lemma_keys_for_surfaces. Use when adding or changing
  tokenize / lemmatize in `{app}/rust`, FFI token DTOs for Flutter
  tap/highlight, or cue text → tokens → lemma → dictionary. Not for Flutter
  paint alone (work-with-reader) or FRB codegen mechanics (work-with-frb-api).
---

# Work With Tokenize + Lemmatize

**my_translator_app** Rust NLP for **Translation Reader**. Runtime:
`{app}/rust` (`docs/projects-map.md` key `app`). Read `{app}/AGENTS.md` first.

**Status:**

| Stage | State |
|-------|--------|
| Tokenize (`panko` 0.2.0, char spans, batch `tokenize_cues`) | **Shipped** — [tokenize.md](tokenize.md) |
| Morph lemmatize / `lemma_key` / `is_in_vocab` | **Shipped** — [lemmatize.md](lemmatize.md) (`wordnet-lemmatizer`, mark / lemma-keys FRB) |

**Core principle:** Rust owns NLP split and morph keys. Flutter paints spans
and gestures from Rust token data. **Do not** re-tokenize or stem in Dart.
Dictionary identity = morph `lemma_key` (call site → Drift); never embed
WordNet in Flutter.

## When to Use

- Cue display string → word tokens with char spans
- FRB `tokenize_cues` / `TokenDto` changes
- Morph lemma keys / mark helpers for highlight / dict match
- EN WordNet + casefold-fallback fixtures and span invariants

**Not this skill:** subtitle parse (→ **work-with-parse**); dictionary CRUD
persist (→ **work-with-dictionary**); FRB codegen (→ **work-with-frb-api**);
Flutter selection/CTA/sheet (→ **work-with-reader**); test harness shape alone
(→ **server-work-with-test**).

## Ownership (current)

| Concern | Layer |
|---------|--------|
| Split words, punctuation policy, char spans | Rust `tokenize/` |
| Batch FRB export | Rust `api/tokenize.rs` |
| Hit-test / selection chrome | Flutter (`work-with-reader`) |
| Dict key on save | Morph `lemma_key` from Rust (call site → Drift) |
| Morph lemma / highlight membership | Rust — [lemmatize.md](lemmatize.md) (`mark_vocab_tokens`) |

Anti-pattern: Dart `RegExp` word split for product tap; inventing alternate
lemma field names outside the locked OpenSpec DTO.

## Specialized topics

| Topic | File |
|-------|------|
| Shipped tokenize + FRB + tests | [tokenize.md](tokenize.md) |
| Lemmatize / WordNet / mark helpers | [lemmatize.md](lemmatize.md) |

## Contracts With Neighbors

| Neighbor | Boundary |
|----------|----------|
| **work-with-parse** | Supplies cue text; tokenize does not parse files |
| **work-with-dictionary** | Persist by morph `lemma` (+ `surface_key`); membership via FRB marks |
| **work-with-frb-api** | Thin `api/` + codegen after changes |
| **server-work-with-test** | `cargo test` on `tokenize/` + `lemmatize/` |
| **work-with-reader** | Consumes tokens for paint/tap/highlight; no Dart NLP |

## Checklist

- [ ] Input = same `text_lines.join('\n')` as Flutter paint
- [ ] Batch `tokenize_cues`; word-only DTOs; char span ↔ surface invariant
- [ ] `panko` pin behind `tokenize/` wrap; `api/` thin; no hand-edit generated
- [ ] Morph: `rust/src/lemmatize/` + `wordnet-lemmatizer`; DTO `lemma_key` / `is_in_vocab`; FRB `mark_vocab_tokens` + `lemma_keys_for_surfaces`
- [ ] Linked: parse, dictionary, frb-api, reader, server-work-with-test
- [ ] Not extending scaffold `process_text` into NLP

## Related Skills (by name)

- `work-with-parse`
- `work-with-dictionary`
- `work-with-frb-api`
- `server-work-with-test`
- `work-with-reader` (Flutter consumer; tap.md / highlight.md)
