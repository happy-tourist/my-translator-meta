## 1. Rust parse core

- [x] 1.1 Прочитать `design.md` (D1–D4, D6), `specs/parse/subtitles/spec.md` (PS-01…PS-07) и skill `.agents/skills/server/work-with-parse/SKILL.md`; entrypoints уже в D1: `srt::Parser::lossy`, `vtt::Parser::new` (built-in recovery), `ass::Parser::lossy` + Dialogue filter; в модуле `parse/` кратко сослаться на них
- [x] 1.2 Добавить `fasrt = "=0.5.0"` в `../my_translator_app/rust/Cargo.toml` (default `std`); `cargo check` в `rust/` и (по возможности) `cargo check --target wasm32-unknown-unknown`; rustc ≥ 1.95
- [x] 1.3 Создать `rust/src/parse/` (cue model `index` / `start_ms` / `end_ms` / `text_lines`, format detect+hint, `ParseError` kinds, map из fasrt через `to_duration`/`plain_text`, lossy policy, strict UTF-8 decode) по design D2–D4/D6; `mod parse` в `lib.rs`
- [x] 1.4 Unit-тесты с fixtures рядом с `parse/`: PS-01 SRT, PS-02 VTT, PS-03 ASS/SSA, PS-04 unsupported, PS-05 encoding (сырые invalid bytes), PS-06 empty, PS-07 lossy (SRT и/или ASS; VTT — recovery malformed timing) — `cargo test` в `rust/` зелёный с SC-ID в именах/комментариях тестов

## 2. FRB API + codegen

- [x] 2.1 Прочитать skills `.agents/skills/server/work-with-frb-api/SKILL.md` и `.agents/skills/client/work-with-frb/SKILL.md`; добавить тонкий `rust/src/api/parse.rs` + `pub mod parse` в `api/mod.rs` по D2 (`parse_subtitles(bytes, format_hint) → ParsedSubtitles` / error kind; `text_lines: Vec<String>`; без grammar в `api/`)
- [x] 2.2 Запустить `flutter_rust_bridge_codegen generate` из корня app; generated Dart под `lib/src/rust/` отражает parse API; `cargo check` + компиляция bindings без ручных правок generated

## 3. Flutter Reader + Library open

- [x] 3.1 Прочитать `specs/reader/cues/spec.md`, `specs/library/documents/spec.md` (LD-10/11), design D5–D6 и skills `work-with-pages` (`reader.md`, `upload.md`), `work-with-reader`, `client-work-with-structure`, `work-with-drift`
- [x] 3.2 Добавить read path: `DocumentContentStore.load` (+ Memory fake) и **`DocumentRepository` helper** `loadBytes(Document)` (native `storageKey` → sandbox; Web `content` → `utf8.encode`); schema Drift не менять
- [x] 3.3 Добавить Reader UI под `lib/features/reader/`: `ListView` cues, время `HH:MM:SS,mmm` start→end (D5), plain `text_lines`, scroll, AppBar title + back pop (RC-01…RC-04); без tokenize/word card
- [x] 3.4 В Library (`_onDocumentTap`): loading overlay → `loadBytes` → FRB `parse_subtitles(bytes, format_hint: doc.format)` → success `Navigator.push` Reader / fail SnackBar по таблице D5 без push (LD-10, LD-11, RC-05); убрать «Reader later»
- [x] 3.5 Widget-тесты: заменить LD-08 («Reader later») на LD-10/11; RC-01/RC-02 timestamps (`HH:MM:SS,mmm`), RC-03 scroll, RC-04 back, RC-05 no Reader on fail — `flutter test` зелёный
- [x] 3.6 `flutter analyze` в корне app без ошибок по затронутому коду
