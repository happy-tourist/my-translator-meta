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

## 4–7. Prior presentation (DONE, superseded)

- [x] 4.x–7.x prior tasks (historical)

## 8. UX revision 2 — paged vertical triple + end-second text

- [x] 8.1–8.7 prior ship (column×3 + end-second; pager was top; timestamps off)

## 9. UX polish — bottom pager, scroll-to-first, cue times from file

- [x] 9.1 Прочитать revised proposal/design D4–D5 / specs VF-03/04/14/15; skill `video-frames.md`
- [x] 9.2 `VideoFramePages`: перенести pager (`[<] n/N [>]`) **под** список кадров (низ original pane) — VF-14
- [x] 9.3 При смене страницы (prev/next/goToPage) прокрутить список кадров к **первому** слоту (ScrollController / jumpTo 0) — VF-15
- [x] 9.4 Под кадром с текстом: `CueRow` / embed с `showTimestamp: true` (`startMs → endMs` из файла); пустой слот без метки — VF-03/VF-04
- [x] 9.5 Обновить widget-тесты VF-03/04/14/15; skill `video-frames.md`
- [x] 9.6 `flutter analyze` + targeted `flutter test` по video/reader tests

## 10. Durable video copy (Web reopen)

- [x] 10.1 Schema 6: `DocumentVideoMedia` (full video bytes + fileName); cascade on delete/detach
- [x] 10.2 `attachVideo` stores media bytes; Web persist `sandboxPath=drift-media` (not blob URL)
- [x] 10.3 Restore: `getVideoMedia` → session object URL / sandbox file via `playbackRefFromMediaBytes`
- [x] 10.4 Unit test media persist + detach clears; analyze/tests green
