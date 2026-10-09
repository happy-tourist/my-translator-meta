# Split View Sync Scroll

## Layout

- Two panes: **original** cues and **translation** cues.
- Same cue **index** on both sides (Rust pairing / parallel lists).
- Flutter owns two scrollables + sync logic; Rust owns texts and alignment data.

## Sync rules

1. **Primary key = cue index**, not raw pixel `offset`.
2. When the user scrolls pane A, determine the **anchor cue** (e.g. first fully
   visible, or centered — pick one and keep it consistent).
3. Scroll pane B so the **same index** sits at the same anchor policy.
4. If line heights differ (wrap, font, missing text), index sync still wins;
   do not require 1:1 pixel offsets.
5. Programmatic jumps (tap cue, restore session position) set **both** panes to
   the same index.

## Missing translation

- Cue *i* without translation → placeholder row at index *i* (empty / “—” /
  muted). **Do not** drop the row (that shifts indices and breaks sync).
- Single-pane mode when split is off: no second controller required.

## Desync recovery

Symptoms: panes show different cue indices after fast fling, layout change, or
keyboard/inset.

Recovery:

1. Read anchor index from the pane that received the last user scroll (or the
   original pane as default authority).
2. Jump the other pane to that index (correct for extent / max scroll).
3. Optional: after font scale / rotate, re-run anchor sync once layout settles.

Avoid copying `controller.offset` blindly between panes.

## Edge cases

| Case | Behavior |
|------|----------|
| Empty document | Both panes empty state; controllers idle |
| One cue only | Sync trivial; still shared index 0 |
| Fast fling | Debounce or sync on scroll-end + settle; then index align |
| Unequal list lengths (bug) | Treat as data error; do not invent cues in Flutter — fix Rust pairing |

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| `translationController.jumpTo(original.offset)` | Jump by cue index / item scroll API |
| Collapsing missing translations | Placeholder rows |
| Independent free scroll with no link | Always index-linked in split mode |
| Sync logic in Rust | Flutter scroll; Rust only supplies parallel lines |
