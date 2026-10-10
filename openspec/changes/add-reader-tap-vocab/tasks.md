## 1. Rust tokenize + FRB

- [x] 1.1 Прочитать `design.md` (D1–D3), `specs/reader/tap/spec.md` (RT-11, RT-12) и skills `.agents/skills/server/work-with-tokenizematize`, `work-with-frb-api`, `server-work-with-structure`, `server-work-with-test`
- [x] 1.2 Добавить `panko = "0.2.0"` в `rust/Cargo.toml`; core `rust/src/tokenize/` (обёртка panko → char spans по design D2) + thin `rust/src/api/tokenize.rs` с locked surface `tokenize_cues(texts) -> Vec<CueTokensDto>` / `TokenDto` (D3); `mod tokenize` в `lib.rs` + `api/mod.rs` — `cargo check` в `rust/` успешен (native; WASM-путь без C-deps у panko)
- [x] 1.3 Unit-тесты RT-11 / RT-12 (`cargo test` в `rust/`): кириллица, `don't`, invariant surface == cue[char_start..char_end]; входные строки как display join `text_lines` через `\n`
- [x] 1.4 Запустить `flutter_rust_bridge_codegen generate` из sibling app; generated Dart API `tokenizeCues` / DTOs доступен; не править `frb_generated` / `lib/src/rust/**` вручную

## 2. Flutter DI + batch tokenize + selection paint

- [x] 2.1 Прочитать `design.md` (D4–D9), `specs/reader/tap/spec.md` (RT-01…RT-04, RT-08…RT-10, RT-13) и skills `.agents/skills/client/work-with-reader` (+ `tap.md` — поведение этого change приоритетнее target «word card»), `work-with-pages`, `client-work-with-structure`, `work-with-frb`
- [x] 2.2 DI (D8): `AppShell` → `LibraryScreen(dictionaryRepository:)` → `ReaderScreen(dictionaryRepository:)` (required, рядом с `documentRepository`); обновить все call sites / pumps
- [x] 2.3 Test seam (D9): optional `tokenizeCues` / `TokenizeCuesFn` на `ReaderScreen` (default = generated FRB); Keys из D9
- [x] 2.4 Batch on open (D4, RT-01): `texts = originalCues.map((c) => c.textLines.join('\n'))`, один вызов; кэш в state; rebuild без повторного tokenize; in-flight = plain text + taps off; FRB/call failure → plain + `SnackBar` `readerTokenizeErrorSnack` (RT-13)
- [x] 2.5 UI оригинала: TextSpan/hit-test из кэша; Keys `readerCueToken_${cueIndex}_${tokenIndex}`; state machine D5 (expand / shrink-to-single / gap reset+~300ms flash / other cue) — RT-02…RT-04, RT-09; phrase = cue substring first..last (RT-10); translation pane без token selection (RT-08); без конкурирующего cue-level tap

## 3. Bottom CTA + sheet → Drift

- [x] 3.1 Прочитать `specs/reader/tap/spec.md` (RT-05…RT-07), `specs/dictionary/entries/spec.md` (DE-12, DE-13), `design.md` (D6) и skills `work-with-drift`, `work-with-dictionary` (persist boundary), `client-work-with-errors`
- [x] 3.2 CTA `readerAddVocabCta` («Add to vocabulary») при non-empty selection; sheet Keys D6/D9: surface read-only, translation required, context = cue text editable (RT-05)
- [x] 3.3 Save через injected `dictionaryRepository.add` (normalize + upsert); пустой перевод → `readerAddVocabValidationError` без persist (RT-07); успех → запись видна в Vocab path (RT-06, DE-12); повтор того же ключа — upsert (DE-13); после save очистить selection / закрыть sheet; **без** highlight refresh

## 4. Tests + analyze

- [x] 4.1 Flutter widget/unit-тесты по SC-ID: RT-01…RT-10, RT-13 (fake `tokenizeCues` + `DictionaryRepository`) и DE-12 / DE-13 — `flutter test` зелёный (skill `client-work-with-test`); обновить существующие Reader/Library pumps под новый ctor
- [x] 4.2 `flutter analyze` в корне app без ошибок по затронутому коду; при необходимости повторный `cargo test` в `rust/` после правок
