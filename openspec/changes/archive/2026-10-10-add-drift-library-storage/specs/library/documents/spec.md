## Purpose

Позволяет импортировать файлы субтитров с устройства в выбранную папку библиотеки, хранить их локально и удалять вместе с метаданными, чтобы позже открывать в Reader.

**Owner (this change):** Flutter Drift (+ native sandbox FS / Web BLOB) в `lib/` — source of truth документов. Rust/FRB storage API **не** требуется; парсинг cues — вне этого change.

## ADDED Requirements

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
Система SHALL показывать документы, принадлежащие выбранной папке.

#### Scenario [LD-04]: List documents after import
- **GIVEN** в папке «S01» импортированы два subtitle-документа
- **WHEN** пользователь открывает «S01»
- **THEN** оба документа видны в содержимом папки

### Requirement: Delete document
Система SHALL позволять удалить отдельный документ; после удаления он не появляется в библиотеке, а связанные локальные данные документа недоступны.

#### Scenario [LD-05]: Delete one document
- **GIVEN** в папке есть документ «episode1.srt»
- **WHEN** пользователь удаляет этот документ
- **THEN** документ исчезает из списка папки, соседние документы и папки остаются

### Requirement: Document row does not open Reader yet
В этом change система SHALL NOT требовать экран Reader при выборе документа: tap по документу MAY быть no-op или кратким сообщением «Reader later», без перехода в parse/reader flow.

#### Scenario [LD-08]: Tap document without Reader
- **GIVEN** в папке есть импортированный документ
- **WHEN** пользователь выбирает документ
- **THEN** приложение не падает и не требует Rust parse API; Reader не является обязательным следующим шагом этого change

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

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| LD-01 | Import valid subtitle file | unit/integration: import → row in folder |
| LD-02 | Reject import without folder | unit/widget: import to null/root rejected |
| LD-03 | Reject unsupported file type | unit/widget: extension policy reject + UI message |
| LD-04 | List documents after import | unit: list by folder id |
| LD-05 | Delete one document | unit: delete removes only that doc |
| LD-06 | Documents survive restart | integration (close/reopen DB) + verify/ops Web same-origin restart |
| LD-07 | Import ASS/SSA file | unit: `.ssa`/`.ass` → format `ass` |
| LD-08 | Tap document without Reader | widget: no Reader/parse dependency |
| LD-09 | Web import stores content | unit (fake platform) or verify/ops Web: content persisted |
