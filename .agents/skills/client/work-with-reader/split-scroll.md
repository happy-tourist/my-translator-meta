# Split View One-Shot Sync

## Layout

- Two panes: **original** and **translation** (cues **or** EPUB paragraphs).
- Flutter owns two scrollables + **one-shot** align helpers (no realtime
  settle / pager / video-page sync).
- **Subtitles (capability `reader/split`):** independent cue lists from two
  `parseSubtitles` calls; one-shot by nearest Dart `CueDto.startMs` (`BigInt`)
  — **no** Rust pairing / align API.
- **EPUB (capability `reader/chapters` + split):** EPUB↔EPUB chapter lists from
  `parse_epub`; one-shot by clamping translation chapter to original; after
  that each pane has an **independent** chapter index. Details:
  [chapters.md](chapters.md).

## Implementation anchors (present)

| Piece | Path / dep |
|-------|------------|
| Sync helper (subtitles) | `lib/features/reader/nearest_cue_index.dart` (`nearestCueIndex`) |
| Shared cue / paragraph row | `lib/features/reader/cue_row.dart` (`CueRow`, key prefixes) |
| Pane lists | `scrollable_positioned_list` → `ItemScrollController` + `ItemPositionsListener` |
| Orchestration | `ReaderScreen` (`lib/features/reader/reader_screen.dart`) StatefulWidget |
| EPUB chapter session | `_originalChapterIndex` + `_translationChapterIndex` + per-pane pager (`_goToChapter(pane, …)`) |

## Sync rules

### One-shot call sites only

Run time-nearest (subtitles) or chapter align (EPUB) **only** when:

1. User **enables** (Show) split and translation content is already in session; or
2. User **successfully loads** translation while split is already on.

Do **not** sync on: scroll settle / fling, `_restoreInitialReadingAnchor`,
video page change, Load video, Remove video, or EPUB pager prev/next.

### Subtitles — time-first (`reader/split`)

1. **Primary key = nearest `CueDto.startMs`**, not raw pixel `offset` and not
   cue index. Compare as `BigInt`; do **not** truncate via `toInt()`.
2. On one-shot: read the top-visible anchor from `ItemPositionsListener`; map
   via `nearestCueIndex`; `scrollTo` / `jumpTo` that index; optional **pixel
   settle** once with `jumpTo(index, alignment: sourceLeadingEdge)`.
3. Guard with a `_syncing` flag so programmatic jumps do not loop listeners.
4. Do **not** require Rust parallel lists or placeholder rows for unequal lengths.
5. After one-shot, panes scroll independently (RS-06: settle MUST NOT move the
   other pane).

### EPUB — independent chapter indices (`reader/chapters`)

1. **Session state = two indices:** `_originalChapterIndex` and
   `_translationChapterIndex` (0-based). Pager on a pane calls
   `_goToChapter(fromOriginal:, next:)` for **that pane only** (RS-14).
2. One-shot: `_translationChapterIndex = clamp(_originalChapterIndex, 0,
   translationN-1)` on Show / load translation (RS-19 / RS-20).
3. Each pane shows paragraphs of `chapters[min(index, len-1)]` for **its**
   index.
4. Do **not** use `nearestCueIndex` / cue `startMs` for EPUB align.
5. On load with unequal chapter counts: attach anyway, clamp, show
   `mismatchedEpubChapterCountMessage` (RS-16); one-shot still runs.
6. Reject subtitle translation against EPUB original (and vice versa) —
   `incompatibleTranslationFormatMessage` (LD-26 / RS-08).

### Target — cue index-first (when subtitle pairing exists)

1. **Primary key = cue index**, not raw pixel `offset`.
2. When pairing exists, programmatic jumps (tap cue, restore session) MAY set
   both panes; until then MVP stays time-nearest **one-shot** only.
3. If line heights differ, index sync still wins; do not require 1:1 pixel
   offsets.

## Toggle split — preserve original scroll (RS-10, RS-11)

**Subtitles:** enabling **or hiding** split remounts the original list. Before
`setState` on either direction, capture the top-visible anchor from
`ItemPositionsListener` (`index` + `itemLeadingEdge`); seed
`ScrollablePositionedList.initialScrollIndex` / `initialAlignment` on the
remounted original list so the first frame is not cue 0; after the first frame
(when `ItemScrollController.isAttached`), `jumpTo` that anchor as backup. On
Show with translation cues, run **one-shot** sync from original (RS-18).
Do **not** treat Hide as a no-op for scroll — the single-pane list is a new
widget tree and will land at cue 0 without capture+restore.

**EPUB book-mode:** on Show with translation chapters, one-shot-align
translation chapter (RS-19). Show/Hide does **not** need cue-list remount
capture for chapter; paragraph list remounts when the chapter changes
(scroll to start is expected — RC-02).

When implementing `reader/video-frames` (**subtitle docs only**), original pane
is either the cue list **or** `VideoFramePages` (not both). Keep the pages
`GlobalKey` / state outside the Show/Hide remount branch. Full contracts:
[video-frames.md](video-frames.md). EPUB hides video chrome entirely
([chapters.md](chapters.md)).

## Reading resume (RS-12 / LD-17 / LD-28)

