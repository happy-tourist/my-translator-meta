# Video second timeline (`reader/video-frames`)

Read with [SKILL.md](SKILL.md). Capability / change: OpenSpec `reader/video-frames`
(`add-reader-video-frames`). **No Rust / FRB video API** — probe + JPEG extract
are Flutter-only (native FFmpegKit / web canvas). Persist = Drift via
`work-with-drift`.

## Paths

| Piece | Path | Role |
|-------|------|------|
| Timeline UI | `lib/features/reader/video_second_timeline.dart` | Virtualized one-row-per-second list; injectable extractor + cache |
| Second math | `lib/features/reader/video_second_math.dart` | Pure `clampReadingTimeToSecond` / `videoTimelineSecondCount` / `isValidTimelineSecond` (VF-02 / VF-05) |
| Cue under row | `lib/features/reader/cue_text_at_second.dart` | Pure `cueIndexAtSecond` + `cueTextAtSecond`; half-open `[startMs,endMs)` (D5 / VF-04) |
| Cache seam | `lib/features/reader/video_frame_cache.dart` | `VideoFrameCache` + `Memory…` / `DocumentRepositoryVideoFrameCache` |
| Extract facade | `lib/data/video/video_frame_extractor.dart` | Abstract + `FakeVideoFrameExtractor` |
| Platform select | `lib/data/video/video_frame_extractor_platform.dart` | Conditional `io` / `html` (mirror translate) |
| Native / Web impl | `…_native.dart` / `…_web.dart` | FFmpegKit path vs canvas seek |
| Usable path | `lib/data/video/video_usable_path*.dart` | Sandbox / blob ref for extract (reject bare `content://`) |
| Picker | `lib/data/video/reader_video_picker.dart` | MP4/MKV only (`kReaderVideoExtensions`); cancel → `null` |
| Typed errors | `lib/data/video/video_frame_errors.dart` | `VideoFrameException` / `VideoFrameErrorKind` + locked EN copy |
| Reader wiring | `lib/features/reader/reader_screen.dart` | AppBar Load/Remove; original pane = timeline **or** cues (not both) |
| Drift attach | `DocumentRepository` + schema 5 | `attachVideo` / `detachVideo` / frame JPEG BLOB — [repositories.md](../work-with-drift/repositories.md) |

Package imports: `package:my_translator_app/data/video/…`,
`package:my_translator_app/features/reader/…`.

**Removed (page-of-3 UX):** `video_frame_strip.dart`, `video_frame_page_math.dart` —
do not revive; interleaved `VideoSecondTimeline` is the canon (design D4).

## UX contracts

- On Load / restore: **replace** the original cue list with a vertical
  second-timeline (`VideoSecondTimeline`). Cue list is hidden, not stacked under
  frames (VF-01). One row per whole-video second `0..durationSeconds-1` (VF-02);
  do not show timeline when `durationSeconds < 1` (VF-11).
- Each row: JPEG (or load/error placeholder) + timestamp + optional cue text
  (empty when no covering cue — VF-03). Lazy extract around viewport; cache hit
  skips re-extract. Tap/highlight: `VideoSecondTimeline.buildCueText` embeds
  `CueRow` with `showTimestamp: false`, zero padding, and `keySuffix: 's$second'`
  so one cue spanning many seconds keeps unique Keys (VF-12).
- AppBar: Load / Remove video **next to** split toggle; Delete translation stays
  in bottom-pane chrome only.
- Jump on Load / restore: `T = clamp(floor(readingTimeMs/1000), 0, duration-1)`
  via `clampReadingTimeToSecond` (VF-05 / VF-13). Prefer persisted `last_second`
  (column `videoLastFramePage` reused as second index) when restoring.
- Remove: capture current timeline second → detach → restore cue list →
  `scrollTo` nearest cue at `T*1000` (VF-08).
- Split: frames **only** in original pane (VF-09). Original timeline scroll →
  sync translation via `nearestCueIndex(T*1000)` (VF-06). Bottom scroll MUST NOT
  move timeline when video is on (VF-07). Show/Hide split: keep timeline
  scroll controllers / key outside the dispose branch that remounts cue lists
  ([split-scroll.md](split-scroll.md)).
- Row extract fail → placeholder in that row; neighbors + text gestures OK
  (VF-10). Tap/highlight on cue text under frames unchanged (VF-12).
- Dead durable link: keep JPEG cache; re-pick with `clearFrameCache: false`
  (LD-22). Fail open video meta MUST NOT block Reader / Library open.
- Bad pick / probe fail → visible SnackBar (`VideoFrameException.userMessage`);
  timeline not shown for that choice. Picker cancel → silent (not an error).

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
`videoLastFramePage` (**semantics = last_second** after UX revision) +
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

Widget tests: inject fakes into `VideoSecondTimeline` / `ReaderScreen`; do not
call real FFmpegKit in CI. Pure helpers:
`test/features/reader/video_second_math_test.dart`,
`cue_text_at_second_test.dart`, `video_second_timeline_test.dart`; Reader
VF-01…VF-13 orchestration in `reader_screen_test.dart`.

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Rust / FRB video API | Dart facade only |
| Page-of-3 strip / pager above cues | Interleaved second timeline replacing cue list |
| Cue list + timeline stacked together | Timeline **or** cues in original pane |
| Bottom scroll driving timeline when video on | Original→translation one-way only |
| Raw FFmpeg / DOM strings in SnackBar | `VideoFrameException.userMessage` |
| Mixing two native FFmpegKit packages | Single pinned `ffmpeg_kit_flutter_new_video` |
| Treating picker cancel as error | Silent return |
| Blocking Reader open on dead video link | Best-effort restore; keep cache; allow re-pick |
| Treating `videoLastFramePage` as page-of-3 index | Store / restore **second** index |
