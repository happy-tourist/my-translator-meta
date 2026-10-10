# library/documents Specification

## Purpose

Позволяет импортировать файлы субтитров с устройства в выбранную папку библиотеки, хранить их локально и удалять вместе с метаданными, чтобы позже открывать в Reader. Source of truth документов — Flutter Drift (+ native sandbox FS / Web BLOB) в `lib/`; Rust/FRB storage API не требуется; парсинг cues — отдельная capability.

## Requirements

### Requirement: Import subtitle into folder
Система SHALL позволять импортировать файл субтитров (SRT, VTT или ASS/SSA) только внутрь существующей папки и сохранять документ в локальном хранилище библиотеки. Поддерживаемые расширения: `.srt`, `.vtt`, `.ass`, `.ssa` (`.ssa` хранится как format `ass`).

#### Scenario [LD-01]: Import valid subtitle file
- **GIVEN** существует папка «S01»
- **WHEN** пользователь выбирает валидный `.srt` файл с устройства и импортирует его в «S01»
- **THEN** в «S01» появляется документ с узнаваемым названием (из имени файла, если иное не задано)

#### Scenario [LD-07]: Import ASS/SSA file
- **GIVEN** существует папка «S01»
- **WHEN** пользователь импортирует файл `.ssa` (или `.ass`)
- **THEN** документ сохраняется с format `ass` и виден в папке

#### Scenario [LD-02]: Reject import without folder
- **GIVEN** пользователь находится в корне библиотеки (список папок)
- **WHEN** он пытается импортировать файл без выбора папки
- **THEN** система не создаёт документ в корне и требует выбрать папку (import CTA отсутствует или явно блокируется в корне)

#### Scenario [LD-03]: Reject unsupported file type
- **GIVEN** пользователь выбрал папку
- **WHEN** он выбирает файл неподдерживаемого типа (не SRT/VTT/ASS/SSA)
- **THEN** система не сохраняет документ и сообщает в UI, что формат не поддерживается

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
Система SHALL при выборе видимого документа в папке загружать его содержимое из локального хранилища библиотеки, запрашивать парсинг субтитров и при успехе открывать Reader с разобранными cues. Если у документа есть связанный сохранённый перевод, система SHALL также загрузить и распарсить перевод (при успехе) и передать его в сессию Reader для нижней панели split. Если у документа сохранён якорь чтения, система SHALL передать его в сессию Reader для восстановления позиции (см. `reader/split` RS-12 и requirement Persist reading anchor ниже). Import по-прежнему не обязан валидировать содержимое; проверка оригинала выполняется при открытии. Сбой загрузки/парсинга **перевода** при успешном оригинале MUST NOT блокировать открытие Reader с оригиналом (ошибка перевода — в контексте Reader / split при необходимости).

#### Scenario [LD-10]: Open document opens Reader after successful parse
- **GIVEN** в папке есть импортированный документ с валидным содержимым субтитров (хотя бы одна реплика) без перевода
- **WHEN** пользователь выбирает этот документ
- **THEN** открывается Reader с репликами документа

#### Scenario [LD-16]: Open original loads linked translation for Reader
- **GIVEN** оригинал имеет валидный связанный перевод в хранилище
- **WHEN** пользователь открывает оригинал из папки
- **THEN** Reader открывается с cues оригинала, и cues перевода доступны для нижней панели split без повторного выбора файла с устройства

### Requirement: Failed open shows error without leaving Library
При ошибке загрузки содержимого, парсинга, encoding или нуле реплик система SHALL оставаться в Library (в текущей папке), SHALL показать пользователю заметную ошибку (SnackBar / эквивалент) и MUST NOT открывать Reader.

#### Scenario [LD-11]: Failed open stays in Library with visible error
- **GIVEN** в папке есть импортированный документ, содержимое которого не парсится успешно или даёт ноль реплик
- **WHEN** пользователь выбирает этот документ
- **THEN** пользователь остаётся в Library, видит ошибку открытия, и Reader не открывается

