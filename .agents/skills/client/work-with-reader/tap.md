# Tap selection + add to vocabulary (MVP)

Canon for capability `reader/tap`. **MVP chrome = bottom CTA + modal bottom
sheet**, not an instant word-card / offline-translate card. Morph lemma
highlight and FRB translate remain later ([highlight.md](highlight.md),
`work-with-translate`).

## Preconditions

1. On Reader open, Flutter batch-tokenizes **original** cues once:
   `texts = originalCues.map((c) => c.textLines.join('\n'))` → injectable
   `tokenizeCues` (default FRB `tokenizeCues`). Cache in state; rebuilds only
   read the cache.
2. Display string for paint **and** tokenize must match (`join('\n')`).
3. Char spans are Unicode scalar indices. Slice with `cueCharSlice` /
   `runes` (`cue_char_slice.dart`) — **not** Dart UTF-16 `substring`.
4. Translation pane: plain text only (no token Keys / selection).

## Selection state machine (original pane)

State: `activeCueIndex` + inclusive token range `[start, end]` (`tokenIndex`
from DTO). Empty → no CTA.

| Tap (tokens ready, same cue) | Result |
|------------------------------|--------|
| Selection empty → token `T` | Select `[T,T]`; show CTA |
| Adjacent outside range (`T == S-1` or `T == E+1`) | Expand to include `T` |
| Inside multi-token range | Shrink to `[T,T]` |
| Same single selected token again | No-op; CTA stays |
| Gap (non-adjacent) | Reset to `[T,T]` + ~300ms flash on new anchor |
| Other original cue | Clear previous; start `[T,T]` there (no gap flash) |

- Gestures: **tap only** (no drag / long-press phrase select in MVP).
- Per-token hit targets (`GestureDetector` / span); Keys
  `readerCueToken_${cueIndex}_${tokenIndex}`.
- Do **not** put a competing cue-level `InkWell`/`GestureDetector` on the same
  tap.
- Phrase surface (RT-10): contiguous cue slice from first token `charStart` to
  last token `charEnd` (via `cueCharSlice`), including separators between words.

## Chrome: CTA + bottom sheet

1. Non-empty selection → `Scaffold.bottomNavigationBar` with
   `Key('readerAddVocabCta')`, label **Add to vocabulary**. Empty → `null`.
2. CTA → `showModalBottomSheet` (`Key('readerAddVocabSheet')`):
   - surface/phrase read-only (`readerAddVocabSurface`)
   - translation editable, initially empty, required (`readerAddVocabTranslation`)
   - context editable, prefill = full cue text (`readerAddVocabContext`)
   - Save (`readerAddVocabSave`); validation error (`readerAddVocabValidationError`)
3. Persist via injected `DictionaryRepository.add` (same `normalizeLemma` +
   upsert as Vocab UI). Empty translation after trim → no persist, error in sheet.
4. Success → close sheet, clear selection (CTA hides). **No** lemma highlight
   refresh in this MVP.
5. Tokenize FRB/call failure → plain text, taps off, SnackBar
   `readerTokenizeErrorSnack` (no auto-retry).

## DI / test seams

- `AppShell` → `LibraryScreen(dictionaryRepository:)` →
  `ReaderScreen(dictionaryRepository:)` (required).
- Optional `ReaderScreen.tokenizeCues` / `TokenizeCuesFn` (like
  `ParseSubtitlesFn`) for widget tests — fake tokenize + real/test Drift repo.

## Later (not MVP)

| Later | Notes |
|-------|--------|
| Word / phrase card with offline gloss | FRB translate + richer chrome |
| Drag / long-press select | Product may add; keep one-cue contiguous default |
| Morph lemma + highlight refresh after save | [highlight.md](highlight.md) |
| Tap on translation pane | Out of scope until a change says so |

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Instant word card on every tap | Select → CTA → sheet |
| Dart `RegExp` / UTF-16 `substring` for tokens | Rust spans + `cueCharSlice` |
| Per-row FRB tokenize on list rebuild | Batch once on open, cache |
| Invent local translation in the sheet | Manual translation field → Drift |
| Dict writes outside `DictionaryRepository` | Same persist path as Vocab |