Persist the **original** pane position only (not translation scroll):

| Piece | Contract |
|-------|----------|
| Columns | `Documents.lastReadStartMs` (nullable int), `lastReadAlignment` (nullable real, clamp `0.0…1.0`) |
| Repo | `ReadingAnchor` + `DocumentRepository.saveReadingAnchor` / `loadReadingAnchor` |
| Semantics | **Subtitles:** cue `startMs` (ms). **EPUB:** 0-based **chapter index** (same column; format-gated at call sites) |
| Open | Library best-effort `loadReadingAnchor` → pass `initialReadingAnchor` into `ReaderScreen` (fail must not block open) |
| Restore (subtitles) | After layout: `nearestCueIndex` by `startMs` → `jumpTo` **original only**; do **not** sync translation until next Show / load (RS-12) |
| Restore (EPUB) | Clamp `startMs` as chapter index → `_originalChapterIndex` only; translation aligns on next Show / load |
| Save (subtitles) | On user scroll settle (original) and dispose/pop (`force: true`). Skip settle writes while `_syncing` |
| Save (EPUB) | On chapter pager settle / dispose — write `_originalChapterIndex` (not translation) |

Do **not** write the translation pane’s scroll as the resume key.

## Book split divider (D13 / RS-17)

In EPUB book split, drive pane heights with a `ValueNotifier` (or equivalent)
so drag updates only `SizedBox` heights — do **not** `setState` the whole
`ReaderScreen` on every pointer move (that rebuilds both paragraph lists).
Subtitle split MAY keep the older setState divider path.

## Delete chrome (RS-07 / D8)

- **AppBar** actions: split toggle (+ Load/Remove video when `reader/video-frames`)
  — **not** Delete translation.
- When split is on and translation cues exist **or** a stuck link
  (`translationDocumentId` without cues): show Delete in a **top chrome strip
  of the bottom pane** (`readerTranslationDelete` /
  `readerTranslationDeleteEmpty`), above the cue list or empty content.
- Keep bottom pane height ≥ Material min tap target (`kMinInteractiveDimension`)
  when that chrome is shown so the IconButton does not overflow the divider
  clamp.
- Stuck-link empty: Delete in chrome first; Load CTA only after unlink.

## Missing translation

- **MVP:** bottom pane empty until a translation file is loaded; no per-cue
  placeholders required (subtitle lists may differ in length; time-nearest
  one-shot still works). EPUB: empty until linked/loaded `.epub`.
- **Target (paired subtitle lists):** cue *i* without translation → placeholder
  row at index *i*. **Do not** drop the row (that shifts indices).
- Single-pane mode when split is off: no second controller required.

## Edge cases

| Case | Behavior |
|------|----------|
| Empty translation pane (MVP) | Show empty + load CTA; top pane scrolls alone |
| Show split after scroll (subtitles) | Restore original top-visible cue (RS-10); one-shot bottom if translation present (RS-18) |
| Hide split after scroll (subtitles) | Same capture+`jumpTo` (RS-11); do not reset to 0 |
| Reopen from Library | Subtitles: cue `startMs`; EPUB: original chapter index; fail → open at top; no translation sync until Show |
| Delete translation | Bottom-pane top chrome only; AppBar stays split (+ video for subtitles) — never Delete |
| Video pages present | Pages key outside remount; page change does **not** sync translation — **not** for EPUB |
| One cue only | One-shot trivial |
| Scroll / fling after one-shot | Panes stay independent (RS-06) |
| Unequal cue lengths (subtitles) | Expected; nearest `startMs` — do **not** invent Rust pairing |
| Unequal EPUB chapter counts | Clamp index; show mismatch notice (RS-16); one-shot still runs |
| Mixed EPUB↔subtitle load | Reject with incompatible-format message |
| EPUB pager after one-shot | Each pane keeps its own chapter index (RS-14) |

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Settle / fling realtime sync (RS-06) | One-shot only on Show + successful load translation |
| Shared `_chapterIndex` moving both EPUB panes | Dual indices + per-pane `_goToChapter` |
| Sync on video page / Load / Remove video | One-shot on Show / load **translation** only |
| Sync translation on open resume | Restore original only; align on next Show / load |
| `translationController.jumpTo(original.offset)` | Jump by nearest time (subtitles) or chapter index (EPUB) once |
| Inventing a Rust pairing API for MVP split | Two `parseSubtitles` + Flutter `nearestCueIndex` |
| Time-sync for EPUB chapters | Dual chapter indices + one-shot clamp |
| Continuous pixel-mirror during fling | Independent panes after one-shot |
| Sync logic in Rust | Flutter owns scroll; Rust only parse |
| Delete translation in AppBar | Bottom-pane top chrome (`readerTranslationDelete*`) |
| Ignoring remount on Show/Hide split (subtitles) | Capture top-visible → post-frame `jumpTo` (RS-10 / RS-11) |
| Disposing video pages key with remounted cue lists | Keep `VideoFramePages` / key outside remount |
| Persisting translation-pane scroll as resume | Persist original `startMs` / chapter index only |
| Blocking open / settle on save/load fail | Best-effort; assert-log; continue (D10) |
