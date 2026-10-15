# Split View Sync Scroll

## Layout

- Two panes: **original** and **translation** (cues **or** EPUB paragraphs).
- Flutter owns two scrollables + sync logic.
- **Subtitles (capability `reader/split`):** independent cue lists from two
  `parseSubtitles` calls; sync by nearest Dart `CueDto.startMs` (`BigInt`),
  then optional pixel settle — **no** Rust pairing / align API.
- **EPUB (capability `reader/chapters` + split):** EPUB↔EPUB chapter lists from
  `parse_epub`; sync by **shared 0-based chapter index** (not time, not
  paragraph scroll offset). Details: [chapters.md](chapters.md).

## Implementation anchors (present)

| Piece | Path / dep |
|-------|------------|
| Sync helper (subtitles) | `lib/features/reader/nearest_cue_index.dart` (`nearestCueIndex`) |
| Shared cue / paragraph row | `lib/features/reader/cue_row.dart` (`CueRow`, key prefixes) |
| Pane lists | `scrollable_positioned_list` → `ItemScrollController` + `ItemPositionsListener` |
| Orchestration | `ReaderScreen` (`lib/features/reader/reader_screen.dart`) StatefulWidget |
| EPUB chapter session | `_chapterIndex` + bottom pager (`_goToChapter`) |

## Sync rules

### Subtitles — time-first (`reader/split`)

1. **Primary key = nearest `CueDto.startMs`**, not raw pixel `offset` and not
   cue index. Compare as `BigInt`; do **not** truncate via `toInt()`.
2. Arm sync only on **user** scroll (`ScrollUpdateNotification.dragDetails` /
   scroll-start from drag); on settle (`ScrollEndNotification`) — or after load
   / enabling split with translation — read the top-visible anchor from
   `ItemPositionsListener`; map via `nearestCueIndex`; `scrollTo` that index;
   then **pixel settle** with `jumpTo(index, alignment: sourceLeadingEdge)`
   (mirror viewport alignment — not continuous offset copy).
3. Guard with a `_syncing` flag so A↔B listeners do not loop.
4. Do **not** require Rust parallel lists or placeholder rows for unequal lengths.

### EPUB — chapter-index (`reader/chapters`)

1. **Primary key = shared `_chapterIndex`** (0-based). Pager on **either**
   book pane calls `_goToChapter` and updates both panes.
2. Each pane shows paragraphs of `chapters[min(index, len-1)]` (clamp when
   lengths differ).
3. Do **not** use `nearestCueIndex` / cue `startMs` for EPUB sync.
4. On load with unequal chapter counts: attach anyway, clamp, show
   `mismatchedEpubChapterCountMessage` (RS-16 / D5).
5. Reject subtitle translation against EPUB original (and vice versa) —
   `incompatibleTranslationFormatMessage` (LD-26 / RS-08).

### Target — cue index-first (when subtitle pairing exists)

1. **Primary key = cue index**, not raw pixel `offset`.
2. When the user scrolls pane A, determine the **anchor cue** (e.g. first fully
   visible, or centered — pick one and keep it consistent).
3. Scroll pane B so the **same index** sits at the same anchor policy.
4. If line heights differ (wrap, font, missing text), index sync still wins;
   do not require 1:1 pixel offsets.
5. Programmatic jumps (tap cue, restore session position) set **both** panes to
   the same index.

## Toggle split — preserve original scroll (RS-10, RS-11)

**Subtitles:** enabling **or hiding** split remounts the original list. Before
`setState` on either direction, capture the top-visible anchor from
`ItemPositionsListener` (`index` + `itemLeadingEdge`); seed
`ScrollablePositionedList.initialScrollIndex` / `initialAlignment` on the
remounted original list so the first frame is not cue 0; after the first frame
(when `ItemScrollController.isAttached`), `jumpTo` that anchor as backup. On
Show with translation cues, run the usual sync from original.
Do **not** treat Hide as a no-op for scroll — the single-pane list is a new
widget tree and will land at cue 0 without capture+restore.

**EPUB book-mode:** chapter index is session state — Show/Hide does **not**
need cue-list remount capture for chapter; paragraph list remounts when the
chapter changes (scroll to start is expected — RC-02).

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
| Restore (subtitles) | After layout: `nearestCueIndex` by `startMs` → `jumpTo`; sync bottom if split has cues |
| Restore (EPUB) | Clamp `startMs` as chapter index; open that chapter (pager) |
| Save (subtitles) | On user scroll settle (original) and dispose/pop (`force: true`). Skip settle writes while `_syncing` |
| Save (EPUB) | On chapter pager settle / dispose — write `_chapterIndex` (not nearest cue) |

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
  still works). EPUB: empty until linked/loaded `.epub`.
- **Target (paired subtitle lists):** cue *i* without translation → placeholder
  row at index *i*. **Do not** drop the row (that shifts indices).
- Single-pane mode when split is off: no second controller required.

## Desync recovery

Symptoms: panes drift after fast fling, layout change, or keyboard/inset.

Recovery:

1. **Subtitles:** re-run time-nearest sync from the pane that received the last
   user scroll (or the original pane as default authority).
2. **EPUB:** re-apply shared `_chapterIndex` via `_goToChapter` (pager is
   authority — not paragraph scroll offset).
3. Optional: after font scale / rotate, re-run anchor sync once layout settles.

Avoid copying `controller.offset` blindly between panes.

## Edge cases

| Case | Behavior |
|------|----------|
| Empty translation pane (MVP) | Show empty + load CTA; top pane scrolls alone |
| Show split after scroll (subtitles) | Restore original top-visible cue (RS-10); do not reset to 0 |
| Hide split after scroll (subtitles) | Same capture+`jumpTo` (RS-11); do not reset to 0 |
| Reopen from Library | Subtitles: cue `startMs`; EPUB: chapter index; fail → open at top |
| Delete translation | Bottom-pane top chrome only; AppBar stays split (+ video for subtitles) — never Delete |
| Video pages present | Pages key outside remount; pages→translation one-way — **not** for EPUB |
| One cue only | Sync trivial |
| Fast fling (subtitles) | Debounce or sync on scroll-end + settle; then time |
| Unequal cue lengths (subtitles) | Expected; nearest `startMs` — do **not** invent Rust pairing |
| Unequal EPUB chapter counts | Clamp index; show mismatch notice (RS-16) |
| Mixed EPUB↔subtitle load | Reject with incompatible-format message |

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| `translationController.jumpTo(original.offset)` | Jump by nearest time (subtitles) or chapter index (EPUB) |
| Inventing a Rust pairing API for MVP split | Two `parseSubtitles` + Flutter `nearestCueIndex` |
| Time-sync for EPUB chapters | Shared `_chapterIndex` + pager |
| Continuous pixel-mirror during fling | Settle → time/chapter match → optional pixel fine-adjust |
| Sync logic in Rust | Flutter owns scroll; Rust only parse |
| Delete translation in AppBar | Bottom-pane top chrome (`readerTranslationDelete*`) |
| Ignoring remount on Show/Hide split (subtitles) | Capture top-visible → post-frame `jumpTo` (RS-10 / RS-11) |
| Disposing video pages key with remounted cue lists | Keep `VideoFramePages` / key outside remount |
| Driving video pages from bottom scroll when video on | One-way pages→translation only |
| Persisting translation-pane scroll as resume | Persist original `startMs` / chapter index only |
| Blocking open / settle on save/load fail | Best-effort; assert-log; continue (D10) |
