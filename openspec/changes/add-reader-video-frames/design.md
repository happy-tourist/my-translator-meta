## Context

В app уже есть paged `VideoFramePages` (столбец×3 + pager **сверху**, end-second text, `showTimestamp: false`). Explore polish (2026-10-13): pager **вниз**; при next/prev — scroll к первому кадру; у субтитра под кадром — метка **из файла** (`startMs → endMs` через `CueRow`).

Extract / Drift / AppBar orchestration без изменений.

## Goals / Non-Goals

**Goals:**

- Paged column of 3; pager chrome pinned to **bottom** of original pane.
- On page change → scroll frame list to **first** slot of the new page.
- End-second text rule unchanged (`round(endMs/1000)`).
- When cue text present under a slot → show cue time range from file (same as normal Reader `CueRow`). Empty slot → no timestamp.
- Load/restore page jump; remove → cues; split one-way sync.
- No Rust video API.

**Non-Goals:**

- Horizontal strip; infinite timeline; frames in translation pane; Rust FFmpeg.

## Decisions

### D1–D3 — Layer / extract / FFmpeg package

- Без изменений (Flutter-only decode; `ffmpeg_kit_flutter_new_video`; usable path).

### D4 — UI Reader: paged vertical triple + bottom pager (REVISION 3)

- Layout:

```
+---------------------------+
|  [frame]                  |
|  cue time + text | empty  |
|  [frame]                  |
|  ...                      |  <- scrollable
|  [frame]                  |
+---------------------------+
|  [<]   n / N   [>]        |  <- pinned bottom
+---------------------------+
```

- Cue list replaced (not strip-over-cues).
- Page math size 3; remount-safe page state.
- **On `goToPage` / prev / next:** after setState, scroll the frames `ScrollController` (or equivalent) to offset 0 / first item so the first frame of the page is visible.

### D5 — Text + timestamp (REVISION 3)

- Attachment second: `round(endMs / 1000)` (unchanged).
- Multi-cue same end → all under that frame (embed `CueRow` per cue or equivalent).
- **Timestamp:** when embedding cue text, use `CueRow` with `showTimestamp: true` → display `format(startMs) → format(endMs)` from the **cue in the subtitle file**. Do not invent frame-second labels.
- Empty slot (no ending cue): frame only; **no** time label.

### D6 — Sync

- Unchanged: Load → page containing `floor(readingTimeMs/1000)`; page change → sync translation; bottom ↛ original; Remove → cues @ T.

### D7 — Persist + durable video media (REVISION 4)

- Schema **6**: `DocumentVideoMedia` — full video `bytes` + `fileName` per original document.
- **Web:** on Load, always persist media bytes; do **not** store ephemeral `blob:` URLs as sandboxPath (use sentinel `drift-media`). On reopen, recreate session object URL from bytes.
- **Native:** may keep sandbox filesystem path; still MAY store media bytes for reopen if path dies.
- JPEG cache + last_second unchanged. Detach/delete cascades media + frames.

### D8 — Форматы pick

- Без изменений: MP4/MKV, VF-11.

### D9 — Tests (REVISION 3)

- Widget: pager keys at bottom; after next, first slot of new page visible / scroll offset ~0.
- Widget: cue under frame shows file time range; empty slot has no timestamp text.
- Prior VF-* for pages / end-second / split remain.

Чеклист — `tasks.md` §9.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Bottom bar eats height in split | Compact IconButton row; frames in Expanded |
| Scroll jump races with setState | post-frame callback after page change |
| Multi-cue timestamps | each CueRow shows its own start→end from file |

## Migration Plan

- Small UI polish on existing `VideoFramePages` + Reader `buildCueText`.
- No schema bump.

## Open Questions

- Нет продуктовых.
