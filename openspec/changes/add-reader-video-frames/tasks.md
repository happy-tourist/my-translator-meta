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

## 4. Pure helpers и strip UI

- [x] 4.1 Прочитать `specs/reader/video-frames/spec.md` (VF-02…VF-05); `design.md` (D4, D5); skills `work-with-pages` / `reader.md`, `work-with-reader`
- [x] 4.2 Pure helpers: page→seconds math; cue text at second (покрытие `[startMs,endMs)`, join `textLines` через `\n`); unit-тесты VF-02/VF-04
- [x] 4.3 Виджет полосы: 3 слота, pager, счётчик, placeholder слота (VF-01…VF-03, VF-05, VF-10) с injectable extractor/cache

## 5. Врезка в Reader + sync

- [x] 5.1 Прочитать `design.md` (D4 remount, D6), VF-06…VF-09, VF-12; `reader/split` sync через `nearestCueIndex`; текущий `reader_screen.dart` AppBar/actions
- [x] 5.2 AppBar: load / remove video рядом со split; оркестрация pick → usable path → probe → attach → показать strip; remove → detach (VF-01, VF-09, VF-11)
- [x] 5.3 Body: `Column(strip?, single|split)`; strip **вне** remount-ветки Show/Hide; смена страницы → scroll original (+ translation если split); Show split → bottom к верху без сброса strip page; scroll cues не двигает strip (VF-06…VF-08)
- [x] 5.4 Open из Library: best-effort `getVideoAttachment` (fail не блокирует open) → Reader ctor или load по `originalDocumentId` (LD-19); re-pick при мёртвой ссылке без потери cache (LD-22)
- [x] 5.5 Widget-тесты VF-01, VF-03, VF-06, VF-07, VF-08, VF-09, VF-12 с fake extractor (без реального FFmpeg)

## 6. Verify

- [x] 6.1 `flutter analyze` в sibling app — без новых ошибок по затронутым файлам
- [x] 6.2 `flutter test` по добавленным/затронутым тестам (SC-ID из Traceability VF-* / LD-18…LD-23)
- [x] 6.3 Негативная проверка Rust: **нет** новых video/FFmpeg модулей в `rust/`, **нет** изменений публичного FRB API ради video, **нет** обязательного `flutter_rust_bridge_codegen generate` из‑за этого change (D1); `cargo check` в `rust/` остаётся зелёным без video deps
