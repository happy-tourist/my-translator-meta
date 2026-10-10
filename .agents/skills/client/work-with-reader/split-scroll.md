# Split View Sync Scroll

## Layout

- Two panes: **original** cues and **translation** cues.
- Flutter owns two scrollables + sync logic.
- **Target (later):** same cue **index** on both sides when Rust supplies
  paired / parallel lists.
- **MVP (capability `reader/split`, change `add-reader-split`):** independent
  cue lists from two `parseSubtitles` calls; sync by nearest Dart
  `CueDto.startMs` (`BigInt`; Rust domain field is `start_ms`), then optional
  pixel settle — **no** Rust pairing / align API.

## Implementation anchors (MVP present)

| Piece | Path / dep |
|-------|------------|
| Sync helper | `lib/features/reader/nearest_cue_index.dart` (`nearestCueIndex`) |
| Shared cue row | `lib/features/reader/cue_row.dart` (`CueRow`, key prefixes) |
| Pane lists | `scrollable_positioned_list` → `ItemScrollController` + `ItemPositionsListener` |
| Orchestration | `ReaderScreen` (`lib/features/reader/reader_screen.dart`) StatefulWidget |

## Sync rules

### MVP — time-first (`reader/split`)

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

### Target — index-first (when pairing exists)

1. **Primary key = cue index**, not raw pixel `offset`.
2. When the user scrolls pane A, determine the **anchor cue** (e.g. first fully
   visible, or centered — pick one and keep it consistent).
3. Scroll pane B so the **same index** sits at the same anchor policy.
4. If line heights differ (wrap, font, missing text), index sync still wins;
   do not require 1:1 pixel offsets.
5. Programmatic jumps (tap cue, restore session position) set **both** panes to
   the same index.

## Missing translation

- **MVP:** bottom pane empty until a translation file is loaded; no per-cue
  placeholders required (lists may differ in length; time-nearest still works).
- **Target (paired lists):** cue *i* without translation → placeholder row at
  index *i* (empty / “—” / muted). **Do not** drop the row (that shifts indices).
- Single-pane mode when split is off: no second controller required.

## Desync recovery

Symptoms: panes drift after fast fling, layout change, or keyboard/inset.

Recovery:

1. **MVP:** re-run time-nearest sync from the pane that received the last user
   scroll (or the original pane as default authority).
2. **Target:** jump the other pane to the same cue index (correct for extent).
3. Optional: after font scale / rotate, re-run anchor sync once layout settles.

Avoid copying `controller.offset` blindly between panes.

## Edge cases

| Case | Behavior |
|------|----------|
| Empty translation pane (MVP) | Show empty + load CTA; top pane scrolls alone |
| One cue only | Sync trivial |
| Fast fling | Debounce or sync on scroll-end + settle; then time (MVP) or index (target) |
| Unequal list lengths (MVP) | Expected; nearest `startMs` — do **not** invent Rust pairing |
| Unequal lengths after pairing (target) | Data error — fix Rust pairing, not Flutter invent cues |

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| `translationController.jumpTo(original.offset)` | Jump by nearest time (MVP) or cue index (target) / item scroll API |
| Inventing a Rust pairing API for MVP split | Two `parseSubtitles` + Flutter `nearestCueIndex` |
| Continuous pixel-mirror during fling | Settle → time/index match → optional pixel fine-adjust |
| Sync logic in Rust | Flutter owns scroll; Rust only parse (MVP) or paired lines (target) |