### Requirement: Persist across app restart
Система SHALL сохранять импортированные документы и их привязку к папкам между перезапусками приложения на поддерживаемых платформах (включая Web / browser storage через Drift).

#### Scenario [LD-06]: Documents survive restart
- **GIVEN** пользователь импортировал документ в папку
- **WHEN** приложение полностью перезапускается (на Web — тот же origin) и пользователь снова открывает ту же папку
- **THEN** документ по-прежнему отображается в этой папке

### Requirement: Platform storage split for bytes
Система SHALL на native сохранять байты документа в app sandbox (path/key в метаданных) и SHALL на Web сохранять содержимое в Drift (BLOB/text), без зависимости от долговечного filesystem path.

#### Scenario [LD-09]: Web import stores content
- **GIVEN** приложение запущено на Web
- **WHEN** пользователь импортирует валидный subtitle в папку
- **THEN** документ доступен после reopen без опоры на путь исходного файла на диске пользователя

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

### Requirement: Persist reading anchor for original document
Система SHALL хранить на **оригинальном** документе якорь последней позиции чтения: время начала якорной реплики (`start` в миллисекундах) и относительное alignment видимой строки (если известно). Система MUST обновлять якорь при завершении жеста прокрутки оригинала в Reader (settle) и при выходе из Reader (pop), если позиция известна. Якорь MUST переживать перезапуск приложения на поддерживаемых платформах. При открытии оригинала система SHALL отдавать сохранённый якорь в сессию Reader (если есть). Если cues после парсинга пусты или якоря нет — поведение как open без resume. При несовпадении сохранённого `start` с текущими cues потребитель (Reader) использует ближайшую реплику по времени. Persist режима split и divider не требуется. Скрытый translation-документ не обязан хранить отдельный якорь чтения.

#### Scenario [LD-17]: Persist and load reading anchor for original
- **GIVEN** оригинал открыт в Reader и пользователь остановился на реплике с известным `start` (не начало файла)
- **WHEN** якорь сохранён (после settle и/или выхода из Reader) и пользователь снова открывает этот оригинал из папки
- **THEN** в сессию Reader передаётся сохранённый якорь (`start` и alignment при наличии), и он доступен для восстановления позиции списка

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| LD-01 | Import valid subtitle file | unit/integration: import → row in folder |
| LD-02 | Reject import without folder | unit/widget: import to null/root rejected |
| LD-03 | Reject unsupported file type | unit/widget: extension policy reject + UI message |
| LD-04 | List documents after import | unit/widget: two visible imports listed (unchanged intent) |
| LD-05 | Delete one document | unit: delete visible doc without translation still works |
| LD-06 | Documents survive restart | integration (close/reopen DB) + verify/ops Web same-origin restart |
| LD-07 | Import ASS/SSA file | unit: `.ssa`/`.ass` → format `ass` |
| LD-09 | Web import stores content | unit (fake platform) or verify/ops Web: content persisted |
| LD-10 | Open document opens Reader after successful parse | Flutter widget: tap valid doc without translation → Reader |
| LD-11 | Failed open stays in Library with visible error | Flutter widget: tap bad/empty → error, no Reader |
| LD-12 | Hidden translation not listed | unit/widget: reader-attached translation absent from folder list |
| LD-13 | Persist translation linked to original | unit: attach creates stored doc + link in same folder |
| LD-14 | Delete translation unlinks and removes | unit: delete translation removes bytes/row and clears link |
| LD-15 | Delete original cascades translation | unit: delete original removes linked translation too |
| LD-16 | Open original loads linked translation for Reader | Flutter widget/unit: open with link → Reader session has translation cues available |
| LD-17 | Persist and load reading anchor for original | unit/widget: save anchor → reopen → Reader gets same startMs (+ alignment) |
