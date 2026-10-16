## 1. Drift: корневые документы

- [x] 1.1 Прочитать `design.md` (D4), delta `specs/library/documents/spec.md` (LD-02/06/29/30); skill `work-with-drift` (schema + repositories) — **delta/design перекрывают** skill до 5.1 (nullable `folderId`, root import)
- [x] 1.2 Schema 7: `Documents.folderId` nullable FK; **table recreate / customStatement** (не addColumn); codegen Drift — verify: `schemaVersion == 7`, upgrade 6→7 сохраняет ряды, чистая DB создаётся
- [x] 1.3 `DocumentRepository`: разрешить `importIntoFolder(folderId: null)`; канон `listVisible({int? folderId})` (`null` → `isNull()`); `attachTranslation` копирует nullable `folderId` — verify: unit LD-02 (root ok), LD-29, LD-13 (root) в `test/data/document_repository_test.dart`

## 2. Library UI: import и смешанный список

- [x] 2.1 Прочитать `design.md` (D5), delta `specs/library/folders/spec.md` (LF-09/10/11); skills `work-with-pages` (`upload.md`), `client-work-with-structure` — **delta/design перекрывают** skill до 5.1 (import в корне, empty root Import+Create)
- [x] 2.2 `LibraryScreen`: убрать early-return import в корне; AppBar Import в корне; `_reload` root через `listVisible(null)`; empty root — Create folder + Import — verify: widget LF-09, LF-11, LD-02, LD-30 в `test/features/library_screen_test.dart`

## 3. Reader: one-shot sync субтитров / video

- [x] 3.1 Прочитать `design.md` (D1, D3), delta `specs/reader/split/spec.md` (RS-05/06/18), `specs/reader/video-frames/spec.md` (VF-06); skills `split-scroll.md`, `video-frames.md` — **delta/design перекрывают** realtime sync в skills до 5.1
- [x] 3.2 Убрать settle-sync из `_onScrollNotification`; оставить persist reading anchor на settle оригинала; убрать sync из `_restoreInitialReadingAnchor`; one-shot только Show + load перевода — verify: widget RS-06 (no sync), RS-05, RS-18
- [x] 3.3 `_onVideoPageChanged` без sync; убрать sync из `_loadVideo` / `_removeVideo`; Show/load перевода с video one-shot по текущей секунде — verify: widget VF-06, VF-07

## 4. Reader: независимые EPUB chapter indices

- [x] 4.1 Прочитать `design.md` (D2), delta `specs/reader/split/spec.md` (RS-14/15/19/20), `specs/reader/chapters/spec.md` (RC-02/10); skill `chapters.md` — **delta/design перекрывают** shared `_chapterIndex` sync в skill до 5.1
- [x] 4.2 Развести `_originalChapterIndex` / `_translationChapterIndex`; `_goToChapter(pane, …)` только своей панели; one-shot в book `_toggleSplit` (enabling) и EPUB `_loadTranslation`; resume/persist только original — verify: widget RS-14, RS-19, RS-20, RC-10 в `reader_epub_split_test` / book tests

## 5. Meta skills + проверка

- [x] 5.1 Обновить meta skills под one-shot sync и root docs: `split-scroll.md`, `video-frames.md`, `chapters.md`, `work-with-reader/SKILL.md`, `work-with-pages/upload.md`, `work-with-drift/schema.md` (schema 7 / nullable `folderId`), `work-with-drift/repositories.md` (`listVisible` / import null) — verify: тексты не требуют realtime RS-06/VF-06/RS-14 sync и не требуют reject LD-02 / «import только в папку»
- [x] 5.2 Из sibling `my_translator_app`: `flutter analyze` и `flutter test` (library + reader + document_repository, затронутые SC) — verify: analyze без errors, тесты зелёные
