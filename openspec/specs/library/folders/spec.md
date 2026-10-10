# library/folders Specification

## Purpose

Даёт пользователю дерево папок в библиотеке субтитров: создавать разделы, вкладывать папки друг в друга и удалять ветки целиком, чтобы группировать документы. Source of truth папок — Flutter Drift в `lib/`; Rust/FRB storage API не требуется.

## Requirements

### Requirement: Create folder in tree
Система SHALL позволять создать папку с непустым именем либо в корне библиотеки, либо внутри существующей папки.

#### Scenario [LF-01]: Create root folder
- **GIVEN** библиотека без папок
- **WHEN** пользователь создаёт папку с именем «Сериалы»
- **THEN** папка появляется в корне библиотеки

#### Scenario [LF-02]: Create nested folder
- **GIVEN** существует папка «Сериалы»
- **WHEN** пользователь создаёт внутри неё папку «S01»
- **THEN** «S01» отображается как дочерняя папка «Сериалов»

#### Scenario [LF-03]: Reject empty name
- **GIVEN** пользователь открыл создание папки
- **WHEN** он подтверждает пустое имя (или только пробелы)
- **THEN** система не создаёт папку и сообщает об ошибке валидации в UI

### Requirement: List folder children
Система SHALL отдавать для выбранной папки (или корня) список непосредственных дочерних папок.

#### Scenario [LF-04]: List children of folder
- **GIVEN** у «Сериалы» есть дочерние папки «S01» и «S02»
- **WHEN** пользователь открывает «Сериалы»
- **THEN** он видит «S01» и «S02» как содержимое этой папки

#### Scenario [LF-05]: Empty folder
- **GIVEN** папка без дочерних папок и без документов
- **WHEN** пользователь открывает её
- **THEN** система показывает пустое состояние без ошибки (с понятным empty UI, не crash)

### Requirement: Cascade delete folder
Система SHALL при удалении папки удалять все вложенные папки и все документы внутри этой ветки. UI SHALL запросить подтверждение перед cascade delete.

#### Scenario [LF-06]: Delete folder with nested content
- **GIVEN** папка «Сериалы» содержит вложенную папку и хотя бы один документ
- **WHEN** пользователь подтверждает удаление «Сериалы»
- **THEN** папка, все вложенные папки и документы этой ветки больше недоступны в библиотеке

#### Scenario [LF-07]: Cancel cascade delete
- **GIVEN** пользователь инициировал удаление папки с содержимым
- **WHEN** он отменяет подтверждение
- **THEN** папка и содержимое остаются без изменений

### Requirement: Library loading and error feedback
Система SHALL показывать состояние загрузки списка папок и SHALL сообщать об ошибке операции (create/delete), не оставляя UI в «тихом» failure.

#### Scenario [LF-08]: List loading then content
- **GIVEN** открыт корень или папка
- **WHEN** идёт чтение из Drift
- **THEN** пользователь видит loading, затем список или empty state

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| LF-01 | Create root folder | unit/widget: create folder → appears in root list |
| LF-02 | Create nested folder | unit: child has parent link; list under parent |
| LF-03 | Reject empty name | unit/widget: validation rejects blank name + UI message |
| LF-04 | List children of folder | unit: list returns only direct children |
| LF-05 | Empty folder | widget: empty state, no error |
| LF-06 | Delete folder with nested content | unit: cascade removes subtree + documents |
| LF-07 | Cancel cascade delete | widget: dismiss confirm → data intact |
| LF-08 | List loading then content | widget: loading → list/empty |
