## Context

См. `proposal.md` — Why / Scope. Сейчас перевод в Vocab и Reader Add/Edit вводится вручную; модуля `translate/` и FRB translate API нет. Канон skills утверждал «offline translate в Rust»; на Web FRB = WASM, а CT2/`trad` на Web нежизнеспособны. Explore зафиксировал dual runtime на одной модели OPUS-MT EN→RU ONNX.

Runtime: sibling `../my_translator_app` — Flutter UI + Rust FRB; user persist = Drift (без изменений схемы словаря в этом change).

## Goals / Non-Goals

**Goals:**

- Кнопка Translate → loading на кнопке → fill Translation на Vocab add, Reader Add-sheet, Reader Edit-sheet.
- Офлайн EN→RU после установки (модель в assets).
- Native: Rust ORT + tokenizer за thin FRB; Web: Transformers.js (worker) за тем же Dart-фасадом.
- Атрибуция OPUS-MT + disclaimer; правка meta-канона под web-исключение.

**Non-Goals:**

- Другие пары / докачка пакетов; AI/cloud; batch субтитров; settings languages; length-filter; отдельный download-экран.

## Decisions

### D1 — Dual runtime (A), одна модель

| Слой | Выбор | Альтернативы |
|------|--------|--------------|
| Модель | `onnx-community/opus-mt-en-ru` (uint8 ONNX + tokenizer), CC-BY 4.0 | NLLB/`trad` (~600MB, native-only) — отвергнуто |
| Native | `ort` (pyke) + Marian encode/decode loop + `tokenizers` (или эквивалент) в `rust/src/translate/`, thin `api/` | Dart-only `onnx_translation` — обходит FRB-канон на native |
| Web | `@huggingface/transformers` pipeline в `web/` worker + `dart:js_interop` | тот же ORT в WASM (`ort-web`) — выше риск сборки FRB+ORT; отложено |
| UI facade | один Dart API (injectable для тестов): `translateOffline(text, {source, target})` → platform impl | два разных вызова из виджетов — отвергнуто |

На Web compute translate **не** идёт через Rust WASM; на native — только через FRB. Документировать исключение в `work-with-translate`, app/meta `AGENTS.md`.

### D2 — Assets в поставке

При apply агент скачивает нужные файлы модели/токенизатора в app assets (и/или `web/` static для Transformers.js local files, если pipeline настроен на local). `pubspec.yaml` объявляет assets. Runtime v1 **без** HF download. Loading = init/инференс на кнопке.

Размер ~100MB+ — ожидать влияние на clone/CI; при необходимости позже LFS/fetch-скрипт (не блокер v1, если бинарники в репо app).

**Имена файлов:** точный набор (encoder/decoder uint8 vs merged, `tokenizer.json` vs SPM) фиксируется в spike (см. tasks §2.0) одной заметкой рядом с assets; native и Web читают **один и тот же** набор имён из этой заметки.

### D3 — FRB surface (native) — locked contract

Аналог загрузки: как `parse_subtitles(bytes)` — **Flutter читает asset bytes**, Rust не открывает `AssetBundle` и не принимает OS-пути модели наружу.

Модуль: `rust/src/api/translate.rs` + `pub mod translate` в `api/mod.rs`; домен в `rust/src/translate/`.

#### Типы (FFI-friendly)

```text
TranslationResult {
  primary_gloss: String   // непустой при Ok; один gloss на запрос (слово или фраза)
}

TranslateErrorKind {
  EmptyQuery,                 // trim-empty / whitespace-only
  UnsupportedLanguagePair,    // не en→ru после нормализации
  EngineUnavailable,          // нет init / повреждённые assets / ORT session fail
  InferenceFailed,            // session жив, инференс упал (TO-05)
  Internal,                   // неожиданное; safe message наружу
}

OfflineModelAsset {
  name: String,               // basename из asset-заметки, напр. "encoder_model_quantized.onnx"
  bytes: Vec<u8>,
}
```

Языковые коды на FRB: case-insensitive; нормализация в Rust к lower ASCII (`"EN"` → `"en"`). Поддерживается только `source_lang == "en"` && `target_lang == "ru"`.

#### Exports

| Export | Сигнатура (логика) | Когда |
|--------|-------------------|--------|
| `init_offline_translate` | `assets: Vec<OfflineModelAsset> -> Result<(), TranslateErrorKind>` | Один раз (или повтор после fail) **до** успешного `translate_text`; тяжёлый I/O **не** в `#[frb(init)] init_app` |
| `translate_text` | `text, source_lang, target_lang -> Result<TranslationResult, TranslateErrorKind>` | Слово **или** фраза одним вызовом (surface as displayed; без lemma-пересчёта) |
| `is_offline_translate_ready` (optional) | `() -> bool` | MAY для UI; иначе facade трактует `EngineUnavailable` |

Правила:

