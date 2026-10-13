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

> **Superseded (UX revision):** page-of-3 strip. 4.2–4.3 invalidated → см. §7. Cue-at-second helper остаётся полезным.

- [x] 4.1 Прочитать `specs/reader/video-frames/spec.md`; `design.md` (D4, D5); skills `work-with-pages` / `reader.md`, `work-with-reader`
- [x] 4.2 ~~Pure helpers: page→seconds math~~ → **переписать:** helpers clamp/jump `readingTimeMs → second`; unit VF-02/VF-04/VF-05 math (page-of-3 math не acceptance)
- [x] 4.3 ~~Виджет полосы 3 слота~~ → **заменить:** виджет/список second-timeline row (кадр + timestamp + текст|пусто + placeholder) с injectable extractor/cache (VF-01…VF-03, VF-10)

## 5. Врезка в Reader + sync

> **Superseded:** `Column(strip?, cues)`. Replace + jump + one-way split sync → §7 / unchecked ниже.

- [x] 5.1 Прочитать `design.md` (D4, D6), VF-*; текущий `reader_screen.dart`
- [x] 5.2 AppBar load/remove: оркестрация pick → usable path → probe → attach → **replace original body timeline** + **jump к текущему reading time**; remove → detach + **restore cue list @ same second** (VF-01, VF-05, VF-08, VF-11)
- [x] 5.3 Body: при video — timeline в original pane (не strip над cues); split — кадры только сверху; original scroll → sync bottom; bottom scroll ↛ original; Show split не сбрасывает second (VF-06, VF-07, VF-09)
- [x] 5.4 Open из Library: best-effort `getVideoAttachment` (fail не блокирует open) → Reader (LD-19); re-pick при мёртвой ссылке (LD-22) — **дополнить jump VF-13**
- [x] 5.5 Widget-тесты VF-01…VF-13 (убрать/переписать strip pager tests) с fake extractor

## 6. Verify

- [x] 6.1 `flutter analyze` в sibling app — без новых ошибок по затронутым файлам
- [x] 6.2 `flutter test` по добавленным/затронутым тестам (VF-* revision / LD-18…LD-23)
- [x] 6.3 Негативная проверка Rust: **нет** новых video/FFmpeg модулей в `rust/`, **нет** изменений публичного FRB API ради video (D1); `cargo check` в `rust/` зелёный

## 7. UX revision — interleaved seconds (новый scope после strip)

- [x] 7.1 Прочитать revised `proposal.md` / `design.md` D4–D6 / `specs/reader/video-frames/spec.md` (VF-01…VF-13); skill `work-with-reader/video-frames.md` — обновить канон strip→timeline при apply или в check-changes
- [x] 7.2 Убрать использование `VideoFrameStrip` 3-в-ряд из `reader_screen`; внедрить virtualized second list (reuse `ItemScrollController` / `ScrollablePositionedList` паттерн Reader)
- [x] 7.3 Persist якорь: `last_second` (переиспользовать/переименовать `lastFramePage`) + запись при скролле; restore jump VF-13
- [x] 7.4 Split listeners: при video отключить sync bottom→original; сохранить original→bottom
- [x] 7.5 Cleanup мёртвого page-of-3 UI/tests или оставить только если ещё referenced; обновить meta skill `video-frames.md` под interleaved
