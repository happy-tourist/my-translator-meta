Связь: Reader Add/edit — `reader/tap` / `reader/highlight`. Identity: morph lemma для single-word; casefold(join) для multi-word phrases. Highlight membership — см. `reader/highlight`.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| DE-01 | Add entry with context | unit: insert morph lemma + translation (+ context) → list |
| DE-03 | Second add of same lemma | unit: second add same key updates one row |
| DE-06 | Lookup hit | unit: lookup hit by key (case-insensitive) |
| DE-10 | Case-insensitive lemma identity | unit: Run/run → one row |
| DE-12 | Add entry from Reader selection | Flutter: Reader single → morph; phrase → casefold(join) |
| DE-13 | Upsert from Reader same as Vocab | Flutter/unit: same key updates one row |
| DE-14 | Surface key stored for token | unit: single keeps surface_key casefold surface + morph lemma |
| DE-15 | Schema migration merges collisions | unit/integration: stub collapse → one morph row |
| DE-16 | Phrase rows survive migration | unit: multi-word key remains as phrase identity |
| DE-17 | Morph-fail key is casefold surface | unit: non-EN/junk single = casefold(surface) |
| DE-18 | Phrase key is casefold join | unit/Flutter: multi-word lemma = casefold(join token surfaces) |

## MODIFIED Requirements

### Requirement: Add dictionary entry
Система SHALL позволять добавить запись словаря с unique identity `lemma` и переводом через **Vocab UI** или **Reader** Add-sheet. Контекст MAY быть пустым. MUST сохраняться **surface_key** (casefold исходного surface / phrase join). Identity:
- **single-word** (нет whitespace в ключе): morph lemma (EN WordNet; morph-fail → casefold(surface));
- **multi-word / phrase** (whitespace в ключе): casefold(join) без WordNet.
Reader MAY добавлять и single, и phrase (contiguous selection). Vocab manual add: single через `lemmaKeysForSurfaces`; multi-word input → casefold identity без morph cascade.

#### Scenario [DE-01]: Add entry with context
- **GIVEN** словарь пуст
- **WHEN** пользователь добавляет morph-лемму «run», перевод «бежать» и контекст «He was running»
- **THEN** запись доступна в списке с этими полями

#### Scenario [DE-02]: Add entry without context
- **GIVEN** словарь может содержать другие записи
- **WHEN** пользователь добавляет «cat» / «кот» без контекста
- **THEN** запись сохраняется; context пуст или отсутствует

#### Scenario [DE-09]: Reject empty lemma or translation
- **GIVEN** форма добавления открыта
- **WHEN** подтверждение с пустой леммой или пустым переводом после trim
- **THEN** запись не создаётся; ошибка валидации

#### Scenario [DE-11]: Empty vocabulary CTA
- **GIVEN** словарь пуст
- **WHEN** открыт экран словаря
- **THEN** empty state с возможностью добавить запись на экране

#### Scenario [DE-12]: Add entry from Reader selection
- **GIVEN** в Reader non-vocab selection (single или contiguous phrase) и открыт Add-sheet
- **WHEN** непустой перевод и подтверждение
- **THEN** запись с корректным identity (morph или casefold join) доступна в словаре

#### Scenario [DE-14]: Surface key stored for token
- **GIVEN** сохранение single surface «Running» (morph «run»)
- **WHEN** запись создана
- **THEN** lemma = «run», surface_key = casefold surface («running»)

#### Scenario [DE-17]: Morph-fail key is casefold surface
- **GIVEN** single слово без отличной EN morph-леммы
- **WHEN** запись сохраняется
- **THEN** lemma identity = casefold(surface)

#### Scenario [DE-18]: Phrase key is casefold join
- **GIVEN** Reader contiguous selection из токенов «Hello» и «world»
- **WHEN** запись сохраняется из Add-sheet
- **THEN** lemma (и surface_key) = casefold(join) («hello world»), без WordNet на фразе целиком

### Requirement: Duplicate lemma policy
Система SHALL при повторном добавлении того же **lemma key** (morph или phrase) выполнять upsert translation/context без второго ряда. Политика едина для Vocab UI, Reader Add, Reader edit.

#### Scenario [DE-03]: Second add of same lemma
- **GIVEN** есть «run»
- **WHEN** снова добавляют «run» с другим переводом
- **THEN** одна запись с обновлёнными полями

#### Scenario [DE-10]: Case-insensitive lemma identity
- **GIVEN** есть «run»
- **WHEN** добавляют «Run»
- **THEN** upsert одной записи

### Requirement: Reader add uses shared upsert policy
Добавление из Reader MUST использовать ту же upsert-политику по `lemma`, что и Vocab UI.

#### Scenario [DE-13]: Upsert from Reader same as Vocab
- **GIVEN** запись с ключом «hello» (или phrase key)
- **WHEN** Reader сохраняет тот же ключ с другим translation/context
- **THEN** одна запись; поля обновлены

### Requirement: Lookup by lemma
Система SHALL поддерживать lookup по `lemma` (case-insensitive). Membership для highlight: morph keys и phrase keys — см. `reader/highlight`.

#### Scenario [DE-06]: Lookup hit
- **GIVEN** сохранена «run»
- **WHEN** lookup «run» / «Run»
- **THEN** hit с данными записи

#### Scenario [DE-07]: Lookup miss
- **GIVEN** «xyz» нет
- **WHEN** lookup «xyz»
- **THEN** miss

## ADDED Requirements

### Requirement: One-time stub-to-morph schema migration
При schema migration (один раз) система SHALL перевести stub-ключи в dual-key: `lemma` = morph для singles; multi-word → casefold phrase identity + `surface_key`. Коллизии singles: merge по max `updated_at`. Phrase rows MUST сохраниться. (Уже реализовано в foundation этого change; legacy cleanup — вне UI.)

#### Scenario [DE-15]: Schema migration merges collisions
- **GIVEN** две stub-записи «running» и «ran» с разными updated_at
- **WHEN** одноразовая migration
- **THEN** одна запись morph «run»; поля от max updated_at

#### Scenario [DE-16]: Phrase rows survive migration
- **GIVEN** multi-word stub-ключ до migration
- **WHEN** schema migration
- **THEN** фразовый ряд остаётся; участвует в phrase highlight membership (`reader/highlight`)