- Внутренний trait `OfflineTranslator` в `rust/src/translate/`; `api/` только map args ↔ domain.
- Единый `translate_text` (не отдельные `translate_word` / `translate_phrase` в этом change) — MT принимает одну строку.
- `NotFound` из старого skill-канона **не** используется для OPUS-MT (нет словарного miss; либо gloss, либо ошибка).
- Пути к файлам модели **не** в публичном FRB; только `name` + `bytes`.
- Codegen после `api/`; generated не править руками.
- Web facade **не** вызывает эти FRB на web.

#### cfg / WASM

В `rust/Cargo.toml`: deps `ort`, `tokenizers` (и пр. ORT-only) — **только** non-wasm, например:

```toml
[target.'cfg(not(target_arch = "wasm32"))'.dependencies]
ort = "…"
tokenizers = "…"
```

На `wasm32`: те же публичные символы `api/translate` **компилируются как stub** → `EngineUnavailable` (или no-op ready=false). Цель: `flutter build web` / FRB WASM **не** линкует native ORT. Реальная логика — `#[cfg(not(target_arch = "wasm32"))]`.

### D4 — Flutter точки врезки (locked)

#### D4.1 — Facade placement + API

| Каталог / файл | Роль |
|----------------|------|
| `lib/data/translate/offline_translate.dart` | Публичный Dart API + `typedef OfflineTranslateFn` + result/error types для UI |
| `lib/data/translate/offline_translate_native.dart` | Native: `rootBundle` named assets → lazy `init_offline_translate` → `translate_text` |
| `lib/data/translate/offline_translate_web.dart` | Web: `dart:js_interop` → Transformers.js worker; **не** вызывает FRB translate |
| `lib/data/translate/offline_translate_platform.dart` | Conditional export: `io` vs `html` (аналог `document_content_store_platform.dart`) |

Сигнатура фасада (логика; имена полей могут mirror FRB):

```text
typedef OfflineTranslateFn = Future<OfflineTranslateOutcome> Function({
  required String text,
  String sourceLang,   // default "en"
  String targetLang,   // default "ru"
});

OfflineTranslateOutcome {
  Ok { primaryGloss: String }   // non-empty
  Err { kind: OfflineTranslateErrorKind }
}

OfflineTranslateErrorKind {
  emptyQuery | unsupportedLanguagePair | engineUnavailable
  | inferenceFailed | internal
}
```

Правила:

- Один `OfflineTranslateFn` на все три UI surface; production default = platform factory из `offline_translate_platform.dart`.
- Lazy init **внутри** native impl на первом вызове (bytes → `init_offline_translate`); UI не зовёт init отдельно.
- На Web facade **никогда** не импортирует/не вызывает generated `init_offline_translate` / `translate_text`.
- Widget tests: hand-written fake `OfflineTranslateFn` через ctor injection — **без** mockito.

#### D4.2 — Injection seams (как lemmaKeys / parseSubtitles)

| Виджет | Как получает facade |
|--------|---------------------|
| `VocabularyScreen` | optional ctor `OfflineTranslateFn? offlineTranslate`; default → platform |
| `ReaderScreen` | optional ctor `OfflineTranslateFn? offlineTranslate`; прокидывает в sheets |
| `_AddVocabSheet` / `_EditVocabSheet` | required (или optional+default) `OfflineTranslateFn` от `ReaderScreen` |
| Shell → Vocab | прокинуть тот же optional, если shell уже прокидывает `lemmaKeysForSurfaces` |

`_AddEntryForm` сейчас `StatelessWidget` — для loading/disable Translate либо поднять state в `_VocabularyScreenState`, либо сделать form `StatefulWidget`. Не вызывать FRB/JS из `build()`.

#### D4.3 — UI placement + disable rules

| Место | Файл | Поведение |
|-------|------|-----------|
| Vocab add | `vocabulary_screen.dart` (`vocabAddForm`) | Translate **рядом с** Lemma (`vocabLemmaField`); loading на кнопке; fill `vocabTranslationField` |
| Reader Add | `_AddVocabSheet` | Translate **рядом с** Word (`readerAddVocabSurface`); query = `surface` as displayed |
| Reader Edit | `_EditVocabSheet` | то же у Word/surface (`readerEditVocabSurface`); query = displayed surface |

Disable:

| Surface | Disabled when | After success/error |
|---------|---------------|---------------------|
| Vocab Translate | lemma trim-empty **или** in-flight loading | снова enabled (можно повторить) |
| Reader Add/Edit Translate | in-flight **или** уже был старт Translate на этом открытии sheet | **остаётся disabled до закрытия sheet** (Add и Edit одинаково) |
| Save / Add entry | **не** блокируется Translate (только своим `_saving` / submit) | — |
| Translation field | остаётся editable; Translate overwrite без confirm | на ошибке **не** менять текст поля |

