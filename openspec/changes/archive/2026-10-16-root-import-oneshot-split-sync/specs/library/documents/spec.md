# library/documents (delta) — root-level documents

Связано с `library/folders` (смешанный список в корне). Документы MAY жить в корне без папки.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| LD-01 | Import valid subtitle file | unit/widget: import into folder still works |
| LD-02 | Import at library root succeeds | unit/widget: import with null folderId succeeds; document visible at root |
| LD-03 | Reject unsupported file type | unchanged |
| LD-07 | Import ASS/SSA file | unchanged intent (folder or root) |
| LD-24 | Import valid EPUB file | unit/widget: epub at root or in folder |
| LD-04 | List documents after import | unchanged in-folder; plus root list |
| LD-06 | Documents survive restart | unit/integration: root or folder doc survives reopen |
| LD-12 | Hidden translation not listed | root or folder: satellite hidden |
| LD-13 | Persist translation linked to original | same location as original (folder or root) |
| LD-29 | List root documents | unit/widget: root shows imported docs without folder |
| LD-30 | Open root document opens Reader | Flutter widget: tap root doc → Reader |

## REMOVED Requirements

### Requirement: Import subtitle into folder
**Reason**: Import разрешён в корне; требование переписано как «папка или корень».
**Migration**: Использовать ADDED «Import subtitle or EPUB into folder or library root» ниже; LD-02 переименован в «Import at library root succeeds» (успешный root import; прежний reject снят).

## ADDED Requirements

### Requirement: Import subtitle or EPUB into folder or library root
Система SHALL позволять импортировать файл субтитров (SRT, VTT или ASS/SSA) **или EPUB** внутрь существующей папки **или в корень** библиотеки (без папки) и сохранять документ в локальном хранилище библиотеки. Поддерживаемые расширения: `.srt`, `.vtt`, `.ass`, `.ssa` (`.ssa` хранится как format `ass`), `.epub` (хранится как format `epub`). Документ в корне MUST NOT требовать предварительного создания папки.

#### Scenario [LD-01]: Import valid subtitle file
- **GIVEN** существует папка «S01»
- **WHEN** пользователь выбирает валидный `.srt` файл с устройства и импортирует его в «S01»
- **THEN** в «S01» появляется документ с узнаваемым названием (из имени файла, если иное не задано)

#### Scenario [LD-02]: Import at library root succeeds
- **GIVEN** пользователь находится в корне библиотеки
- **WHEN** он успешно импортирует валидный поддерживаемый файл без выбора папки
- **THEN** документ появляется в корневом списке библиотеки (import без папки разрешён; прежнее reject-поведение снято)

#### Scenario [LD-03]: Reject unsupported file type
- **GIVEN** пользователь в корне или в папке
- **WHEN** он выбирает файл неподдерживаемого типа (не SRT/VTT/ASS/SSA/EPUB)
- **THEN** система не сохраняет документ и сообщает в UI, что формат не поддерживается

#### Scenario [LD-07]: Import ASS/SSA file
- **GIVEN** существует папка «S01» или пользователь в корне
- **WHEN** пользователь импортирует файл `.ssa` (или `.ass`)
- **THEN** документ сохраняется с format `ass` и виден в выбранном расположении (папка или корень)

#### Scenario [LD-24]: Import valid EPUB file
- **GIVEN** пользователь в корне или в папке «S01»
- **WHEN** пользователь импортирует валидный `.epub` файл
- **THEN** документ сохраняется с format `epub` и виден в этом расположении

### Requirement: Open document from root into Reader
Система SHALL при выборе видимого документа в корне библиотеки открывать Reader так же, как из папки: parse по формату через **существующий** FRB (`parse_subtitles` / `parse_epub`, без новых API), linked translation и reading/video anchors при наличии (`library/documents` open rules). Сбой перевода/видео MUST NOT блокировать open оригинала.

#### Scenario [LD-30]: Open root document opens Reader
- **GIVEN** в корне есть импортированный документ с валидным содержимым субтитров или EPUB
- **WHEN** пользователь выбирает этот документ
- **THEN** открывается Reader с содержимым документа (cues или book-режим)

## MODIFIED Requirements

### Requirement: List documents in folder
Система SHALL показывать документы, принадлежащие выбранной папке **или корню** (документы без папки), за исключением скрытых переводных документов, созданных через загрузку в нижней панели Reader и привязанных к оригиналу. Обычный import по-прежнему SHALL делать документ видимым в списке соответствующего расположения.

#### Scenario [LD-04]: List documents after import
- **GIVEN** в папке «S01» импортированы два subtitle-документа обычным import
- **WHEN** пользователь открывает «S01»
- **THEN** оба документа видны в содержимом папки

#### Scenario [LD-12]: Hidden translation not listed
- **GIVEN** в расположении (папка или корень) есть видимый оригинал и связанный с ним перевод, загруженный через нижнюю панель Reader
- **WHEN** пользователь открывает это расположение в Library
- **THEN** в списке виден оригинал, а переводный документ не отображается как отдельная строка

#### Scenario [LD-29]: List root documents
- **GIVEN** в корне импортирован хотя бы один видимый документ и есть ноль или более папок
- **WHEN** пользователь открывает корень библиотеки
- **THEN** корневые документы видны в списке вместе с папками верхнего уровня (см. `library/folders`)

### Requirement: Persist Reader translation as hidden linked document
Система SHALL при успешной загрузке перевода из нижней панели Reader сохранять файл перевода в **том же расположении**, что и оригинал (та же папка или корень, если оригинал без папки), как полноценный документ хранилища (байты/format по тем же правилам platform storage, что и обычный import), и SHALL установить связь original→translation (один слот перевода на оригинал). Такой переводный документ MUST NOT появляться в списке расположения (см. List documents). Связь MUST переживать перезапуск приложения на поддерживаемых платформах. Повторная загрузка поверх уже привязанного перевода без предварительного удаления не требуется (см. `reader/split`: delete, затем load).

#### Scenario [LD-13]: Persist translation linked to original
- **GIVEN** Reader открыт для оригинала из папки «S01» или из корня, split включён
- **WHEN** пользователь успешно загружает переводный файл в нижнюю панель
- **THEN** перевод сохраняется в том же расположении, что оригинал, связан с ним, и после перезапуска приложения связь и содержимое доступны при повторном открытии оригинала

### Requirement: Persist across app restart
Система SHALL сохранять импортированные документы и их расположение (папка или корень без папки) между перезапусками приложения на поддерживаемых платформах (включая Web / browser storage через Drift).

#### Scenario [LD-06]: Documents survive restart
- **GIVEN** пользователь импортировал документ в папку или в корень
- **WHEN** приложение полностью перезапускается (на Web — тот же origin) и пользователь снова открывает то же расположение
- **THEN** документ по-прежнему отображается в этом расположении
