# Lemmatize + highlight keys

**Shipped** with OpenSpec `add-reader-lemma-highlight` (capabilities
`reader/highlight` + morph identity in `dictionary/entries`).

Legacy Flutter `normalizeLemma` stub keys are replaced by morph `lemma_key`.

## Pipeline

```text
parsed cue text
  → tokenize  → Token { surface, char_start/end, … }   # shipped
  → lemmatize → lemma_key (EN WordNet + casefold fallback)
  → mark      → is_in_vocab from Drift morph + phrase keys
  → consumers: dictionary membership, Reader highlight paint
```

| Concern | Layer |
|---------|--------|
| Lemma / morph key | Rust `lemmatize/` + `wordnet-lemmatizer` |
| Same-lemma equality across wordforms | Rust `lemma_key` |
| Phrase identity | casefold + single-space join (no WordNet) |
| Dictionary membership | Drift lemma set → FRB `mark_vocab_tokens` (morph **or** phrase windows) |
| Surface→lemma without tokenize | FRB `lemma_keys_for_surfaces` (Vocab / stub→morph migration) |
| Highlight style | Flutter (`work-with-styles` / `ReaderColors`) |

## Layout

```text
my_translator_app/rust/src/
  lemmatize/mod.rs     # OnceLock Lemmatizer::embedded; lemma_key_for_surface;
                       # mark_vocab_flags (morph singles + phrase windows)
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
  wins; else casefold. Whitespace surfaces → casefold + collapse internal
  whitespace to single spaces (no WordNet).
- `tokenize_cues` sets `lemma_key` and leaves `is_in_vocab = false` until mark.
- `mark_vocab_flags` / FRB `mark_vocab_tokens`: singles via casefold
  `lemma_key`; phrases via sliding consecutive
  `casefold(join(surfaces, " "))` windows; overlaps OR to true; mark all
  tokens in each hit span (not cue-substring match).
- Full algorithm: change `design.md` D1–D3 / D2 phrase delta (archive after sync).

## Neighbor contracts

- Persist stays Drift (`work-with-dictionary`) — lemmatize does not own SQLite.
- After vocab add/remove, refresh marks via `mark_vocab_tokens`
  (`work-with-reader` / highlight.md).
- Do not invent a second morph algorithm in Dart.
