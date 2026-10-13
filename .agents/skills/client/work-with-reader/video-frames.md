# Video frame strip (`reader/video-frames`)

Read with [SKILL.md](SKILL.md). Capability / change: OpenSpec `reader/video-frames`
(`add-reader-video-frames`). **No Rust / FRB video API** — probe + JPEG extract
are Flutter-only (native FFmpegKit / web canvas). Persist = Drift via
`work-with-drift`.

## Paths

| Piece | Path | Role |
|-------|------|------|
| Strip UI | `lib/features/reader/video_frame_strip.dart` | 3 slots/page + pager; injectable extractor + cache |
| Page math | `lib/features/reader/video_frame_page_math.dart` | Pure 1-based page → seconds (VF-02) |
| Cue under slot | `lib/features/reader/cue_text_at_second.dart` | Pure half-open `[startMs,endMs)` join (D5 / VF-04) |
| Cache seam | `lib/features/reader/video_frame_cache.dart` | `VideoFrameCache` + `Memory…` / `DocumentRepositoryVideoFrameCache` |
| Extract facade | `lib/data/video/video_frame_extractor.dart` | Abstract + `FakeVideoFrameExtractor` |
| Platform select | `lib/data/video/video_frame_extractor_platform.dart` | Conditional `io` / `html` (mirror translate) |
| Native / Web impl | `…_native.dart` / `…_web.dart` | FFmpegKit path vs canvas seek |
| Usable path | `lib/data/video/video_usable_path*.dart` | Sandbox / blob ref for extract (reject bare `content://`) |
| Picker | `lib/data/video/reader_video_picker.dart` | MP4/MKV only (`kReaderVideoExtensions`); cancel → `null` |
| Typed errors | `lib/data/video/video_frame_errors.dart` | `VideoFrameException` / `VideoFrameErrorKind` + locked EN copy |
| Reader wiring | `lib/features/reader/reader_screen.dart` | AppBar Load/Remove; strip **above** split remount |
| Drift attach | `DocumentRepository` + schema 5 | `attachVideo` / `detachVideo` / frame JPEG BLOB — [repositories.md](../work-with-drift/repositories.md) |

Package imports: `package:my_translator_app/data/video/…`,
`package:my_translator_app/features/reader/…`.

## UX contracts

- Strip sits in a common `Column` **above** the single-pane / split remount
  branch so Show/Hide split does not reset page or in-flight extract
  ([split-scroll.md](split-scroll.md)).
- AppBar: Load / Remove video **next to** split toggle; Delete translation stays
  in bottom-pane chrome only.
- Pagination: 3 whole-second slots per **1-based** page; `totalSeconds =
  floor(duration)`; do not show strip when `totalSeconds < 1` (VF-11).
- Sync: **one-way** frames → cues only — page change → `nearestCueIndex` jump on
  original (+ translation if split). Cue / bottom scroll MUST NOT move strip page.
- Slot extract fail → placeholder in that slot; other slots + cues OK (VF-10).
- Dead durable link: keep JPEG cache; re-pick with `clearFrameCache: false`
  (LD-22). Fail open video meta MUST NOT block Reader / Library open.
- Bad pick / probe fail → visible SnackBar (`VideoFrameException.userMessage`);
  strip not shown for that choice. Picker cancel → silent (not an error).

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
`videoLastFramePage` + `DocumentVideoFrames` JPEG BLOB. Repo ops and LD-18…LD-23
→ `work-with-drift` [schema.md](../work-with-drift/schema.md) /
[repositories.md](../work-with-drift/repositories.md). Library open best-effort
restores attachment (fail must not block).

## Tests

| Seam | Fake |
|------|------|
| Extract | `FakeVideoFrameExtractor` |
| JPEG cache | `MemoryVideoFrameCache` |
| Picker | injectable `ReaderVideoPicker` (cancel → `null`) |
| Pure math / cue text | unit tests — no widget host |

Widget tests: inject fakes into `VideoFrameStrip` / `ReaderScreen`; do not call
real FFmpegKit in CI. Pure helpers:
`test/features/reader/video_frame_page_math_test.dart`,
`cue_text_at_second_test.dart`, `video_frame_strip_test.dart`.

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Rust / FRB video API | Dart facade only |
| Strip inside split remount body | Strip above single/split branch |
| Cue scroll driving strip page | Frames→cues one-way only |
| Raw FFmpeg / DOM strings in SnackBar | `VideoFrameException.userMessage` |
| Mixing two native FFmpegKit packages | Single pinned `ffmpeg_kit_flutter_new_video` |
| Treating picker cancel as error | Silent return |
| Blocking Reader open on dead video link | Best-effort restore; keep cache; allow re-pick |
