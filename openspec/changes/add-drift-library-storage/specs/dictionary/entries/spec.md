## Purpose

Хранит локальный личный словарь пользователя (лемма, перевод, опциональный контекст) для проверки persistence и будущего матчинга подсветки в Reader.

**Owner (this change):** Flutter Drift в `lib/` — source of truth записей. Rust/FRB storage API для словаря **не** требуется; compute/NLP (lemma highlight later) — вне этого change.

## ADDED Requirements

### Requirement: Add dictionary entry
Система SHALL позволять добавить запись словаря с леммой и переводом через Vocab UI (Reader word-card add — вне этого change). Контекст MAY быть пустым. Идентичность для lookup — **нормализованная** лемма (`trim` + Unicode case-fold / lowercase).

#### Scenario [DE-01]: Add entry with context
- **GIVEN** словарь пуст
- **WHEN** пользователь добавляет лемму «run», перевод «бежать» и контекст «He was running»
- **THEN** запись доступна в списке словаря с этими полями

#### Scenario [DE-02]: Add entry without context
- **GIVEN** словарь может уже содержать другие записи
- **WHEN** пользователь добавляет лемму «cat» и перевод «кот» без контекста
- **THEN** запись сохраняется; поле контекста пустое или отсутствует

#### Scenario [DE-09]: Reject empty lemma or translation
- **GIVEN** пользователь открыл форму добавления
- **WHEN** он подтверждает пустую лемму или пустой перевод (после trim)
- **THEN** система не создаёт запись и показывает ошибку валидации

#### Scenario [DE-11]: Empty vocabulary CTA
- **GIVEN** словарь пуст
- **WHEN** пользователь открывает экран словаря
- **THEN** видит empty state с возможностью добавить запись на этом экране (не требуется переход в Reader)

### Requirement: Duplicate lemma policy
Система SHALL при повторном добавлении той же (нормализованной) леммы выполнять upsert: обновить `translation` и `context` существующей записи и не создавать второй ряд. Политика upsert MUST быть единой во всём приложении (см. design D4).

#### Scenario [DE-03]: Second add of same lemma
- **GIVEN** в словаре уже есть лемма «run»
- **WHEN** пользователь снова добавляет «run» с другим переводом (и опционально новым контекстом)
- **THEN** в списке остаётся одна запись с леммой «run» и обновлёнными полями перевода/контекста, без двух строк с одной леммой

#### Scenario [DE-10]: Case-insensitive lemma identity
- **GIVEN** в словаре есть лемма «run»
- **WHEN** пользователь добавляет «Run» (другой регистр) с новым переводом
- **THEN** выполняется upsert одной записи; второго ряда нет

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

### Requirement: Lookup by lemma
Система SHALL поддерживать проверку наличия / получение записи по нормализованной лемме.

#### Scenario [DE-06]: Lookup hit
- **GIVEN** сохранена лемма «run»
- **WHEN** выполняется lookup по «run» или «Run»
- **THEN** система сообщает, что запись есть (и возвращает данные записи при запросе полной карточки)

#### Scenario [DE-07]: Lookup miss
- **GIVEN** леммы «xyz» нет в словаре
- **WHEN** выполняется lookup по «xyz»
- **THEN** система сообщает отсутствие записи

### Requirement: Dictionary persists across restart
Система SHALL сохранять записи словаря между перезапусками приложения на поддерживаемых платформах (включая Web / browser storage через Drift).

#### Scenario [DE-08]: Entries survive restart
- **GIVEN** пользователь сохранил хотя бы одну запись
- **WHEN** приложение перезапускается (на Web — тот же origin)
- **THEN** запись по-прежнему доступна в списке и через lookup

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| DE-01 | Add entry with context | unit: insert → list contains fields |
| DE-02 | Add entry without context | unit: optional context null/empty |
| DE-03 | Second add of same lemma | unit: upsert updates fields; never two rows |
| DE-04 | List after several adds | unit/widget: list size / contents |
| DE-05 | Remove entry | unit/widget: delete + lookup miss |
| DE-06 | Lookup hit | unit: membership true (incl. case fold) |
| DE-07 | Lookup miss | unit: membership false |
| DE-08 | Entries survive restart | integration (close/reopen DB) + verify/ops Web same-origin restart |
| DE-09 | Reject empty lemma or translation | unit/widget: validation |
| DE-10 | Case-insensitive lemma identity | unit: Run/run → one row |
| DE-11 | Empty vocabulary CTA | widget: empty allows in-UI add |
