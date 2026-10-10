## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| DE-12 | Add entry from Reader selection | Flutter: Reader sheet save → dictionary list/lookup contains entry |
| DE-13 | Upsert from Reader same as Vocab | Flutter/unit: Reader save of existing normalized key updates one row |

Поведение Vocab CRUD (DE-01…DE-11) без изменений сценариев; requirement «Add dictionary entry» расширяется источником Reader (раньше main: только Vocab UI; Reader — отдельный change). UI selection/sheet — `reader/tap`. Upsert из Reader — ADDED requirement ниже (согласован с main «Duplicate lemma policy»).

## MODIFIED Requirements

### Requirement: Add dictionary entry
Система SHALL позволять добавить запись словаря с леммой (или multi-word surface/фразой как ключом) и переводом через **Vocab UI** или через **Reader** (bottom sheet после selection на оригинале; см. `reader/tap`). Контекст MAY быть пустым. Идентичность для lookup — **нормализованный** ключ (`trim` + Unicode case-fold / lowercase). В этом change морфологическая лемматизация не требуется: ключ = нормализованный surface выбранного слова или contiguous фразы.

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

#### Scenario [DE-12]: Add entry from Reader selection
- **GIVEN** в Reader на оригинале есть contiguous selection и открыт sheet добавления
- **WHEN** пользователь вводит непустой перевод и подтверждает сохранение
- **THEN** запись с нормализованным ключом selection и этим переводом доступна в словаре так же, как запись, добавленная из Vocab UI

## ADDED Requirements

### Requirement: Reader add uses shared upsert policy
Добавление из Reader MUST использовать ту же политику upsert по нормализованному ключу, что и Vocab UI: повторное сохранение того же ключа обновляет `translation` и `context`, не создавая второй ряд.

#### Scenario [DE-13]: Upsert from Reader same as Vocab
- **GIVEN** в словаре уже есть запись с ключом «hello»
- **WHEN** пользователь из Reader сохраняет selection «Hello» с другим переводом и/или context
- **THEN** остаётся одна запись; поля обновлены; второго ряда нет
