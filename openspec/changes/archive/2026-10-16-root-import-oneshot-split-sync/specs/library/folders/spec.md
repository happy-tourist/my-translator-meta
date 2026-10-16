# library/folders (delta) — mixed root/folder list

Связано с `library/documents` (корневые документы). Папки остаются опциональной группировкой.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| LF-05 | Empty folder | widget: empty folder (не корень) → empty state, no error |
| LF-09 | Root lists folders and documents together | Flutter widget: root shows folders then docs in one list |
| LF-10 | Folder lists child folders and documents together | Flutter widget: open folder → mixed children |
| LF-11 | Empty root shows import and create folder | Flutter widget: empty root → Create folder + Import |

## MODIFIED Requirements

### Requirement: List folder children
Система SHALL отдавать для выбранной папки (или корня) список непосредственных дочерних папок **и** видимых документов этого расположения в **одном** списке UI (папки для удобной группировки, не обязательный контейнер для import). В корне документы без папки MUST отображаться рядом с папками верхнего уровня. Порядок SHOULD быть: сначала папки, затем документы.

#### Scenario [LF-04]: List children of folder
- **GIVEN** у «Сериалы» есть дочерние папки «S01» и «S02»
- **WHEN** пользователь открывает «Сериалы»
- **THEN** он видит «S01» и «S02» как содержимое этой папки

#### Scenario [LF-09]: Root lists folders and documents together
- **GIVEN** в корне есть папка «Сериалы» и корневой документ «episode.srt»
- **WHEN** пользователь открывает корень библиотеки
- **THEN** в одном списке видны и папка, и документ (папки выше документов)

#### Scenario [LF-10]: Folder lists child folders and documents together
- **GIVEN** в папке «S01» есть вложенная папка и документ
- **WHEN** пользователь открывает «S01»
- **THEN** в одном списке видны вложенная папка и документ

#### Scenario [LF-05]: Empty folder
- **GIVEN** папка без дочерних папок и без документов
- **WHEN** пользователь открывает её
- **THEN** система показывает пустое состояние без ошибки (с понятным empty UI, не crash)

## ADDED Requirements

### Requirement: Empty root offers import and create folder
В пустом корне библиотеки система SHALL предлагать и создание папки, и import файла (субтитры/EPUB) без обязательного предварительного создания папки.

#### Scenario [LF-11]: Empty root shows import and create folder
- **GIVEN** корень без папок и без документов
- **WHEN** пользователь открывает корень
- **THEN** доступны действия Create folder и Import
