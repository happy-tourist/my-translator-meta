## 1. Rust — rbook + parse EPUB

- [x] 1.1 Прочитать `design.md` (D1–D3, D8), delta `specs/parse/epub/spec.md`; skills `work-with-parse`, `work-with-config`, `server-work-with-structure`
- [x] 1.2 В sibling `my_translator_app/rust/Cargo.toml` добавить `rbook` по D1 (`default-features = false`, `threadsafe` + `prelude`); `cargo check` (host)
- [x] 1.3 Реализовать `rust/src/parse/epub/` (bytes → главы/абзацы; TOC/nested per D3; no-TOC = одна глава; strip per D9; fragment ignore) + unit-фикстуры; `cargo test` покрывает PE-01…PE-06
- [x] 1.4 Thin FRB `api/epub.rs`: locked `parse_epub` / `ParsedEpub` / `EpubChapterDto` / `EpubParseErrorKind` (D2); `pub mod epub` в `api/mod.rs`; `cargo check`
- [x] 1.5 `flutter_rust_bridge_codegen generate` из app; убедиться, что Dart API появился под `lib/src/rust/`; при необходимости web pkg build per `work-with-frb`
- [x] 1.6 `cargo check --target wasm32-unknown-unknown` (или каноничный web FRB check) — parse линкуется на wasm; при getrandom/`zip` сюрпризе применить митигацию D7 (не JS EPUB)

## 2. Library — import / open EPUB

- [x] 2.1 Прочитать delta `specs/library/documents/spec.md` + design D4/D6/D10; skills `work-with-drift`, `work-with-pages` / upload.md
- [x] 2.2 Расширить `document_format` / validation: `.epub` → format `epub`; обновить reject messages import + `attachTranslation` (LD-03, LD-24)
- [x] 2.3 **Binary-safe persist (D6 / LD-27):** Web store/load EPUB через base64 в `Documents.content` (format-gated); native sandbox без UTF-8 roundtrip; unit roundtrip bytes
- [x] 2.4 Open-flow: ветка format → `parse_epub` vs `parse_subtitles`; успех EPUB → Reader book-mode с chapter payload + `documentFormat` (LD-25 / D4); fail → Library + `EpubParseErrorKind` copy (LD-11, D10); не грузить video attachment для epub
- [x] 2.5 UI copy: `messageForEpubParseError` (или рядом с `parse_error_messages.dart`) — UnsupportedFormat|Malformed vs Empty (PE-05)
- [x] 2.6 Linked translation: для epub original грузить/парсить только epub (`parse_epub`); `attachTranslation` reject non-epub (LD-16/LD-26)
- [x] 2.7 Reading anchor: format-gated `lastReadStartMs` = chapter index для epub (D6 / LD-28); save/load API ветка; schema bump не нужен при reuse колонки
- [x] 2.8 Тесты Library/repo: LD-03, LD-24…LD-28 (+ regression subtitle import/Web UTF-8); `flutter test` targeted

## 3. Reader — book mode UI

- [x] 3.1 Прочитать delta `specs/reader/chapters/spec.md`; skills `work-with-reader`, `work-with-pages` / reader.md, `client-work-with-structure`
- [x] 3.2 Расширить `ReaderScreen` session: book-mode params (`ParsedEpub`/chapters + format); не пушить epub как пустой `originalCues` (D4)
- [x] 3.3 Режим книги — список абзацев текущей главы (RC-01); без timed cue chrome (`showTimestamp: false`)
- [x] 3.4 Нижний chapter pager `[<] n/N [>]` (паттерн video-frames chrome); N = число глав (RC-02, RC-03); смена главы сбрасывает scroll абзацев к началу
- [x] 3.5 Tap / highlight / vocab / translate на абзацах через `tokenize_cues(texts: paragraphs)` + token hit-targets (D4) — RC-04
- [x] 3.6 Скрыть Load/Replace/Remove video для epub (RC-05)
- [x] 3.7 Restore chapter из якоря (RC-06 / LD-28); persist chapter index on settle/dispose (не `nearestCueIndex`)
- [x] 3.8 Widget-тесты RC-01…RC-06; `flutter test` targeted

## 4. Split — EPUB↔EPUB + chapter sync

- [x] 4.1 Прочитать delta `specs/reader/split/spec.md`; skill `work-with-reader` / split-scroll.md
- [x] 4.2 Load translation: epub original → только `.epub` + `parse_epub` + book-режим снизу (RS-13); reject subtitle (RS-08 / LD-26); EPUB parse errors → D10 copy
- [x] 4.3 Общий chapter index: pager любой book-панели двигает другую (RS-14, RS-15); при неравном N — clamp + заметная ошибка на load (D5 / RS-16); subtitle time-sync не ломать (RS-05/06)
- [x] 4.4 Delete / restore linked epub translation в book-режиме (RS-07, RS-09)
- [x] 4.5 Widget-тесты RS-13…RS-16 (+ regression RS-04/05 / RS-12 epub branch); `flutter test` targeted

## 5. Verify

- [x] 5.1 `cargo test` в `rust/` (PE-*) зелёный
- [x] 5.2 `flutter analyze` в app без новых ошибок по затронутым путям
- [x] 5.3 `flutter test` по library/reader/epub/split тестам change — зелёный (включая LD-27 Web binary roundtrip)
