## Purpose

Подсвечивает на панели оригинала (1) single-token вхождения morph-лемм и (2) consecutive phrase spans по ключу casefold(join token surfaces). Тап по подсвеченному открывает edit-sheet (слово или фраза). Membership считает Rust; Flutter красит и показывает chrome.

Связанные capabilities: selection/Add — `reader/tap`; persist — `dictionary/entries`.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RH-01 | Saved lemma highlights wordforms | Flutter: entry «run» → «running»/«ran» highlighted |
| RH-02 | Non-vocab tokens stay default | Flutter: unmarked → default body |
| RH-03 | Refresh after save | Flutter: after Add (single or phrase) → matching spans gain highlight |
| RH-04 | Refresh after delete | Flutter: edit delete → re-mark; Vocab delete while unmounted → clear on next open |
| RH-05 | Tap in_vocab opens edit sheet | Flutter: tap highlighted → edit, no Add CTA; resolves longest phrase > shorter > single |
| RH-06 | Edit sheet upsert | Flutter: save translation/context → upsert by dict key |
| RH-07 | Edit sheet delete | Flutter: delete → entry gone, highlight clears |
| RH-08 | Phrase entry highlights all span tokens | Flutter: multi-word dict key paints every token in the matched consecutive span |
| RH-09 | Membership from Rust flags | Rust/Flutter: paint uses compute `is_in_vocab`, not Dart-only Set |
| RH-10 | Lemma key for EN wordforms | Rust: dogs/running → dog/run |
| RH-11 | Morph fail falls back to casefold | Rust: Cyrillic/junk → casefold(surface) |
| RH-12 | Translation pane no highlight | Flutter: translation pane no vocab highlight |
| RH-13 | Overlapping phrases both paint | Flutter: two overlapping phrase keys → all tokens of both spans highlighted |
| RH-14 | Phrase match is token-join not substring | Rust/Flutter: match casefold(join surfaces); punct between tokens does not require cue substring equality |

## ADDED Requirements

### Requirement: Batch lemma keys and vocab membership on original tokens
При открытии Reader система SHALL для токенов оригинала вычислить **lemma_key** (morph для single-word surfaces) и выставить **is_in_vocab** по словарю пользователя: (a) morph equality для single-token keys без whitespace; (b) consecutive token windows, чьё casefold(join surfaces через один пробел) совпадает с multi-word dict key. Membership MUST вычисляться на стороне Rust compute (FRB); Flutter MUST NOT быть единственным источником истины для флага подсветки. Пакетность: один batch на открытие (как tokenize); MUST NOT per-token FFI на каждый rebuild. После save/delete/migrate система MUST обновить flags на кэше без обязательного переоткрытия документа. Панель перевода MUST NOT получать vocab highlight.

#### Scenario [RH-09]: Membership from Rust flags
- **GIVEN** Reader открыт, словарь содержит хотя бы одну запись, batch токенов готов
- **WHEN** система готовит токены оригинала к отрисовке
- **THEN** токены, входящие в morph hit или phrase span hit, имеют `is_in_vocab` = true из compute-контракта; остальные = false

#### Scenario [RH-10]: Lemma key for EN wordforms
- **GIVEN** тексты реплик содержат словоформы «dogs» и «running»
- **WHEN** выполняется batch lemma compute для оригинала
- **THEN** lemma_key для них соответствуют морфологическим ключам «dog» и «run» (EN WordNet)

#### Scenario [RH-11]: Morph fail falls back to casefold
- **GIVEN** токен с кириллическим или иным surface, для которого EN morph не даёт леммы
- **WHEN** вычисляется lemma_key
- **THEN** lemma_key равен casefold(surface)

#### Scenario [RH-12]: Translation pane no highlight
- **GIVEN** Reader открыт в split с переводом
- **WHEN** пользователь смотрит панель перевода
- **THEN** vocab highlight UX на панели перевода отсутствует

