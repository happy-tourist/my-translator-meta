Связь: Reader Add/edit — `reader/tap` / `reader/highlight`. Stub-only identity заменяется morph lemma; фразовые записи остаются, но без highlight до later.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| DE-01 | Add entry with context | unit: insert morph lemma + translation (+ context) → list |
| DE-03 | Second add of same lemma | unit: second add same morph lemma updates one row |
| DE-06 | Lookup hit | unit: lookup hit by morph key (case-insensitive) |
| DE-10 | Case-insensitive lemma identity | unit: Run/run → one morph row |
| DE-12 | Add entry from Reader selection | Flutter: Reader single-token save → entry keyed by morph lemma |
| DE-13 | Upsert from Reader same as Vocab | Flutter/unit: Reader save of existing morph key updates one row |
| DE-14 | Surface key stored for token | unit: entry keeps surface_key (casefold surface) alongside morph lemma |
| DE-15 | Schema migration merges collisions | unit/integration: two stub rows collapsing to one morph → one row; translation/context from max updated_at |
| DE-16 | Phrase rows survive migration | unit: multi-word stub key remains; not forced into single-token morph highlight identity |
| DE-17 | Morph-fail key is casefold surface | unit: non-EN/junk lemma identity = casefold(surface) |

## MODIFIED Requirements

### Requirement: Add dictionary entry
Система SHALL позволять добавить запись словаря с **morph lemma** (unique identity) и переводом через **Vocab UI** или через **Reader** (single-token Add-sheet; см. `reader/tap`). Контекст MAY быть пустым. Для записи MUST сохраняться **surface_key** (casefold исходного surface / stub-ключа) для legacy lookup и миграции. Идентичность для upsert/lookup — **morph lemma** (EN WordNet для английских словоформ; при morph-fail — casefold(surface)). Multi-word фраза как **новый** ключ из Reader в этом change MUST NOT добавляться (phrase save — later); уже существующие фразовые ряды MAY оставаться в persist без обязательной подсветки.

#### Scenario [DE-01]: Add entry with context
- **GIVEN** словарь пуст
- **WHEN** пользователь добавляет morph-лемму «run», перевод «бежать» и контекст «He was running»
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
- **GIVEN** в Reader на оригинале есть одиночный non-vocab selection и открыт sheet добавления
- **WHEN** пользователь вводит непустой перевод и подтверждает сохранение
- **THEN** запись с morph lemma identity выбранного токена и этим переводом доступна в словаре так же, как запись из Vocab UI

#### Scenario [DE-14]: Surface key stored for token
- **GIVEN** пользователь сохраняет токен с surface «Running» (morph «run»)
- **WHEN** запись создана
- **THEN** у записи lemma = morph «run» и surface_key отражает casefold surface («running»)

#### Scenario [DE-17]: Morph-fail key is casefold surface
- **GIVEN** добавляется слово, для которого EN morph не даёт отличной леммы (например кириллица)
- **WHEN** запись сохраняется
- **THEN** lemma identity = casefold(surface)

### Requirement: Duplicate lemma policy
Система SHALL при повторном добавлении той же **morph lemma** выполнять upsert: обновить `translation` и `context` существующей записи и не создавать второй ряд. Политика upsert MUST быть единой во всём приложении (Vocab UI, Reader Add, Reader edit).

#### Scenario [DE-03]: Second add of same lemma
- **GIVEN** в словаре уже есть лемма «run»
- **WHEN** пользователь снова добавляет «run» с другим переводом (и опционально новым контекстом)
- **THEN** в списке остаётся одна запись с леммой «run» и обновлёнными полями перевода/контекста, без двух строк с одной леммой

#### Scenario [DE-10]: Case-insensitive lemma identity
- **GIVEN** в словаре есть лемма «run»
- **WHEN** пользователь добавляет «Run» (другой регистр) с новым переводом
- **THEN** выполняется upsert одной записи; второго ряда нет

### Requirement: Reader add uses shared upsert policy
Добавление из Reader MUST использовать ту же политику upsert по **morph lemma**, что и Vocab UI: повторное сохранение того же ключа обновляет `translation` и `context`, не создавая второй ряд.

#### Scenario [DE-13]: Upsert from Reader same as Vocab
- **GIVEN** в словаре уже есть запись с morph-ключом «hello»
- **WHEN** пользователь из Reader сохраняет токен, чья lemma_key нормализуется к «hello», с другим переводом и/или context
- **THEN** остаётся одна запись; поля обновлены; второго ряда нет

### Requirement: Lookup by lemma
Система SHALL поддерживать проверку наличия / получение записи по **morph lemma** (case-insensitive). Lookup MAY дополнительно использовать `surface_key` для переходных/legacy сценариев, но membership для highlight MUST опираться на morph lemma identity.

#### Scenario [DE-06]: Lookup hit
- **GIVEN** сохранена лемма «run»
- **WHEN** выполняется lookup по «run» или «Run»
- **THEN** система сообщает, что запись есть (и возвращает данные записи при запросе полной карточки)

#### Scenario [DE-07]: Lookup miss
- **GIVEN** леммы «xyz» нет в словаре
- **WHEN** выполняется lookup по «xyz»
- **THEN** система сообщает отсутствие

## ADDED Requirements

### Requirement: One-time stub-to-morph schema migration
При schema migration (один раз, не на каждый cold start) система SHALL перевести существующие stub-ключи в dual-key модель: `lemma` = morph identity, `surface_key` = прежний stub/casefold surface. Если несколько stub-рядов схлопываются в одну morph lemma, система MUST **merge** в одну запись: `translation` и `context` берутся у ряда с наибольшим `updated_at`. Multi-word фразовые ряды MUST сохраниться как есть (без принудительного single-token morph highlight). Ручной UI разбора коллизий — later.

#### Scenario [DE-15]: Schema migration merges collisions
- **GIVEN** до миграции есть две stub-записи «running» и «ran» с разными translation/context и разными `updated_at`
- **WHEN** выполняется одноразовая schema migration
- **THEN** остаётся одна запись с morph lemma «run»; translation и context соответствуют ряду с большим `updated_at`; surface_key отражает legacy surface(s) по выбранной схеме dual-key

#### Scenario [DE-16]: Phrase rows survive migration
- **GIVEN** до миграции есть multi-word stub-ключ (фраза)
- **WHEN** выполняется schema migration
- **THEN** фразовый ряд остаётся в словаре и не удаляется; он не обязан получать single-token highlight membership
