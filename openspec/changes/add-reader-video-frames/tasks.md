## 1. Dependencies и фасад extract

- [x] 1.1 Прочитать `design.md` (D2, D3), `proposal.md` Scope/Out of scope; skills `work-with-env-deploy`, `client-work-with-structure`
- [x] 1.2 В sibling `my_translator_app` подключить native FFmpegKit Dart-пакет по D3 (ffmpeg-kit-next path **или** fallback `ffmpeg_kit_flutter_new` min/video); зафиксировать выбранный artifact в комментарии/pubspec; `flutter pub get`
- [x] 1.3 Добавить web-зависимости по необходимости (`web` / interop); без WASM video crate
- [x] 1.4 Ввести абстрактный `VideoFrameExtractor` + stub/fake для тестов (`probeDuration`, `extractJpegAtSecond`) в `lib/data/` (рядом с translate facade pattern / conditional import)
- [x] 1.5 Native path helper: после pick — usable filesystem path (picker `path` или sandbox copy); **не** передавать сырой `content://` в FFmpeg (D2)

## 2. Persist video attachment + frame cache

- [x] 2.1 Прочитать `design.md` (D7), delta `specs/library/documents/spec.md` (LD-18…LD-22); skill `work-with-drift` (+ schema.md)
- [x] 2.2 Расширить Drift schema (bump с v4): meta привязки видео на оригинал + хранение JPEG посещённых секунд (файлы native / BLOB или OPFS на web) + migration
- [x] 2.3 `DocumentRepository`: attach / detach / get attachment / put+get frame cache; cascade в delete оригинала (LD-18, LD-21)
- [x] 2.4 Unit-тесты репозитория: LD-18, LD-20, LD-21, LD-22, LD-23 (dead link + cache + re-attach + replace clears prior cache)

## 3. Native и Web backends

- [x] 3.1 Прочитать `design.md` (D2, D3, D8); skill `client-work-with-errors` (user-visible errors)
- [x] 3.2 Реализовать `NativeFfmpegFrameExtractor` (probe + JPEG at second) за фасадом по usable path
- [x] 3.3 Реализовать `WebCanvasFrameExtractor` (`package:web` / canvas seek) за тем же фасадом
- [x] 3.4 Platform select default extractor; отдельный injectable `ReaderVideoPicker` + фильтр MP4/MKV (и понятная ошибка VF-11)

## 4–7. Prior presentation (DONE, superseded by §8)

> Infinite interleaved `VideoSecondTimeline` + interval-cover text + timestamp — shipped then product-rejected.
> Keep extract/Drift/AppBar orchestration; rewrite UI per §8.

- [x] 4.x–7.x prior tasks (historical)

## 8. UX revision 2 — paged vertical triple + end-second text

- [x] 8.1 Прочитать revised `proposal.md` / `design.md` D4–D6 / `specs/reader/video-frames/spec.md` (VF-01…VF-13); skill `work-with-reader/video-frames.md`
- [x] 8.2 Pure helpers: page-of-3 math (page ↔ seconds); `round(endMs/1000)` → attachment second + multi-cue collect; unit VF-02/VF-04 (no timestamp contract)
- [x] 8.3 UI: replace infinite timeline with **column of 3 frames + pager arrows** in original body (replace cue list); no timestamp under frames; empty text OK (VF-01…VF-03, VF-10)
- [x] 8.4 Wire Reader: Load/restore → jump to **page** containing reading second; Remove → cue list @ same second; split one-way sync on page change (VF-05…VF-09, VF-13)
- [x] 8.5 Widget/unit tests VF-01…VF-13 under paged column + end-second text; remove/rewrite infinite-timeline / timestamp tests
- [x] 8.6 Update meta skill `video-frames.md` (+ related reader/pages notes) под столбец×3 + end-second; cleanup мёртвого infinite-list API если не referenced
- [x] 8.7 `flutter analyze` + targeted `flutter test` (VF-* / LD-*); Rust: no new video/FRB (D1) / `cargo check` green
