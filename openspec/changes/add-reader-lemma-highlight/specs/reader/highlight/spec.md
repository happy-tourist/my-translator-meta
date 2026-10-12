## Purpose

Подсвечивает в Reader на панели оригинала single-token вхождения сохранённых морфологических лемм словаря и по тапу открывает edit-sheet с содержимым записи (правка перевода/контекста, удаление). Membership считает Rust; Flutter только красит и показывает chrome.

Связанные capabilities: токены/Add — `reader/tap`; persist — `dictionary/entries`. Phrase-span highlight — later.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RH-01 | Saved lemma highlights wordforms | Flutter: entry «run» → surfaces «running»/«ran» highlighted on original |
| RH-02 | Non-vocab tokens stay default | Flutter: unmarked tokens use default body style |
| RH-03 | Refresh after save | Flutter: after Add save, matching tokens gain highlight without reopening Reader |
| RH-04 | Refresh after delete | Flutter: after edit-sheet delete → re-mark cache; Vocab delete while Reader not mounted → clear on next open |
| RH-05 | Tap in_vocab opens edit sheet | Flutter: tap highlighted token → edit sheet prefilled, no Add CTA |
| RH-06 | Edit sheet upsert | Flutter: change translation/context → save → Drift upsert by morph lemma |
| RH-07 | Edit sheet delete | Flutter: delete from edit sheet → entry gone, highlight clears |
| RH-08 | Phrase entries not highlighted | Flutter: multi-word dict key does not paint any token highlight |
| RH-09 | Membership from Rust flags | Rust/Flutter: paint uses `is_in_vocab` from compute, not Dart-only Set without flags |
| RH-10 | Lemma key for EN wordforms | Rust `cargo test`: dogs/running → dog/run keys |
| RH-11 | Morph fail falls back to casefold | Rust `cargo test`: Cyrillic/junk surface → casefold(surface) as lemma_key |
| RH-12 | Translation pane no highlight | Flutter: split translation pane has no vocab highlight UX |

## ADDED Requirements

### Requirement: Batch lemma keys and vocab membership on original tokens
При открытии Reader система SHALL для токенов оригинала вычислить **lemma_key** (морфологический ключ) и выставить **is_in_vocab** по множеству morph-лемм из словаря пользователя. Membership MUST вычисляться на стороне Rust compute (FRB); Flutter MUST NOT быть единственным источником истины для флага подсветки. Пакетность: один batch на открытие документа (как tokenize); MUST NOT вызывать per-token FFI на каждый rebuild строки. После изменения словаря (save/delete/migrate) система MUST обновить `is_in_vocab` для видимых/закэшированных токенов оригинала без обязательного полного переоткрытия документа. Панель перевода MUST NOT получать vocab highlight в этом change.

#### Scenario [RH-09]: Membership from Rust flags
- **GIVEN** Reader открыт, словарь содержит хотя бы одну morph-лемму, batch токенов готов
- **WHEN** система готовит токены оригинала к отрисовке
- **THEN** у токенов, чья lemma_key есть в словаре, `is_in_vocab` = true; у остальных = false; флаги приходят из compute-контракта, а не только из локального Dart-сравнения поверхностей

#### Scenario [RH-10]: Lemma key for EN wordforms
- **GIVEN** тексты реплик содержат словоформы «dogs» и «running»
- **WHEN** выполняется batch lemma compute для оригинала
- **THEN** lemma_key для них соответствуют морфологическим ключам «dog» и «run» (EN WordNet)

#### Scenario [RH-11]: Morph fail falls back to casefold
- **GIVEN** токен с кириллическим или иным surface, для которого EN morph не даёт леммы
- **WHEN** вычисляется lemma_key
- **THEN** lemma_key равен casefold(surface) (trim + lowercase / Unicode case-fold)

#### Scenario [RH-12]: Translation pane no highlight
- **GIVEN** Reader открыт в split с переводом
- **WHEN** пользователь смотрит панель перевода
- **THEN** vocab highlight UX этого change на панели перевода отсутствует

### Requirement: Paint vocab highlight on original single tokens
Система SHALL подсвечивать на панели оригинала каждый **word**-токен с `is_in_vocab` = true стилем vocab highlight (theme token light+dark, разумный green tint). Подсветка MUST быть per-token span, не целая реплика. Selection chrome и vocab highlight MAY сосуществовать. Multi-word (фразовые) записи словаря MUST NOT подсвечивать токены в этом change. Не-vocab токены MUST оставаться в стиле body по умолчанию.

#### Scenario [RH-01]: Saved lemma highlights wordforms
- **GIVEN** в словаре есть morph-лемма «run», в оригинале есть токены «running» и «ran» с lemma_key «run»
- **WHEN** пользователь просматривает оригинал после готового batch с membership
- **THEN** эти токены визуально подсвечены vocab highlight; другие токены без этой леммы не обязаны быть подсвечены

#### Scenario [RH-02]: Non-vocab tokens stay default
- **GIVEN** токен оригинала с `is_in_vocab` = false
- **WHEN** реплика отрисована
- **THEN** токен отображается стилем body по умолчанию (без vocab highlight)

#### Scenario [RH-08]: Phrase entries not highlighted
- **GIVEN** в словаре есть multi-word ключ (фраза), а в оригинале есть отдельные токены этих слов
- **WHEN** применяется membership/highlight
- **THEN** ни один токен не подсвечивается только из-за этой фразовой записи

#### Scenario [RH-03]: Refresh after save
- **GIVEN** Reader открыт; пользователь сохраняет новую single-token запись из Add-sheet
- **WHEN** save успешен
- **THEN** токены оригинала с той же morph lemma_key получают highlight без обязательного закрытия Reader

#### Scenario [RH-04]: Refresh after delete
- **GIVEN** токены подсвечены по лемме L; запись L удалена из edit-sheet **или** из Vocab (Reader уже не смонтирован)
- **WHEN** удаление применено
- **THEN** при живом кэше Reader — highlight для lemma_key L снимается через re-mark; если Reader не смонтирован — подсветка отсутствует при следующем открытии (tokenize→mark)

### Requirement: Tap in-vocab token opens edit sheet
Тап по токену оригинала с `is_in_vocab` = true MUST сразу открывать edit bottom sheet с prefilled полями записи (surface/lemma read-only по продуктовому контракту, translation и context editable) и MUST NOT показывать bottom CTA «Add to vocabulary» для этого жеста. Sheet MUST позволять сохранить изменения (upsert по morph lemma) и удалить запись. Пустой translation после trim при save MUST NOT персистить и MUST показать ошибку валидации. После успешного save или delete система MUST закрыть sheet и обновить highlight. Тап по токену с `is_in_vocab` = false остаётся в зоне `reader/tap` (selection + Add).

#### Scenario [RH-05]: Tap in_vocab opens edit sheet
- **GIVEN** токен оригинала подсвечен (`is_in_vocab` = true), selection для Add пуст
- **WHEN** пользователь тапает по этому токену
- **THEN** открывается edit-sheet с содержимым записи словаря; bottom CTA Add для этого жеста не показывается

#### Scenario [RH-06]: Edit sheet upsert
- **GIVEN** edit-sheet открыт для сохранённой леммы
- **WHEN** пользователь меняет translation и/или context и подтверждает с непустым translation
- **THEN** запись обновлена в словаре по morph lemma; sheet закрыт; highlight сохраняется

#### Scenario [RH-07]: Edit sheet delete
- **GIVEN** edit-sheet открыт для сохранённой леммы
- **WHEN** пользователь подтверждает удаление записи
- **THEN** записи нет в словаре; sheet закрыт; highlight этой леммы снят
