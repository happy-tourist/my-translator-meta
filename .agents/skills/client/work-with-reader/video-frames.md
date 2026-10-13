# Video frame pages (`reader/video-frames`)

Read with [SKILL.md](SKILL.md). Capability / change: OpenSpec `reader/video-frames`
(`add-reader-video-frames`). **No Rust / FRB video API** — probe + JPEG extract
are Flutter-only (native FFmpegKit / web canvas). Persist = Drift via
`work-with-drift`.

## Paths

| Piece | Path | Role |
|-------|------|------|
| Pages UI | `lib/features/reader/video_frame_pages.dart` | Vertical column of ≤3 frames + pager arrows under list; injectable extractor + cache |
| Page math | `lib/features/reader/video_frame_page_math.dart` | Pure page↔seconds (`videoFramePageCount` / `videoFrameSecondsForPage` / `videoFramePageFirstSecond` / `videoFramePageForSecond`) |
| Second helpers | `lib/features/reader/video_second_math.dart` | `clampReadingTimeToSecond` / `clampReadingTimeToPage` (VF-05) |
| Cue under slot | `lib/features/reader/cue_text_at_second.dart` | `cueAttachmentSecond` / `cueIndicesAtEndSecond` / `cueTextAtSecond` — `round(endMs/1000)` (D5 / VF-04) |
| Cache seam | `lib/features/reader/video_frame_cache.dart` | `VideoFrameCache` + `Memory…` / `DocumentRepositoryVideoFrameCache` |
| Extract facade | `lib/data/video/video_frame_extractor.dart` | Abstract + `FakeVideoFrameExtractor` |
| Platform select | `lib/data/video/video_frame_extractor_platform.dart` | Conditional `io` / `html` (mirror translate) |
| Native / Web impl | `…_native.dart` / `…_web.dart` | FFmpegKit path vs canvas seek |
| Usable path | `lib/data/video/video_usable_path*.dart` | Sandbox / blob ref for extract (reject bare `content://`) |
| Picker | `lib/data/video/reader_video_picker.dart` | MP4/MKV only (`kReaderVideoExtensions`); cancel → `null` |
| Typed errors | `lib/data/video/video_frame_errors.dart` | `VideoFrameException` / `VideoFrameErrorKind` + locked EN copy |
| Reader wiring | `lib/features/reader/reader_screen.dart` | AppBar Load/Remove; original pane = pages **or** cues (not both) |
| Drift attach | `DocumentRepository` + schema 5 | `attachVideo` / `detachVideo` / frame JPEG BLOB — [repositories.md](../work-with-drift/repositories.md) |

Package imports: `package:my_translator_app/data/video/…`,
`package:my_translator_app/features/reader/…`.

**Removed:** infinite `VideoSecondTimeline` / horizontal `VideoFrameStrip` over cues —
do not revive; **paged vertical triple + bottom pager** is the canon
(design D4 revision 3).

## UX contracts

- On Load / restore: **replace** the original cue list with a **column of three**
  frames + pager arrows (`VideoFramePages`). Cue list is hidden, not stacked under
  frames (VF-01). One model slot per whole-video second `0..durationSeconds-1`
  (VF-02); UI shows ≤3 per page. Do not show pages when `durationSeconds < 1`
  (VF-11).
- **Pager chrome** (`[<] n/N [>]`) is pinned at the **bottom** of the original
  pane, under the scrollable frame list — not above it (VF-14).
- On page change (`prev` / `next` / `goToPage`): scroll the frame list to the
  **first** slot (`ScrollController.jumpTo(0)` in a post-frame callback)
  (VF-15).
- Each slot: JPEG (or load/error placeholder) + optional cue text. Lazy extract
  for the current page; cache hit skips re-extract. Tap/highlight: `buildCueText`
  embeds `CueRow` with `showTimestamp: true` (file `startMs → endMs`), zero
  padding, and `keySuffix: 's$second'` (VF-04 / VF-12). Empty slot → no CueRow /
  no time label (VF-03).
- Cue text under second `t`: all cues with `round(endMs/1000) == t`, in list order;
  join `textLines` / multi-cue with `\n`. Empty on start/middle seconds of the same
  cue (VF-04). Empty text → frame remains without timestamp (VF-03).
- AppBar: Load / Remove video **next to** split toggle; Delete translation stays
  in bottom-pane chrome only.
