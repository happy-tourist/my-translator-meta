# Vocab Highlight by Lemma

## Contract

- Dictionary entries key on **lemma**.
- Any token whose lemma is in the user dictionary is marked for highlight —
  including other word forms of the same lemma.
- **Rust** computes membership (and may batch per cue / document). **Flutter**
  only paints (background / underline / weight via theme — see
  `work-with-styles`).

## Data flow

```text
Rust: tokens[{ text, lemma_id, in_vocab, … }]
  → Flutter: TextSpan / rich line with highlight style when in_vocab
```

After vocab add/remove (from card or vocab screen), reload marks for visible
cues (or whole doc) from Rust. Do not flip `in_vocab` only in Dart memory
without persisting through Rust.

## Paint rules

- Highlight is per **token span**, not the whole cue.
- Non-vocab tokens stay default body style.
- Selection / tap focus chrome is separate from vocab highlight (both may show).
- Split view: apply the same lemma marks on the **original** pane; translation
  pane highlights only if Rust provides marks for that side (default: original
  only).

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| `RegExp` word list in Dart | `in_vocab` / lemma flags from Rust |
| Highlight by exact surface string only | Lemma equality |
| Hard-coded highlight Color in every widget | Shared style token (`work-with-styles`) |
| Stale highlights after dict change | Refresh from Rust |
