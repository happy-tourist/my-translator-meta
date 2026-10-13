# reader/tap Specification (delta)

## Purpose

Delta к main `reader/tap`: офлайн Translate на Add-sheet и Edit-sheet. Selection/CTA/persist — main spec + `dictionary/entries`. Engine — `translate/offline`.

**SC-ID:** продолжение после main RT-17 — новые сценарии **RT-18…RT-25** (не переиспользовать RT-14…RT-17: там contiguous Add / in-vocab CTA / shrink / retap).

## ADDED Requirements

### Requirement: Reader Add-sheet offline translate button
На bottom sheet добавления в словарь из Reader (`readerAddVocabSheet`) система SHALL показывать действие Translate (`Key('readerAddVocabTranslate')`) рядом с полем Word (`readerAddVocabSurface`, read-only surface selection). По нажатию Translate система MUST показать loading на кнопке, запросить офлайн EN→RU gloss через injectable `OfflineTranslateFn` для **surface как на экране**, и при успехе вставить результат в поле Translation (`readerAddVocabTranslation`, перезапись без подтверждения). После старта Translate кнопка MUST оставаться disabled до закрытия этого sheet (и при успехе, и при ошибке). Save (`readerAddVocabSave`) MUST NOT блокироваться только из-за Translate. Система MUST NOT сохранять запись только из-за Translate. При ошибке — SnackBar `Key('readerAddVocabTranslateErrorSnack')` (design D4.5); MUST NOT менять Translation из этого запроса. Пустой Word не возникает: sheet открывается только при непустой selection (main tap flow).

#### Scenario [RT-18]: Translate fills Add-sheet translation
- **GIVEN** открыт Add-sheet с непустым Word (слово или фраза)
- **WHEN** пользователь нажимает Translate (`readerAddVocabTranslate`) и движок успешно отвечает
- **THEN** поле Translation (`readerAddVocabTranslation`) содержит gloss; sheet остаётся открытым; persist ещё не выполнен; Translate остаётся disabled

#### Scenario [RT-19]: Translate stays disabled until Add-sheet closes
- **GIVEN** на Add-sheet пользователь нажал Translate
- **WHEN** перевод завершился успехом или ошибкой, sheet ещё открыт
- **THEN** Translate (`readerAddVocabTranslate`) остаётся disabled (`onPressed == null`) до закрытия sheet

#### Scenario [RT-20]: Translate error on Add-sheet
- **GIVEN** открыт Add-sheet
- **WHEN** Translate завершился ошибкой
- **THEN** показан SnackBar `readerAddVocabTranslateErrorSnack`; Translation не изменена этим запросом; Translate остаётся disabled до закрытия sheet

### Requirement: Reader Edit-sheet offline translate button
На bottom sheet редактирования (`readerEditVocabSheet`) система SHALL предоставлять Translate (`Key('readerEditVocabTranslate')`) у Word/surface (`readerEditVocabSurface`), с тем же loading, fill `readerEditVocabTranslation`, disable до закрытия sheet после старта Translate, injectable `OfflineTranslateFn`, и без auto-save — сохранение только явным Save (`readerEditVocabSave`). При ошибке — SnackBar `Key('readerEditVocabTranslateErrorSnack')` (design D4.5); MUST NOT менять Translation из этого запроса.

#### Scenario [RT-21]: Translate on Edit-sheet fills translation
- **GIVEN** открыт Edit-sheet существующей записи
- **WHEN** пользователь нажимает Translate (`readerEditVocabTranslate`) и движок успешно отвечает
- **THEN** поле Translation (`readerEditVocabTranslation`) обновлено gloss; изменения в словарь не сохранены, пока пользователь не подтвердит Save; Translate остаётся disabled до закрытия sheet

#### Scenario [RT-24]: Translate stays disabled until Edit-sheet closes
- **GIVEN** на Edit-sheet пользователь нажал Translate
- **WHEN** перевод завершился успехом или ошибкой, sheet ещё открыт
- **THEN** Translate (`readerEditVocabTranslate`) остаётся disabled до закрытия sheet

#### Scenario [RT-25]: Translate error on Edit-sheet
- **GIVEN** открыт Edit-sheet
- **WHEN** Translate завершился ошибкой
- **THEN** показан SnackBar `readerEditVocabTranslateErrorSnack`; Translation не изменена этим запросом; Translate остаётся disabled до закрытия sheet

### Requirement: Reader sheet attribution and accuracy notice
На Add-sheet и Edit-sheet система SHALL показывать блоки `Key('readerAddVocabTranslateAttribution')` / `Key('readerEditVocabTranslateAttribution')` с copy design D4.6: `OPUS-MT (Helsinki-NLP), CC-BY 4.0` и `Translation may be inaccurate`.

#### Scenario [RT-22]: Attribution on Add-sheet
- **GIVEN** открыт Add-sheet
- **WHEN** пользователь видит содержимое sheet
- **THEN** виден `readerAddVocabTranslateAttribution` с OPUS-MT / CC-BY 4.0 и disclaimer

#### Scenario [RT-23]: Attribution on Edit-sheet
- **GIVEN** открыт Edit-sheet
- **WHEN** пользователь видит содержимое sheet
- **THEN** виден `readerEditVocabTranslateAttribution` с OPUS-MT / CC-BY 4.0 и disclaimer

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RT-18 | Translate fills Add-sheet translation | `flutter test` Add-sheet + injectable `OfflineTranslateFn` |
| RT-19 | Translate stays disabled until Add-sheet closes | `flutter test` `readerAddVocabTranslate` disabled |
| RT-20 | Translate error on Add-sheet | `flutter test` `readerAddVocabTranslateErrorSnack` |
| RT-21 | Translate on Edit-sheet fills translation | `flutter test` Edit-sheet fill |
| RT-22 | Attribution on Add-sheet | `flutter test` `readerAddVocabTranslateAttribution` |
| RT-23 | Attribution on Edit-sheet | `flutter test` `readerEditVocabTranslateAttribution` |
| RT-24 | Translate stays disabled until Edit-sheet closes | `flutter test` `readerEditVocabTranslate` disabled |
| RT-25 | Translate error on Edit-sheet | `flutter test` `readerEditVocabTranslateErrorSnack` |
