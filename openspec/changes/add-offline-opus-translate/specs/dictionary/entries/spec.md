# dictionary/entries Specification (delta)

## Purpose

Delta к main `dictionary/entries`: кнопка офлайн-перевода на форме добавления Vocab. Persist/upsert без изменений — см. main spec. Engine — `translate/offline`.

## ADDED Requirements

### Requirement: Vocab add form offline translate button
На форме добавления записи словаря (экран Vocabulary) система SHALL показывать действие Translate (`Key('vocabTranslateButton')`) рядом с полем lemma/слова (`vocabLemmaField`). Пока lemma после trim пуста, Translate MUST быть недоступно (disabled). По нажатию система MUST показать loading на кнопке, запросить офлайн EN→RU gloss через injectable `OfflineTranslateFn` для **surface текста поля lemma как на экране**, и при успехе вставить результат в поле Translation (`vocabTranslationField`, перезапись без подтверждения). Пока идёт запрос, Translate MUST быть disabled; после успеха или ошибки Translate снова MUST быть enabled (если lemma непуста). Кнопка Add MUST NOT блокироваться только из-за Translate. Система MUST NOT сохранять запись в словарь только из-за Translate — persist по-прежнему через явное Add. При ошибке перевода система MUST показать SnackBar (`Key('vocabTranslateErrorSnack')`) с user-safe copy (design D4.5) и MUST NOT менять текст Translation из этого запроса.

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
На форме добавления Vocab система SHALL показывать блок (`Key('vocabTranslateAttribution')`) с атрибуцией `OPUS-MT (Helsinki-NLP), CC-BY 4.0` и disclaimer `Translation may be inaccurate` (design D4.6).

#### Scenario [DE-22]: Attribution and disclaimer visible on Vocab
- **GIVEN** открыт экран Vocabulary с формой добавления
- **WHEN** пользователь видит форму
- **THEN** видны `vocabTranslateAttribution` с текстами OPUS-MT / CC-BY 4.0 и `Translation may be inaccurate`

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| DE-19 | Translate fills translation on Vocab form | `flutter test` Vocab + injectable `OfflineTranslateFn` + Keys |
| DE-20 | Translate disabled when lemma empty | `flutter test` `vocabTranslateButton` disabled |
| DE-21 | Translate error on Vocab does not fake gloss | `flutter test` `vocabTranslateErrorSnack` |
| DE-22 | Attribution and disclaimer visible on Vocab | `flutter test` `vocabTranslateAttribution` + copy |
