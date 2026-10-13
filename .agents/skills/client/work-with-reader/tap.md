# Tap selection + add to vocabulary

Canon for capability `reader/tap`. Chrome = bottom CTA + modal bottom
sheet (Add). Morph + phrase highlight + edit on in-vocab tokens:
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

State: `activeCueIndex` + inclusive `[selectionStart, selectionEnd]` within
that cue’s tokens. Empty → no Add CTA. Gestures: **tap only** (no drag /
long-press). Selection never spans cues.

| Tap (tokens ready) | Result |
|--------------------|--------|
| Token with `is_in_vocab` | Clear Add selection; open **edit** sheet ([highlight.md](highlight.md) D8 resolve); **no** Add CTA |
| Selection empty → non-vocab `T` | Select `[T,T]`; show Add CTA |
| Adjacent outside range (same cue), non-vocab | Expand contiguous range; CTA stays |
| Inside multi-token range (`end - start ≥ 1`) | Shrink to `[T,T]`; CTA stays |
| Same single non-vocab again | Clear selection; hide CTA |
| Gap (non-adjacent, same cue), non-vocab | Reset to `[T,T]` + ~300ms gap-flash; CTA stays |
| Other cue, non-vocab | Move selection to that cue’s `[T,T]` (no gap-flash) |

- Per-token hit targets (`GestureDetector` / span); Keys
  `readerCueToken_${cueIndex}_${tokenIndex}` (list pane). Under video timeline
  rows, `CueRow.keySuffix` appends `_s$second` so the same cue spanning several
  seconds stays unique (VF-12) — see [video-frames.md](video-frames.md).
- Paint priority in `cue_row`: **flash > selection > vocab highlight > body**.
  Gap-flash uses `ColorScheme.tertiaryContainer` / `onTertiaryContainer`
  (not `ReaderColors.lemmaHighlight`).
- Do **not** put a competing cue-level `InkWell`/`GestureDetector` on the same
  tap.

## Chrome: CTA + Add bottom sheet

1. Non-empty **non-vocab** selection → `Scaffold.bottomNavigationBar` with
   `Key('readerAddVocabCta')`, label **Add to vocabulary**. Empty / in-vocab
   edit path → `null`.
2. CTA → `showModalBottomSheet` (`Key('readerAddVocabSheet')`):
   - surface read-only (`readerAddVocabSurface`) — single token surface, or
     casefold(join) for multi-token
   - **Translate** (`readerAddVocabTranslate`) → `OfflineTranslateFn(text: surface)`;
     loading on button; after first start stays disabled until sheet closes;
     Ok fills `readerAddVocabTranslation`; Err → SnackBar
     `readerAddVocabTranslateErrorSnack` (do not touch Translation)
   - attribution (`readerAddVocabTranslateAttribution`) — D4.6 copy from facade
   - translation editable, initially empty, required (`readerAddVocabTranslation`)
   - context editable, prefill = full cue text (`readerAddVocabContext`)
   - Save (`readerAddVocabSave`); validation error (`readerAddVocabValidationError`)
3. Edit sheet (in-vocab): same Translate Keys with `readerEditVocab*` prefix.
4. Persist via injected `DictionaryRepository.add`:
   - **single:** `lemma: token.lemmaKey`,
     `surfaceKey: normalizeSurface(token.surface)`
   - **phrase:** `lemma` = `surfaceKey` =
     `normalizeSurface(join(surfaces, " "))` (no WordNet)
5. Success → close sheet, clear selection, **refresh** marks via
   `markVocabTokens` ([highlight.md](highlight.md)).
6. Tokenize / mark FRB failure → plain text, taps off, SnackBar
   `readerTokenizeErrorSnack` (no auto-retry).

## DI / test seams

- `AppShell` → one shared `OfflineTranslateFn` → Library/Reader + Vocab
  (design D4.1); override in widget tests.
- `AppShell` → `LibraryScreen(dictionaryRepository:)` →
  `ReaderScreen(dictionaryRepository:)` (required).
- Optional `ReaderScreen.tokenizeCues` / `TokenizeCuesFn`,
  `markVocabTokens` / `MarkVocabTokensFn`, and shared
  `LemmaKeysForSurfacesFn` (Vocab + migration) — fake compute + real/test Drift.

## Later (not this change)

| Later | Notes |
|-------|--------|
| Instant word/phrase card chrome (beyond Translate-on-sheet) | Richer card UX; gloss engine already via `lib/data/translate/` |
| Drag / long-press select | Product may add; tap-contiguous is shipped |
| Tap on translation pane | Out of scope until a change says so |

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Instant word card on every tap | Select → CTA → sheet (non-vocab); edit sheet (in-vocab) |
| Single-token-only as current canon | Contiguous tap expand/shrink/gap-flash (one cue) |
| Dart `RegExp` / UTF-16 `substring` for tokens | Rust spans + `cueCharSlice` |
| Per-row FRB tokenize on list rebuild | Batch once on open, cache |
| Invent local glosses in the sheet | Offline Translate via `OfflineTranslateFn` facade, then Drift save |
| Dict writes outside `DictionaryRepository` | Same persist path as Vocab |
| Stub `normalizeLemma(surface)` as morph identity | Single: `token.lemmaKey`; phrase: casefold(join) |
