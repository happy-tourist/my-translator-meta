# Vocab Highlight by Lemma

## Contract

- Dictionary entries key on **morph lemma** (`lemma` / `lemma_key`).
- Any original-pane **word** token whose `lemma_key` is in the user dictionary
  is marked for highlight — including other word forms of the same lemma.
- **Rust** computes `lemma_key` + `is_in_vocab` (batch). **Flutter** only paints
  (theme vocab highlight — see `work-with-styles`).
- OpenSpec change: `add-reader-lemma-highlight` (`reader/highlight`) — shipped.
  Phrase-span highlight and translation-pane highlight are out of that change.

## Data flow

```text
Rust tokenize_cues → tokens[{ surface, spans, lemma_key, is_in_vocab=false }]
  → Flutter: Drift morph lemmas → mark_vocab_tokens → is_in_vocab
  → Flutter: TextSpan / rich line when is_in_vocab (theme token)
```

After Reader Add/edit save or delete, reload marks on the live cache via
`mark_vocab_tokens`. Vocab-tab delete while Reader is not mounted → next
Reader open (tokenize→mark). Do not flip `is_in_vocab` only in Dart without
the FRB mark path.

Tap on `is_in_vocab` token → edit sheet (no Add CTA); see change
`reader/highlight` / `reader/tap` deltas.

## Paint rules

- Highlight is per **token span**, not the whole cue.
- Color from `Theme.of(context).extension<ReaderColors>()!.lemmaHighlight`
  (light+dark) — not inline `Colors.*` in the row.
- Non-vocab tokens stay default body style.
- Selection / tap focus chrome is separate from vocab highlight (both may show).
- Split view: vocab highlight on the **original** pane only in this change;
  translation pane MUST NOT get vocab highlight UX.
- After Reader Add/edit save or delete: re-run `markVocabTokens` on the cache.
  Vocab-tab deletes while Reader is not mounted are covered on next Reader open.

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| `RegExp` word list in Dart | `is_in_vocab` / `lemma_key` from Rust |
| Highlight by exact surface string only | Morph lemma equality |
| Hard-coded highlight Color in every widget | Shared style token (`work-with-styles`) |
| Stale highlights after dict change | Refresh via `mark_vocab_tokens` |
| Dart-only Set membership as sole truth | FRB flags (RH-09) |
