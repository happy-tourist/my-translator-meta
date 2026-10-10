## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| LD-04 | List documents after import | unit/widget: two visible imports listed (unchanged intent) |
| LD-12 | Hidden translation not listed | unit/widget: reader-attached translation absent from folder list |
| LD-13 | Persist translation linked to original | unit: attach creates stored doc + link in same folder |
| LD-14 | Delete translation unlinks and removes | unit: delete translation removes bytes/row and clears link |
| LD-15 | Delete original cascades translation | unit: delete original removes linked translation too |
| LD-10 | Open document opens Reader after successful parse | Flutter widget: tap valid doc without translation → Reader |
| LD-16 | Open original loads linked translation for Reader | Flutter widget/unit: open with link → Reader session has translation cues available |
| LD-05 | Delete one document | unit: delete visible doc without translation still works |

## MODIFIED Requirements

### Requirement: List documents in folder
Система SHALL показывать документы, принадлежащие выбранной папке, за исключением скрытых переводных документов, созданных через загрузку в нижней панели Reader и привязанных к оригиналу. Обычный import в папку по-прежнему SHALL делать документ видимым в списке.

#### Scenario [LD-04]: List documents after import
- **GIVEN** в папке «S01» импортированы два subtitle-документа обычным import
- **WHEN** пользователь открывает «S01»
- **THEN** оба документа видны в содержимом папки

#### Scenario [LD-12]: Hidden translation not listed
- **GIVEN** в папке есть видимый оригинал и связанный с ним перевод, загруженный через нижнюю панель Reader
- **WHEN** пользователь открывает эту папку в Library
- **THEN** в списке виден оригинал, а переводный документ не отображается как отдельная строка

### Requirement: Delete document
Система SHALL позволять удалить отдельный видимый документ; после удаления он не появляется в библиотеке, а связанные локальные данные документа недоступны. Если у удаляемого оригинала есть связанный скрытый перевод, система MUST удалить и этот перевод (cascade).

#### Scenario [LD-05]: Delete one document
- **GIVEN** в папке есть документ «episode1.srt» без связанного перевода
- **WHEN** пользователь удаляет этот документ
- **THEN** документ исчезает из списка папки, соседние документы и папки остаются

#### Scenario [LD-15]: Delete original cascades translation
- **GIVEN** оригинал имеет связанный скрытый переводный документ
- **WHEN** пользователь удаляет оригинал из Library
- **THEN** оригинал и связанный перевод удаляются из хранилища, и перевод не остаётся сиротой в папке

### Requirement: Open document from folder into Reader
Система SHALL при выборе видимого документа в папке загружать его содержимое из локального хранилища библиотеки, запрашивать парсинг субтитров и при успехе открывать Reader с разобранными cues. Если у документа есть связанный сохранённый перевод, система SHALL также загрузить и распарсить перевод (при успехе) и передать его в сессию Reader для нижней панели split. Import по-прежнему не обязан валидировать содержимое; проверка оригинала выполняется при открытии. Сбой загрузки/парсинга **перевода** при успешном оригинале MUST NOT блокировать открытие Reader с оригиналом (ошибка перевода — в контексте Reader / split при необходимости).

#### Scenario [LD-10]: Open document opens Reader after successful parse
- **GIVEN** в папке есть импортированный документ с валидным содержимым субтитров (хотя бы одна реплика) без перевода
- **WHEN** пользователь выбирает этот документ
- **THEN** открывается Reader с репликами документа

#### Scenario [LD-16]: Open original loads linked translation for Reader
- **GIVEN** оригинал имеет валидный связанный перевод в хранилище
- **WHEN** пользователь открывает оригинал из папки
- **THEN** Reader открывается с cues оригинала, и cues перевода доступны для нижней панели split без повторного выбора файла с устройства

## ADDED Requirements

### Requirement: Persist Reader translation as hidden linked document
Система SHALL при успешной загрузке перевода из нижней панели Reader сохранять файл перевода в той же папке, что и оригинал, как полноценный документ хранилища (байты/format по тем же правилам platform storage, что и обычный import), и SHALL установить связь original→translation (один слот перевода на оригинал). Такой переводный документ MUST NOT появляться в списке папки (см. List documents). Связь MUST переживать перезапуск приложения на поддерживаемых платформах. Повторная загрузка поверх уже привязанного перевода без предварительного удаления не требуется (см. `reader/split`: delete, затем load).

#### Scenario [LD-13]: Persist translation linked to original
- **GIVEN** Reader открыт для оригинала из папки «S01», split включён
- **WHEN** пользователь успешно загружает переводный subtitle-файл в нижнюю панель
- **THEN** перевод сохраняется в хранилище папки «S01», связан с оригиналом, и после перезапуска приложения связь и содержимое доступны при повторном открытии оригинала

### Requirement: Delete translation from Reader unlinks and removes storage
При удалении перевода из нижней панели Reader система SHALL удалить связанный переводный документ из локального хранилища и MUST снять связь с оригиналом. Оригинал MUST остаться в папке.

#### Scenario [LD-14]: Delete translation unlinks and removes
- **GIVEN** оригинал имеет сохранённый связанный перевод
- **WHEN** пользователь удаляет перевод в нижней панели Reader
- **THEN** переводный документ и его байты удалены, связь снята, оригинал остаётся видимым в папке
