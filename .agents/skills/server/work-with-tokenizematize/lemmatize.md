# Lemmatize + highlight keys

**Shipped** with OpenSpec `add-reader-lemma-highlight` (capabilities
`reader/highlight` + morph identity in `dictionary/entries`).

Legacy Flutter `normalizeLemma` stub keys are replaced by morph `lemma_key`.

## Pipeline

```text
parsed cue text
  → tokenize  → Token { surface, char_start/end, … }   # shipped
  → lemmatize → lemma_key (EN WordNet + casefold fallback)
  → mark      → is_in_vocab from Drift morph-lemma set
  → consumers: dictionary membership, Reader highlight paint
```

| Concern | Layer |
|---------|--------|
| Lemma / morph key | Rust `lemmatize/` + `wordnet-lemmatizer` |
| Same-lemma equality across wordforms | Rust `lemma_key` |
| Dictionary membership by lemma | Drift lemma set → FRB `mark_vocab_tokens` |
| Surface→lemma without tokenize | FRB `lemma_keys_for_surfaces` (Vocab / stub→morph migration) |
| Highlight style | Flutter (`work-with-styles` / `ReaderColors`) |

## Layout

```text
my_translator_app/rust/src/
  lemmatize/mod.rs     # OnceLock Lemmatizer::embedded; lemma_key_for_surface
  api/tokenize.rs      # TokenDto.lemma_key / is_in_vocab; mark_vocab_tokens;
                       # lemma_keys_for_surfaces (thin FRB over lemmatize/)
```

## Library (locked)

```toml
wordnet-lemmatizer = { version = "0.1", default-features = false }
```

- `Lemmatizer::embedded()` once via `OnceLock` (not per token).
- EN-only WordNet always; RU morph / settings language gate — out of scope.
- POS cascade **Noun → Verb → Adj**; first `lemmatize` result ≠ casefold(surface)
  wins; else casefold. Whitespace surfaces → casefold only (no WordNet).
- `tokenize_cues` sets `lemma_key` and leaves `is_in_vocab = false` until mark.
- Full algorithm: change `design.md` D1–D3 (archive after sync).

## Neighbor contracts

- Persist stays Drift (`work-with-dictionary`) — lemmatize does not own SQLite.
- After vocab add/remove, refresh marks via `mark_vocab_tokens`
  (`work-with-reader` / highlight.md).
- Do not invent a second morph algorithm in Dart.