Loading: `CircularProgressIndicator` **на** Translate button (child), `onPressed: null` на время запроса — покрывает lazy init + inference.

#### D4.4 — Widget Keys (для `flutter test`)

| Key | Где |
|-----|-----|
| `vocabTranslateButton` | Vocab Translate |
| `vocabTranslateAttribution` | блок атрибуции+disclaimer на Vocab form |
| `vocabTranslateErrorSnack` | SnackBar ошибки Translate на Vocab |
| `readerAddVocabTranslate` | Add-sheet Translate |
| `readerAddVocabTranslateAttribution` | атрибуция+disclaimer Add-sheet |
| `readerAddVocabTranslateErrorSnack` | SnackBar ошибки Translate Add-sheet |
| `readerEditVocabTranslate` | Edit-sheet Translate |
| `readerEditVocabTranslateAttribution` | атрибуция+disclaimer Edit-sheet |
| `readerEditVocabTranslateErrorSnack` | SnackBar ошибки Translate Edit-sheet |

Существующие Keys полей (`vocabLemmaField`, `vocabTranslationField`, `readerAddVocabTranslation`, `readerEditVocabTranslation`, …) **не** переименовывать.

#### D4.5 — Error UX + copy

Transient translate errors → **SnackBar** с Key из D4.4 (паттерн `libraryErrorSnack` / `readerTokenizeErrorSnack`), **не** inline validation slot (`vocabFormError` / `reader*ValidationError` — те только для Drift/empty Translation).

| Kind | User-visible (EN, locked v1) |
|------|------------------------------|
| `emptyQuery` | `Enter a word or phrase to translate` |
| `unsupportedLanguagePair` | `Only English → Russian is supported` |
| `engineUnavailable` | `Translation unavailable. Try again later` |
| `inferenceFailed` | `Couldn't translate. Try again` |
| `internal` | `Couldn't translate. Try again` |

После `await`: check `mounted` / sheet still open before `setState` / SnackBar. **Не** подставлять gloss при Err. Panic/Anyhow с bridge → generic `Couldn't translate. Try again` + `debugPrint`.

#### D4.6 — Attribution / disclaimer copy (locked v1)

Один текст на все три surface (внутри Key `*TranslateAttribution`):

- Attribution: `OPUS-MT (Helsinki-NLP), CC-BY 4.0`
- Disclaimer: `Translation may be inaccurate`

Достаточно `find.text` / Key на контейнер для DE-22 / RT-22 / RT-23.

### D5 — Meta docs в том же change

Обновить `work-with-translate` (+ при необходимости app/meta AGENTS / `openspec/config.yaml` context): engine = OPUS-MT en→ru ONNX; Web = Transformers.js exception; native = Rust ORT; FRB = `init_offline_translate` + `translate_text` + kinds выше. Client skills (`client-work-with-structure`, `work-with-pages`, `client-work-with-errors`, `work-with-reader`) — web-exception + D4 SnackBar/Keys pointer; убрать «offline translate = Later Rust». Не оставлять «engine TBD» для этого продукта после merge. Reader delta SC-ID: **RT-18…RT-25** (после main RT-17).

### Prerequisites (из explore — закрыты)

| ID | Решение |
|----|---------|
| Engine | OPUS-MT ONNX, не trad |
| Pair | EN→RU |
| Platforms | Web + native via dual |
| Packaging | assets при apply; native load = Flutter bytes → FRB init |
| UI surfaces | Vocab + Add + Edit |
| Query text | surface as displayed |
| Loading | on button (covers lazy init + inference) |
| Trust copy | attribution + «может быть неточным» |

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Dual engines расходятся по качеству/ошибкам | Один facade + одинаковые error kinds; fixture-тесты на контракт; spot-check одних фраз |
| ORT + Marian loop в Rust сложнее Transformers.js pipeline | Spike early (tasks 2.0) с AC; при блоке — update change (native stub + Web-first) до продолжения native path |
| Transformers.js + COOP/COEP / worker + Flutter web | Worker в `web/`; проверить с канон `flutter run` port 5555 + headers |
| Большие assets в git | Accept v1; note LFS later |
| FRB WASM build не должен линковать тяжёлый ORT | target-conditional Cargo deps + wasm stubs (D3) |

## Migration Plan

- Нет миграции Drift schema.
- Новые deps: Cargo (`ort`, tokenizer — non-wasm), Flutter (js interop / web assets), npm или vendored transformers для web worker — зафиксировать версии в apply notes / lockfiles.
- Rollback: убрать кнопку и facade; assets можно оставить без вызова.

## Open Questions (закрываются spike, не product-human)

- Точный набор ONNX-файлов (encoder/decoder uint8 vs merged) — **spike 2.0**; записать в asset-заметку.
- `tokenizer.json` vs SPM для native — **spike 2.0**; тот же список имён для Web local files.

Чеклист реализации — `tasks.md`.
