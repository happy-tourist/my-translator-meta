# Tap selection + add to vocabulary

Canon for capability `reader/tap`. Chrome = bottom CTA + modal bottom
sheet (Add). Morph lemma highlight + edit on in-vocab tokens:
[highlight.md](highlight.md) / OpenSpec `add-reader-lemma-highlight`.

## Preconditions

1. On Reader open, Flutter batch-tokenizes **original** cues once:
   `texts = originalCues.map((c) => c.textLines.join('\n'))` → injectable
   `tokenizeCues` (default FRB `tokenizeCues`). Then load morph lemmas from
   Drift → injectable `markVocabTokens` → cache marked cues. Rebuilds only
   read the cache.
2. Display string for paint **and** tokenize must match (`join('\n')`).
3. Char spans are Unicode scalar indices. Slice with `cueCharSlice` /
   `runes` (`cue_char_slice.dart`) — **not** Dart UTF-16 `substring`.
4. Translation pane: plain text only (no token Keys / selection / highlight).

## Selection state machine (original pane)

State: `activeCueIndex` + single selected `tokenIndex` (or inclusive range
with `start == end`). Empty → no Add CTA.

| Tap (tokens ready) | Result |
|--------------------|--------|
| Token with `is_in_vocab` | Clear Add selection; open **edit** sheet ([highlight.md](highlight.md)); **no** Add CTA |
| Selection empty → non-vocab `T` | Select `T`; show Add CTA |
| Other non-vocab in same / other cue | Replace selection with that single token; CTA stays |
| Same single non-vocab again | No-op; CTA stays |

- Gestures: **tap only** (no drag / long-press; **no** contiguous multi-token
  expand in this change — phrase Add later).
- Per-token hit targets (`GestureDetector` / span); Keys
  `readerCueToken_${cueIndex}_${tokenIndex}`.
- Do **not** put a competing cue-level `InkWell`/`GestureDetector` on the same
  tap.

## Chrome: CTA + Add bottom sheet

1. Non-empty **non-vocab** selection → `Scaffold.bottomNavigationBar` with
   `Key('readerAddVocabCta')`, label **Add to vocabulary**. Empty / in-vocab
   edit path → `null`.
2. CTA → `showModalBottomSheet` (`Key('readerAddVocabSheet')`):
   - surface read-only (`readerAddVocabSurface`) — one token
   - translation editable, initially empty, required (`readerAddVocabTranslation`)
   - context editable, prefill = full cue text (`readerAddVocabContext`)
   - Save (`readerAddVocabSave`); validation error (`readerAddVocabValidationError`)
3. Persist via injected `DictionaryRepository.add` with
   `lemma: token.lemmaKey`, `surfaceKey: normalizeSurface(token.surface)`
   (same upsert as Vocab UI). Empty translation after trim → no persist, error.
4. Success → close sheet, clear selection, **refresh** marks via
   `markVocabTokens` ([highlight.md](highlight.md)).
5. Tokenize / mark FRB failure → plain text, taps off, SnackBar
   `readerTokenizeErrorSnack` (no auto-retry).

## DI / test seams

- `AppShell` → `LibraryScreen(dictionaryRepository:)` →
  `ReaderScreen(dictionaryRepository:)` (required).
- Optional `ReaderScreen.tokenizeCues` / `TokenizeCuesFn`,
  `markVocabTokens` / `MarkVocabTokensFn`, and shared
  `LemmaKeysForSurfacesFn` (Vocab + migration) — fake compute + real/test Drift.

## Later (not this change)

| Later | Notes |
|-------|--------|
| Word / phrase card with offline gloss | FRB translate + richer chrome |
| Contiguous / drag / long-press phrase Add | Product may add with phrase highlight |
| Tap on translation pane | Out of scope until a change says so |

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Instant word card on every tap | Select → CTA → sheet (non-vocab); edit sheet (in-vocab) |
| Contiguous multi-token Expand as current canon | Single-token Add only |
| Dart `RegExp` / UTF-16 `substring` for tokens | Rust spans + `cueCharSlice` |
| Per-row FRB tokenize on list rebuild | Batch once on open, cache |
| Invent local translation in the sheet | Manual translation field → Drift |
| Dict writes outside `DictionaryRepository` | Same persist path as Vocab |
| Stub `normalizeLemma(surface)` as morph identity | `token.lemmaKey` from Rust |
