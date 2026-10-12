## 1. Rust lemmatize + FRB contract

- [x] 1.1 Прочитать `design.md` (D1–D3, D7), delta `specs/reader/highlight/spec.md` (RH-09…RH-11), skills `work-with-tokenizematize` / lemmatize.md, `work-with-frb-api`, `server-work-with-structure`; сверить текущие `rust/src/tokenize/`, `rust/src/api/tokenize.rs`, `Cargo.toml`
- [x] 1.2 Добавить `wordnet-lemmatizer` (`default-features = false`) в `rust/Cargo.toml`; ввести модуль `lemmatize/` с D3 (`OnceLock<Lemmatizer>`, POS Noun→Verb→Adj, whitespace→casefold-only) — verify: `cargo check` в `rust/` проходит
- [x] 1.3 Расширить `TokenDto` полями `lemma_key` / `is_in_vocab`; обновить `tokenize_cues` (lemmatize после split); добавить `mark_vocab_tokens` и `lemma_keys_for_surfaces` (+ тонкий api export) по design D2 — verify: `cargo check` + типы совпадают с design
- [x] 1.4 Запустить `flutter_rust_bridge_codegen generate` из app; не править generated вручную — verify: generated Dart/`frb_generated` содержат новые поля, `markVocabTokens`, `lemmaKeysForSurfaces`
- [x] 1.5 Rust unit tests RH-10 / RH-11 (EN wordforms dogs/running → dog/run; Cyrillic/junk → casefold; whitespace phrase → casefold без WordNet) + span invariant регрессия RT-12 — verify: `cargo test` в `rust/` зелёный

## 2. Drift dual-key + migration

- [x] 2.1 Прочитать `design.md` D4 / D5.1, delta `specs/dictionary/entries/spec.md` (DE-14…DE-17), skills `work-with-drift`, `work-with-dictionary`; открыть `lib/data/database/tables.dart`, `app_database.dart`, `dictionary_repository.dart`
- [x] 2.2 Bump `schemaVersion` (3→4); в `onUpgrade` только DDL `surface_key`; post-open идемпотентный `migrateStubKeysIfNeeded` (FRB `lemmaKeysForSurfaces`) + merge по max `updated_at`; фразы (whitespace) → casefold identity — verify: unit/integration DE-15 / DE-16; вызов bootstrap сразу после `openAppDatabase` в `TranslationReaderApp` / shell
- [x] 2.3 Обновить `DictionaryRepository`: upsert/lookup по morph `lemma`; `surfaceKey` на add; `normalizeSurface` вместо stub-only identity; **repo не вызывает FRB** — morph key с call site (D5.1) — verify: unit DE-01 / DE-03 / DE-06 / DE-10 / DE-14 / DE-17
- [x] 2.4 Обновить Vocab UI: перед `add` прогон ввода через injected `LemmaKeysForSurfacesFn` — verify: vocab widget/unit + DE-сценарии зелёные или обновлены

## 3. Flutter Reader highlight + single-token tap/edit

- [x] 3.1 Прочитать `design.md` D5–D6 / D5.1, deltas `reader/highlight` + `reader/tap`, skills `work-with-reader` (tap/highlight), `work-with-styles`, `work-with-pages` / reader, `client-work-with-test`; открыть `reader_screen.dart`, `cue_row.dart`, tokenize seam
- [x] 3.2 `ReaderColors` ThemeExtension (light+dark green tint) на `MaterialApp` theme/darkTheme — verify: `cue_row` берёт `lemmaHighlight` из extension, нет inline hardcode
- [x] 3.3 Reader open: tokenize → Drift morph lemmas → `markVocabTokens` → cache; paint `is_in_vocab`; injectable `MarkVocabTokensFn` — verify: widget RH-01 / RH-02 / RH-08 / RH-09 / RH-12
- [x] 3.4 Single-token Add only (убрать contiguous expand/gap-flash); in_vocab → edit-sheet с locked Keys (design D5); refresh marks после Reader save/delete; Add save с `lemmaKey` + `surfaceKey` — verify: widget RT-02…RT-07 / RT-14 / RT-15 + RH-03…RH-07
- [x] 3.5 Обновить fakes: `TokenizeCuesFn` + `MarkVocabTokensFn` + `LemmaKeysForSurfacesFn` и тесты под новый `TokenDto` — verify: `flutter test` по затронутым reader/vocab/dictionary тестам зелёный

## 4. Analyze + cross-layer verification

- [x] 4.1 `flutter analyze` в app — verify: без новых ошибок в затронутых путях
- [x] 4.2 Повторить `cargo test` + целевой `flutter test` по Traceability (RH-*, RT-* из delta, DE-12…DE-17) — verify: зелёные
- [x] 4.3 Smoke WASM/size: `cargo check` для web target crate и/или `flutter build web` (или apk) после добавления `wordnet-lemmatizer` — verify: сборка проходит; размер не блокирует (см. design Risks)
