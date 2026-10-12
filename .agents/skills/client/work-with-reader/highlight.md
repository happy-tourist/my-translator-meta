# Vocab Highlight by Lemma / Phrase

## Contract

- Dictionary entries key on **morph lemma** for singles, or **phrase identity**
  `casefold(join(surfaces, " "))` for multi-word rows.
- Any original-pane token with `is_in_vocab` is painted — morph wordforms of the
  same lemma **and** every token in a matched consecutive phrase span.
- **Rust** computes `lemma_key` + `is_in_vocab` (batch mark: morph singles **OR**
  phrase windows). **Flutter** only paints (theme vocab highlight — see
  `work-with-styles`).
- OpenSpec change: `add-reader-lemma-highlight` (`reader/highlight`) — shipped,
  including phrase-span paint. Translation-pane highlight remains out of scope.

## Data flow

```text
Rust tokenize_cues → tokens[{ surface, spans, lemma_key, is_in_vocab=false }]
  → Flutter: Drift lemmas (morph + phrase) → mark_vocab_tokens → is_in_vocab
  → Flutter: TextSpan / rich line when is_in_vocab (theme token)
```

`mark_vocab_tokens` (via `lemmatize::mark_vocab_flags`):

1. Split vocab keys into **singles** (no whitespace) and **phrases** (whitespace).
2. Singles: casefold `lemma_key` membership.
3. Phrases: sliding consecutive windows; match
   `casefold(join(surfaces, " "))` (token-join, not cue substring); mark **all**
   tokens in the hit window. Overlaps **OR** to true.
4. Irregular whitespace in dict keys collapses to single spaces before match.

After Reader Add/edit save or delete, reload marks on the live cache via
`mark_vocab_tokens`. Vocab-tab delete while Reader is not mounted → next
Reader open (tokenize→mark). Do not flip `is_in_vocab` only in Dart without
the FRB mark path.

## Edit on in-vocab tap (D8)

Tap on `is_in_vocab` token → clear Add selection → edit sheet (no Add CTA).
When several dict keys hit the same token, resolve Drift entry by priority:
**longest phrase span (token count) > shorter phrase > single morph**.
Sheet upsert/delete uses that resolved key; paint stays OR of all hits.

## Paint rules

- Highlight is per **token span**, not the whole cue.
- Color from `Theme.of(context).extension<ReaderColors>()!.lemmaHighlight`
  (light+dark) — not inline `Colors.*` in the row.
- Non-vocab tokens stay default body style.
- Selection / gap-flash chrome is separate from vocab highlight (priority:
  flash > selection > vocab > body).
- Split view: vocab highlight on the **original** pane only;
  translation pane MUST NOT get vocab highlight UX.
- After Reader Add/edit save or delete: re-run `markVocabTokens` on the cache.
  Vocab-tab deletes while Reader is not mounted are covered on next Reader open.

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| `RegExp` word list in Dart | `is_in_vocab` / `lemma_key` from Rust |
| Highlight by exact surface string only | Morph lemma equality + phrase windows |
| Cue-substring phrase match | Consecutive token-join windows |
| Hard-coded highlight Color in every widget | Shared style token (`work-with-styles`) |
| Stale highlights after dict change | Refresh via `mark_vocab_tokens` |
| Dart-only Set membership as sole truth | FRB flags (RH-09) |
| Ambiguous overlap → random entry | Longest phrase > shorter > single (D8) |
