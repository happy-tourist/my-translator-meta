# dictionary/entries Specification

## Purpose

Хранит локальный личный словарь пользователя (лемма, перевод, опциональный контекст) для проверки persistence и матчинга подсветки в Reader. Source of truth записей — Flutter Drift в `lib/`; Rust/FRB storage API для словаря не требуется; compute/NLP (lemma highlight) — `reader/highlight`.

## Requirements

### Requirement: Add dictionary entry
Система SHALL позволять добавить запись словаря с unique identity `lemma` и переводом через **Vocab UI** или **Reader** Add-sheet. Контекст MAY быть пустым. MUST сохраняться **surface_key** (casefold исходного surface / phrase join). Identity:
- **single-word** (нет whitespace в ключе): morph lemma (EN WordNet; morph-fail → casefold(surface));
- **multi-word / phrase** (whitespace в ключе): casefold(join) без WordNet.
Reader MAY добавлять и single, и phrase (contiguous selection). Vocab manual add: single через `lemmaKeysForSurfaces`; multi-word input → casefold identity без morph cascade.

#### Scenario [DE-01]: Add entry with context
- **GIVEN** словарь пуст
- **WHEN** пользователь добавляет morph-лемму «run», перевод «бежать» и контекст «He was running»
- **THEN** запись доступна в списке с этими полями

#### Scenario [DE-02]: Add entry without context
- **GIVEN** словарь может содержать другие записи
- **WHEN** пользователь добавляет «cat» / «кот» без контекста
- **THEN** запись сохраняется; context пуст или отсутствует

#### Scenario [DE-09]: Reject empty lemma or translation
- **GIVEN** форма добавления открыта
- **WHEN** подтверждение с пустой леммой или пустым переводом после trim
- **THEN** запись не создаётся; ошибка валидации

#### Scenario [DE-11]: Empty vocabulary CTA
- **GIVEN** словарь пуст
- **WHEN** открыт экран словаря
- **THEN** empty state с возможностью добавить запись на экране

#### Scenario [DE-12]: Add entry from Reader selection
- **GIVEN** в Reader non-vocab selection (single или contiguous phrase) и открыт Add-sheet
- **WHEN** непустой перевод и подтверждение
- **THEN** запись с корректным identity (morph или casefold join) доступна в словаре

#### Scenario [DE-14]: Surface key stored for token
- **GIVEN** сохранение single surface «Running» (morph «run»)
- **WHEN** запись создана
- **THEN** lemma = «run», surface_key = casefold surface («running»)

#### Scenario [DE-17]: Morph-fail key is casefold surface
- **GIVEN** single слово без отличной EN morph-леммы
- **WHEN** запись сохраняется
- **THEN** lemma identity = casefold(surface)

#### Scenario [DE-18]: Phrase key is casefold join
- **GIVEN** Reader contiguous selection из токенов «Hello» и «world»
- **WHEN** запись сохраняется из Add-sheet
- **THEN** lemma (и surface_key) = casefold(join) («hello world»), без WordNet на фразе целиком

### Requirement: Duplicate lemma policy
Система SHALL при повторном добавлении того же **lemma key** (morph или phrase) выполнять upsert translation/context без второго ряда. Политика едина для Vocab UI, Reader Add, Reader edit.

#### Scenario [DE-03]: Second add of same lemma
- **GIVEN** есть «run»
- **WHEN** снова добавляют «run» с другим переводом
- **THEN** одна запись с обновлёнными полями

#### Scenario [DE-10]: Case-insensitive lemma identity
- **GIVEN** есть «run»
- **WHEN** добавляют «Run»
- **THEN** upsert одной записи

### Requirement: Reader add uses shared upsert policy
Добавление из Reader MUST использовать ту же upsert-политику по `lemma`, что и Vocab UI.

#### Scenario [DE-13]: Upsert from Reader same as Vocab
- **GIVEN** запись с ключом «hello» (или phrase key)
- **WHEN** Reader сохраняет тот же ключ с другим translation/context
- **THEN** одна запись; поля обновлены

### Requirement: Lookup by lemma
Система SHALL поддерживать lookup по `lemma` (case-insensitive). Membership для highlight: morph keys и phrase keys — см. `reader/highlight`.

#### Scenario [DE-06]: Lookup hit
- **GIVEN** сохранена «run»
- **WHEN** lookup «run» / «Run»
- **THEN** hit с данными записи

#### Scenario [DE-07]: Lookup miss
- **GIVEN** «xyz» нет
- **WHEN** lookup «xyz»
- **THEN** miss

### Requirement: One-time stub-to-morph schema migration
При schema migration (один раз) система SHALL перевести stub-ключи в dual-key: `lemma` = morph для singles; multi-word → casefold phrase identity + `surface_key`. Коллизии singles: merge по max `updated_at`. Phrase rows MUST сохраниться. (Legacy cleanup — вне UI.)

#### Scenario [DE-15]: Schema migration merges collisions
- **GIVEN** две stub-записи «running» и «ran» с разными updated_at
- **WHEN** одноразовая migration
- **THEN** одна запись morph «run»; поля от max updated_at

#### Scenario [DE-16]: Phrase rows survive migration
- **GIVEN** multi-word stub-ключ до migration
- **WHEN** schema migration
- **THEN** фразовый ряд остаётся; участвует в phrase highlight membership (`reader/highlight`)

### Requirement: List dictionary entries
Система SHALL отдавать список всех сохранённых записей словаря.

#### Scenario [DE-04]: List after several adds
- **GIVEN** добавлены минимум две разные леммы
- **WHEN** пользователь открывает словарь
- **THEN** обе записи видны в списке

