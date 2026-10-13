## 1. Assets и канон meta

- [x] 1.1 Прочитать `design.md` (D2 assets, D3 FRB, D5 docs) и `proposal.md` Scope / Out of scope
- [x] 1.2 Скачать в sibling app необходимые файлы OPUS-MT en→ru ONNX + tokenizer из `onnx-community/opus-mt-en-ru` (**имена — после spike 2.0**); объявить в `pubspec.yaml` assets; зафиксировать список файлов в краткой заметке рядом с assets или в app README fragment
- [x] 1.3 Обновить meta: `.agents/skills/server/work-with-translate/SKILL.md` + client (`client-work-with-structure`, `work-with-pages`, `client-work-with-errors`, `work-with-reader`) и при необходимости `AGENTS.md` / app `AGENTS.md` / `openspec/config.yaml` context — engine OPUS-MT EN→RU; Web Transformers.js exception; native Rust ORT; FRB + D4 facade/Keys/SnackBar (dual из design). **Примечание align-meta:** skills + config context уже правлены в working tree — при apply сверить и добить только app `AGENTS.md`, если там ещё «весь translate в Rust»

## 2. Spike native ORT (early gate)

- [x] 2.0 Spike в `rust/` (можно throwaway/bin или `#[cfg(test)]`): загрузить выбранные ONNX + tokenizer, прогнать Marian encode/decode на 1–2 фразах («hello», короткая фраза) через `ort` **без** полного Flutter UI
  - **AC (выход):** (a) зафиксирован список basename файлов для assets; (b) `cargo test` или spike binary печатает непустой RU gloss offline; (c) заметка «что линковать в Cargo» (версии `ort` / tokenizer); (d) если AC недостижим за разумный spike — **остановить native path**, сделать `openspec-update-change` (Web-first / native stub) до задач 2.2+
  - Не коммитить огромные модели в meta; только в app assets после успешного spike
  - **Spike result:** basenames `tokenizer.json`, `encoder_model_uint8.onnx`, `decoder_model_uint8.onnx`; Cargo `ort = "2.0.0-rc.14"`, `tokenizers` 0.21 (`default-features=false`, `fancy-regex`), `ndarray = "0.17"` (non-wasm); notes in `my_translator_app/rust/src/translate/SPIKE.md` + `assets/models/opus-mt-en-ru/README.md`; run `cargo test spike_opus_mt_en_ru_hello_offline -- --nocapture` → `Привет.` / `Доброе утро`

## 3. Rust translate + FRB (native)

- [x] 3.1 Прочитать skills `work-with-translate`, `work-with-frb-api`, `server-work-with-structure`, `server-work-with-errors`; delta `specs/translate/offline/spec.md` (TO-01…TO-06, TO-08)
- [x] 3.2 Cargo: `[target.'cfg(not(target_arch = "wasm32"))'.dependencies]` для `ort` / tokenizer; модуль `rust/src/translate/` с `OfflineTranslator` + `TranslateErrorKind` { EmptyQuery, UnsupportedLanguagePair, EngineUnavailable, InferenceFailed, Internal }; на `wasm32` — stub тех же публичных API → EngineUnavailable
- [x] 3.3 Thin FRB в `rust/src/api/translate.rs`: `init_offline_translate(assets: Vec<OfflineModelAsset>)`, `translate_text(text, source_lang, target_lang) -> Result<TranslationResult, TranslateErrorKind>` (`TranslationResult.primary_gloss`); optional `is_offline_translate_ready`; `pub mod translate` в `api/mod.rs`; codegen `flutter_rust_bridge_codegen generate`; не править generated вручную; **не** класть тяжёлый load в `init_app`
- [x] 3.4 `cargo test` в `rust/` на TO-01…TO-06 и TO-08 (fixture/fake engine допустим для EmptyQuery / Unsupported / Unavailable / InferenceFailed; real path — где стабильно после spike)

## 4. Web Transformers.js путь

- [x] 4.1 Прочитать design D1 (Web worker) и skill `work-with-env-deploy` (COOP/COEP, port 5555)
- [x] 4.2 Добавить web worker + pipeline OPUS-MT en→ru (локальные assets / зафиксированная версия `@huggingface/transformers`); bridge через `dart:js_interop` к Dart facade; те же логические error kinds, что native
- [x] 4.3 Убедиться, что web-сборка app собирается (`flutter build web` или эквивалент из env-deploy) без линковки native-only ORT в WASM FRB

## 5. Dart facade и UI

- [x] 5.1 Прочитать design **D4** (facade path, Keys, disable, error copy) + skills `client-work-with-structure`, `work-with-pages`, `work-with-reader`, `client-work-with-errors`; delta specs `dictionary/entries` (DE-19…22), `reader/tap` (RT-18…25)
- [x] 5.2 Реализовать `lib/data/translate/`: `offline_translate.dart` (`OfflineTranslateFn` + outcome/kinds) + `offline_translate_native.dart` (AssetBundle → lazy `init_offline_translate` → `translate_text`) + `offline_translate_web.dart` (JS worker, **без** FRB translate) + `offline_translate_platform.dart` (conditional `io`/`html`, аналог content-store)
- [x] 5.3 Vocab: inject `OfflineTranslateFn?` в `VocabularyScreen`; `vocabTranslateButton` у lemma; loading на кнопке; fill `vocabTranslationField`; disable empty/loading; re-enable после; SnackBar `vocabTranslateErrorSnack`; `vocabTranslateAttribution` (DE-19…22); Add не блокировать Translate
- [x] 5.4 Reader: inject `OfflineTranslateFn?` в `ReaderScreen` → `_AddVocabSheet` / `_EditVocabSheet`; Keys `readerAddVocabTranslate` / `readerEditVocabTranslate` (+ Attribution / ErrorSnack); disable до закрытия sheet после старта (Add **и** Edit, RT-19/RT-24); fill Translation; Save не блокировать Translate; Edit error SnackBar (RT-18…25)
- [x] 5.5 `flutter test` на DE-19…22 и RT-18…25 (hand-written fake `OfflineTranslateFn`, Key finders); `flutter analyze`

## 6. Проверка слоёв

- [x] 6.1 `cargo test` / `cargo check` в `rust/` после финальных правок translate (в т.ч. что wasm/host cfg не тянет ORT в web build)
- [x] 6.2 `flutter analyze` и `flutter test` из корня sibling app по затронутым тестам