#### Scenario [RH-14]: Phrase match is token-join not substring
- **GIVEN** в словаре ключ «hello world»; в реплике токены «hello» и «world» с пунктуацией между ними в сыром тексте
- **WHEN** применяется membership
- **THEN** оба токена получают `is_in_vocab` = true по casefold(join), без требования равенства cue substring

### Requirement: Paint vocab highlight on original tokens (single and phrase)
Система SHALL подсвечивать на панели оригинала каждый word-токен с `is_in_vocab` = true стилем vocab highlight (theme light+dark). Подсветка MUST быть per-token span. Selection chrome и vocab highlight MAY сосуществовать. Если токен входит в одну или несколько matched phrase spans и/или morph hit — он MUST быть подсвечен (overlaps: красить все затронутые токены). Не-vocab токены MUST оставаться в стиле body по умолчанию.

#### Scenario [RH-01]: Saved lemma highlights wordforms
- **GIVEN** в словаре morph-лемма «run»; в оригинале токены «running» и «ran» с lemma_key «run»
- **WHEN** batch membership готов
- **THEN** эти токены подсвечены vocab highlight

#### Scenario [RH-02]: Non-vocab tokens stay default
- **GIVEN** токен с `is_in_vocab` = false
- **WHEN** реплика отрисована
- **THEN** токен без vocab highlight

#### Scenario [RH-08]: Phrase entry highlights all span tokens
- **GIVEN** в словаре multi-word ключ (фраза); в оригинале consecutive токены, чей casefold(join) равен ключу
- **WHEN** применяется membership/highlight
- **THEN** каждый токен этого span подсвечен

#### Scenario [RH-13]: Overlapping phrases both paint
- **GIVEN** две overlapping phrase-записи в словаре и оба span находятся в реплике
- **WHEN** применяется membership/highlight
- **THEN** токены обоих spans подсвечены (обе фразы видны)

#### Scenario [RH-03]: Refresh after save
- **GIVEN** Reader открыт; пользователь сохраняет single или phrase из Add-sheet
- **WHEN** save успешен
- **THEN** matching токены получают highlight без обязательного закрытия Reader

#### Scenario [RH-04]: Refresh after delete
- **GIVEN** токены подсвечены по ключу K; запись K удалена из edit-sheet или Vocab (Reader не смонтирован)
- **WHEN** удаление применено
- **THEN** при живом кэше — re-mark снимает highlight; если Reader не смонтирован — нет highlight при следующем открытии

### Requirement: Tap in-vocab opens edit sheet (single or phrase)
Тап по токену с `is_in_vocab` = true MUST открывать edit bottom sheet для **одной** записи словаря и MUST NOT показывать Add CTA. Если токен принадлежит нескольким hits, система MUST выбрать запись по приоритету: **самая длинная phrase span (по числу токенов) > более короткая phrase > single morph**. Sheet: surface/lemma read-only по контракту, translation/context editable; upsert/delete; пустой translation после trim — ошибка без persist. После save/delete — закрыть sheet и обновить highlight. Тап по `is_in_vocab` = false — зона `reader/tap`.

#### Scenario [RH-05]: Tap in_vocab opens edit sheet
- **GIVEN** токен подсвечен; при overlaps применим приоритет longest > shorter > single
- **WHEN** пользователь тапает по токену
- **THEN** открывается edit-sheet выбранной записи; Add CTA для жеста нет

#### Scenario [RH-06]: Edit sheet upsert
- **GIVEN** edit-sheet открыт для сохранённого ключа (morph или phrase)
- **WHEN** пользователь меняет translation и/или context и подтверждает с непустым translation
- **THEN** запись обновлена по этому ключу; sheet закрыт; highlight сохраняется

#### Scenario [RH-07]: Edit sheet delete
- **GIVEN** edit-sheet открыт
- **WHEN** пользователь подтверждает удаление
- **THEN** записи нет; sheet закрыт; highlight этого ключа снят