- Jump on Load / restore: `T = clamp(floor(readingTimeMs/1000), 0, duration-1)`
  via `clampReadingTimeToSecond` → open **page containing T**
  (`videoFramePageForSecond`) (VF-05 / VF-13). Prefer persisted `last_second`
  (column `videoLastFramePage` reused as second index) when restoring; page =
  `videoFramePageForSecond(second, duration)`.
- Remove: capture current page’s first second (`videoFramePageFirstSecond` /
  `VideoFramePagesState.currentSecond`) → detach → restore cue list →
  `scrollTo` nearest cue at `T*1000` (VF-08).
- Split: frames **only** in original pane (VF-09). Original **page** change →
  sync translation via `nearestCueIndex(T*1000)` where T =
  `videoFramePageFirstSecond(page, duration)` (VF-06). Bottom scroll MUST NOT
  move pages when video is on (VF-07). Show/Hide split: keep `VideoFramePages` /
  key outside the dispose branch that remounts cue lists
  ([split-scroll.md](split-scroll.md)).
- Slot extract fail → placeholder in that slot; neighbors + text gestures OK
  (VF-10). Tap/highlight on cue text under frames unchanged (VF-12).
- Dead durable link: keep JPEG cache; re-pick with `clearFrameCache: false`
  (LD-22). Fail open video meta MUST NOT block Reader / Library open.
- Bad pick / probe fail → visible SnackBar (`VideoFrameException.userMessage`);
  pages not shown for that choice. Picker cancel → silent (not an error).

## Extract facade (platform)

Same pattern as `lib/data/translate/`:

1. Abstract API + test fake in `video_frame_extractor.dart`.
2. Conditional export via `video_frame_extractor_platform.dart`.
3. Production: `createPlatformVideoFrameExtractor()`; tests inject
   `FakeVideoFrameExtractor` (never hit FFmpeg / canvas).

Native needs a **usable filesystem path** (`video_usable_path`); persist
`videoUri` may be `content://` (bookmark only). Web: blob / handle id — no WASM
video crate.

Deps (`pubspec.yaml`): `ffmpeg_kit_flutter_new_video` (native; do **not** mix a
second FFmpegKit package), `web` (canvas interop). See `work-with-env-deploy`.

## Persist (pointer)

Schema 5: `Documents.videoUri` / `videoSandboxPath` / `videoDurationSec` /
`videoLastFramePage` (**semantics = last_second**; page =
`videoFramePageForSecond` on restore) +
`DocumentVideoFrames` JPEG BLOB. Repo ops and LD-18…LD-23 → `work-with-drift`
[schema.md](../work-with-drift/schema.md) /
[repositories.md](../work-with-drift/repositories.md). Library open best-effort
restores attachment (fail must not block).

## Tests

| Seam | Fake |
|------|------|
| Extract | `FakeVideoFrameExtractor` |
| JPEG cache | `MemoryVideoFrameCache` |
| Picker | injectable `ReaderVideoPicker` (cancel → `null`) |
| Pure math / cue text | unit tests — no widget host |

Widget tests: inject fakes into `VideoFramePages` / `ReaderScreen`; do not
call real FFmpegKit in CI. Pure helpers:
`test/features/reader/video_frame_page_math_test.dart`,
`video_second_math_test.dart`, `cue_text_at_second_test.dart`,
`video_frame_pages_test.dart` (VF-01…VF-04, VF-10, VF-14, VF-15); Reader
VF-01…VF-13 in `reader_screen_test.dart`.

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Rust / FRB video API | Dart facade only |
| Infinite all-seconds list / top pager | Paged column of 3; pager **under** frames |
| Keeping list scroll after next/prev | `jumpTo(0)` / first slot visible (VF-15) |
| `showTimestamp: false` under frames with text | `CueRow(showTimestamp: true)` file start→end |
| Invented “frame second” time labels | Subtitle-file `startMs → endMs` only; empty = no label |
| Horizontal strip of 3 over unchanged cue list | Replace cue list with vertical triple |
| Cue list + pages stacked together | Pages **or** cues in original pane |
| Interval-cover `[start,end)` text | `round(endMs/1000)` end-second text |
| Bottom scroll driving pages when video on | Original→translation one-way only |
| Raw FFmpeg / DOM strings in SnackBar | `VideoFrameException.userMessage` |
| Mixing two native FFmpegKit packages | Single pinned `ffmpeg_kit_flutter_new_video` |
| Treating picker cancel as error | Silent return |
| Blocking Reader open on dead video link | Best-effort restore; keep cache; allow re-pick |
| Treating `videoLastFramePage` as 1-based page index | Store / restore **second** → `videoFramePageForSecond` |
