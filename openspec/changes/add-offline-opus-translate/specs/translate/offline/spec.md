# translate/offline Specification

## Purpose

Офлайн EN→RU перевод слова или фразы по запросу UI (без обязательной сети после установки сборки с моделью в assets). Persist словаря — `dictionary/entries`; точки вызова UI — `dictionary/entries` (Vocab) и `reader/tap` (Add/Edit sheets).

Native compute: Rust FRB (`init_offline_translate` + `translate_text`); Web compute: Transformers.js за тем же Dart-фасадом `lib/data/translate/` (design D1/D3/D4; Web **не** вызывает FRB translate). Ошибки на обеих платформах мапятся на одни kinds для UI (`OfflineTranslateErrorKind` / FRB `TranslateErrorKind`).

## ADDED Requirements

### Requirement: Offline EN-RU gloss for word or phrase
Система SHALL по запросу UI принимать непустой текст (слово или фраза, surface как показан пользователю) и возвращать один primary gloss на русском для пары EN→RU. Система MUST NOT требовать API-ключ или сеть для успешного перевода, когда модель уже доступна в поставке приложения. Система MUST NOT автоматически записывать результат в словарь — сохранение остаётся явным действием пользователя.

На native путь система MUST экспортировать thin FRB `translate_text(text, source_lang, target_lang) -> Result<TranslationResult, TranslateErrorKind>`, где `TranslationResult.primary_gloss` — единственный gloss; слово и фраза — один и тот же вызов (без пословной склейки на UI).

#### Scenario [TO-01]: Translate English word to Russian
- **GIVEN** offline translate engine готов
- **WHEN** UI запрашивает перевод непустого английского слова (например «hello») с парой en→ru
- **THEN** возвращается `Ok` с непустым `primary_gloss` на русском

#### Scenario [TO-02]: Translate English phrase to Russian
- **GIVEN** offline translate engine готов
- **WHEN** UI запрашивает перевод непустой английской фразы из нескольких слов с парой en→ru
- **THEN** возвращается `Ok` с одним `primary_gloss` для фразы целиком (не склейка пословных переводов на стороне UI)

#### Scenario [TO-03]: Empty query rejected
- **GIVEN** offline translate доступен
- **WHEN** UI запрашивает перевод пустой или whitespace-only строки
- **THEN** перевод не выполняется; возвращается `TranslateErrorKind::EmptyQuery` (не успешный пустой gloss)

### Requirement: Engine readiness and failure signaling
Система SHALL сообщать UI типизированной ошибкой, когда движок недоступен или инференс не удался, так чтобы UI мог показать user-visible ошибку и не вставлять ложный перевод. Пока идёт первая инициализация модели из assets, система MAY сообщать состояние загрузки (facade/button loading). Поддерживаемая пара в этом change — только EN→RU; запрос иной пары MUST давать `UnsupportedLanguagePair` (коды case-insensitive; нормализация к `"en"` / `"ru"`).

На native путь система MUST принимать model assets как `Vec<{name, bytes}>` через FRB `init_offline_translate` (Flutter читает AssetBundle); MUST NOT требовать OS-путь к модели в публичном API. Вызов `translate_text` до успешного init MUST возвращать `EngineUnavailable`. На `wasm32` публичные символы MAY быть stub с `EngineUnavailable` (Web UI не вызывает FRB translate).

#### Scenario [TO-04]: Engine unavailable
- **GIVEN** модель или рантайм недоступны (нет init / повреждённый asset / init fail)
- **WHEN** UI запрашивает перевод
- **THEN** возвращается `TranslateErrorKind::EngineUnavailable`; успешного gloss нет

#### Scenario [TO-05]: Inference failure
- **GIVEN** engine инициализирован
- **WHEN** инференс завершается сбоем
- **THEN** возвращается `TranslateErrorKind::InferenceFailed`; UI MUST NOT заполнять Translation успешным значением

#### Scenario [TO-06]: Unsupported language pair
- **GIVEN** engine готов
- **WHEN** UI запрашивает перевод с парой, отличной от EN→RU (после casefold нормализации)
- **THEN** возвращается `TranslateErrorKind::UnsupportedLanguagePair`

#### Scenario [TO-08]: Native init with asset bytes
- **GIVEN** Flutter передал полный набор named asset bytes из поставки
- **WHEN** вызывается `init_offline_translate`
- **THEN** при валидных файлах init успешен; при отсутствии обязательного файла или битых bytes — `EngineUnavailable` (предпочтительно); `Internal` только если сбой не классифицируется как assets/init

### Requirement: Offline after install
После установки/загрузки сборки, включающей model assets, успешный перевод EN→RU MUST быть возможен без доступа к сети (проверка: airplane mode / offline после старта приложения с уже доступными assets).

#### Scenario [TO-07]: Translate works offline after install
- **GIVEN** приложение запущено со встроенными model assets и engine успешно инициализирован
- **WHEN** сеть недоступна и UI запрашивает перевод непустого EN текста
- **THEN** primary gloss возвращается успешно

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| TO-01 | Translate English word to Russian | Rust `cargo test` (fixture/fake или real engine) +/или injectable Flutter seam |
| TO-02 | Translate English phrase to Russian | Rust `cargo test` phrase path |
| TO-03 | Empty query rejected | Rust `cargo test` EmptyQuery |
| TO-04 | Engine unavailable | Rust `cargo test` EngineUnavailable (no init / bad assets) |
| TO-05 | Inference failure | Rust `cargo test` InferenceFailed (fake engine) +/или Flutter seam |
| TO-06 | Unsupported language pair | Rust `cargo test` UnsupportedLanguagePair |
| TO-07 | Translate works offline after install | verify/ops (airplane) + design note; автотест — init without network mock where feasible |
| TO-08 | Native init with asset bytes | Rust `cargo test` init success / missing-file → EngineUnavailable |
