## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| LD-08 | *(removed)* | superseded by LD-10 / LD-11 |
| LD-10 | Open document opens Reader after successful parse | Flutter widget: tap valid doc → Reader |
| LD-11 | Failed open stays in Library with visible error | Flutter widget: tap bad/empty → error, no Reader |

## ADDED Requirements

### Requirement: Open document from folder into Reader
Система SHALL при выборе документа в папке загружать его содержимое из локального хранилища библиотеки, запрашивать парсинг субтитров и при успехе открывать Reader с разобранными cues. Import по-прежнему не обязан валидировать содержимое; проверка выполняется при открытии.

#### Scenario [LD-10]: Open document opens Reader after successful parse
- **GIVEN** в папке есть импортированный документ с валидным содержимым субтитров (хотя бы одна реплика)
- **WHEN** пользователь выбирает этот документ
- **THEN** открывается Reader с репликами документа

### Requirement: Failed open shows error without leaving Library
При ошибке загрузки содержимого, парсинга, encoding или нуле реплик система SHALL оставаться в Library (в текущей папке), SHALL показать пользователю заметную ошибку (SnackBar / эквивалент; стабильный test key — как в design D5) и MUST NOT открывать Reader. Тексты ошибок по kind — design D5 (UnsupportedFormat / Encoding / Empty / Malformed / load).

#### Scenario [LD-11]: Failed open stays in Library with visible error
- **GIVEN** в папке есть импортированный документ, содержимое которого не парсится успешно или даёт ноль реплик
- **WHEN** пользователь выбирает этот документ
- **THEN** пользователь остаётся в Library, видит ошибку открытия, и Reader не открывается

## REMOVED Requirements

### Requirement: Document row does not open Reader yet
**Reason:** Reader и parse входят в этот change; placeholder «Reader later» больше не актуален.
**Migration:** tap по документу следует требованиям «Open document from folder into Reader» и «Failed open shows error without leaving Library» (сценарии LD-10 / LD-11). Сценарий LD-08 снят.
