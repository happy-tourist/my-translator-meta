# Lemmatize + highlight keys (later)

**Not shipped.** MVP dict keys use Flutter `normalizeLemma` (trim + case-fold)
on the selected surface/phrase. Morph lemmatization is a future change.

## Target pipeline (when landing)

```text
parsed cue text
  → tokenize  → Token { surface, char_start/end, … }   # already shipped
  → lemmatize → lemma_key (language-aware)
  → consumers: dictionary membership, Reader highlight paint
```

| Concern | Layer |
|---------|--------|
| Lemma / stem / morph key | Rust |
| Same-lemma equality across wordforms | Rust |
| Dictionary membership by lemma | Drift (+ this key) |
| Highlight style | Flutter (`work-with-styles`) |

## Library choice

Do **not** hard-pin a morph/lemmatizer crate without an explicit developer
decision (**нужно решение разработчика**). Criteria when deciding: mobile +
WASM size, EN/RU quality, offline, license, determinism, FFI cost,
extensibility.

Until then: keep tokenize-only FRB; do not invent a fake `lemma_key` field on
the public tokenize DTO that Flutter already calls.

## Languages (target)

| Lang | Priority | Notes |
|------|----------|--------|
| EN | First | Wordforms → shared lemma key |
| RU | First | Cyrillic morph; same DTO shape |
| Other | Extensible | Pluggable backend behind one trait/API |

## Neighbor contracts

- Persist stays Drift (`work-with-dictionary`) — lemmatize does not own SQLite.
- After vocab add/remove, refresh `in_vocab` / marks for highlight
  (`work-with-reader` / highlight.md) — not part of tokenize MVP.