### Requirement: Remove dictionary entry
Система SHALL позволять удалить запись; после удаления lookup по этой лемме MUST сообщать отсутствие.

#### Scenario [DE-05]: Remove entry
- **GIVEN** в словаре есть лемма «run»
- **WHEN** пользователь удаляет эту запись
- **THEN** «run» больше не в списке и lookup по «run» не находит запись

### Requirement: Dictionary persists across restart
Система SHALL сохранять записи словаря между перезапусками приложения на поддерживаемых платформах (включая Web / browser storage через Drift).

#### Scenario [DE-08]: Entries survive restart
- **GIVEN** пользователь сохранил хотя бы одну запись
- **WHEN** приложение перезапускается (на Web — тот же origin)
- **THEN** запись по-прежнему доступна в списке и через lookup

### Requirement: Vocab add form offline translate button
На форме добавления записи словаря (экран Vocabulary) система SHALL показывать действие Translate (`Key('vocabTranslateButton')`) рядом с полем lemma/слова (`vocabLemmaField`). Пока lemma после trim пуста, Translate MUST быть недоступно (disabled). По нажатию система MUST показать loading на кнопке, запросить офлайн EN→RU gloss через injectable `OfflineTranslateFn` для **surface текста поля lemma как на экране**, и при успехе вставить результат в поле Translation (`vocabTranslationField`, перезапись без подтверждения). Пока идёт запрос, Translate MUST быть disabled; после успеха или ошибки Translate снова MUST быть enabled (если lemma непуста). Кнопка Add MUST NOT блокироваться только из-за Translate. Система MUST NOT сохранять запись в словарь только из-за Translate — persist по-прежнему через явное Add. При ошибке перевода система MUST показать SnackBar (`Key('vocabTranslateErrorSnack')`) с user-safe copy и MUST NOT менять текст Translation из этого запроса. Engine — `translate/offline`.

#### Scenario [DE-19]: Translate fills translation on Vocab form
- **GIVEN** на Vocab в поле lemma непустой английский текст, Translation может быть пустым или заполненным
- **WHEN** пользователь нажимает Translate (`vocabTranslateButton`) и движок успешно отвечает
- **THEN** поле Translation (`vocabTranslationField`) содержит полученный gloss; запись в словарь ещё не создана, пока не нажато Add

#### Scenario [DE-20]: Translate disabled when lemma empty
- **GIVEN** поле lemma пусто или только whitespace
- **WHEN** пользователь смотрит на форму добавления
- **THEN** Translate (`vocabTranslateButton`) недоступно (`onPressed == null`)

#### Scenario [DE-21]: Translate error on Vocab does not fake gloss
- **GIVEN** lemma непуста, но offline translate завершился ошибкой
- **WHEN** Translate завершился
- **THEN** показан SnackBar `vocabTranslateErrorSnack`; поле Translation не изменено этим запросом; Translate снова enabled

### Requirement: Vocab translate attribution and accuracy notice
На форме добавления Vocab система SHALL показывать блок (`Key('vocabTranslateAttribution')`) с атрибуцией `OPUS-MT (Helsinki-NLP), CC-BY 4.0` и disclaimer `Translation may be inaccurate`.

#### Scenario [DE-22]: Attribution and disclaimer visible on Vocab
- **GIVEN** открыт экран Vocabulary с формой добавления
- **WHEN** пользователь видит форму
- **THEN** видны `vocabTranslateAttribution` с текстами OPUS-MT / CC-BY 4.0 и `Translation may be inaccurate`

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| DE-01 | Add entry with context | unit: insert morph lemma + translation (+ context) → list |
| DE-02 | Add entry without context | unit: optional context null/empty |
| DE-03 | Second add of same lemma | unit: second add same key updates one row |
| DE-04 | List after several adds | unit/widget: list size / contents |
| DE-05 | Remove entry | unit/widget: delete + lookup miss |
| DE-06 | Lookup hit | unit: lookup hit by key (case-insensitive) |
| DE-07 | Lookup miss | unit: membership false |
| DE-08 | Entries survive restart | integration (close/reopen DB) + verify/ops Web same-origin restart |
| DE-09 | Reject empty lemma or translation | unit/widget: validation |
| DE-10 | Case-insensitive lemma identity | unit: Run/run → one row |
| DE-11 | Empty vocabulary CTA | widget: empty allows in-UI add |
| DE-12 | Add entry from Reader selection | Flutter: Reader single → morph; phrase → casefold(join) |
| DE-13 | Upsert from Reader same as Vocab | Flutter/unit: same key updates one row |
| DE-14 | Surface key stored for token | unit: single keeps surface_key casefold surface + morph lemma |
| DE-15 | Schema migration merges collisions | unit/integration: stub collapse → one morph row |
| DE-16 | Phrase rows survive migration | unit: multi-word key remains as phrase identity |
| DE-17 | Morph-fail key is casefold surface | unit: non-EN/junk single = casefold(surface) |
| DE-18 | Phrase key is casefold join | unit/Flutter: multi-word lemma = casefold(join token surfaces) |
| DE-19 | Translate fills translation on Vocab form | `flutter test` Vocab + injectable `OfflineTranslateFn` + Keys |
| DE-20 | Translate disabled when lemma empty | `flutter test` `vocabTranslateButton` disabled |
| DE-21 | Translate error on Vocab does not fake gloss | `flutter test` `vocabTranslateErrorSnack` |
| DE-22 | Attribution and disclaimer visible on Vocab | `flutter test` `vocabTranslateAttribution` + copy |
