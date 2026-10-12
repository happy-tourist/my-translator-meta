Связь: lemma highlight / edit in_vocab — `reader/highlight`; morph identity persist — `dictionary/entries`. Этот delta сужает Add до single-token и переводит ключ сохранения на morph lemma.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RT-02 | Single tap shows selection and CTA | Flutter: tap non-vocab word → selected + Add CTA |
| RT-03 | Contiguous tap extends selection | Flutter: tap adjacent non-vocab → selection **replaces** with new single token (no multi-token range; Scenario title kept for MODIFIED continuity vs main) |
| RT-04 | Non-contiguous tap resets anchor | Flutter: tap another non-vocab in same cue → new single selection |
| RT-05 | CTA opens sheet with prefilled fields | Flutter: CTA → sheet surface = one token; translation empty; context = cue |
| RT-06 | Save from sheet adds dictionary entry | Flutter: save → dict entry identity = morph lemma_key of token |
| RT-07 | Reject empty translation from Reader sheet | Flutter: empty translation → no persist, validation error |
| RT-09 | Selection stays within one cue | Flutter: tap other cue → selection only there |
| RT-10 | Phrase surface is contiguous cue substring | Flutter: multi-token phrase Add unavailable; no phrase surface path |
| RT-14 | Multi-token Add not available | Flutter: cannot form multi-token selection / phrase Add |
| RT-15 | In-vocab tap does not show Add CTA | Flutter: tap in_vocab → no Add CTA (edit owned by reader/highlight) |

## MODIFIED Requirements

### Requirement: Tap selection of contiguous tokens in one cue
Система SHALL позволять выделять на оригинале **ровно один** word-токен тапом внутри **одной** реплики для пути Add. Первый тап по токену с `is_in_vocab` = false выбирает этот токен и показывает bottom CTA Add. Тап по **другому** токену той же реплики с `is_in_vocab` = false MUST заменить selection на этот одиночный токен (contiguous multi-token range MUST NOT формироваться). Повторный тап по уже выбранному одиночному non-vocab токену MUST NOT менять selection и MUST оставить CTA. Selection MUST NOT объединять токены из разных реплик: тап в другой реплике начинает новое одиночное выделение только там. Drag-select не требуется. Поведение тапа по `is_in_vocab` = true — см. `reader/highlight` (edit-sheet, без Add CTA).

#### Scenario [RT-02]: Single tap shows selection and CTA
- **GIVEN** Reader открыт, токены оригинала доступны, selection пуст, целевой токен не в словаре
- **WHEN** пользователь тапает по одному такому слову в реплике оригинала
- **THEN** это слово выделено и видна bottom CTA для добавления в словарь

#### Scenario [RT-03]: Contiguous tap extends selection
- **GIVEN** в реплике уже выделен один non-vocab токен
- **WHEN** пользователь тапает по соседнему non-vocab слову той же реплики
- **THEN** selection становится одиночным выделением нового слова (диапазон из двух слов не образуется); CTA остаётся доступной

#### Scenario [RT-04]: Non-contiguous tap resets anchor
- **GIVEN** в реплике выделен один non-vocab токен
- **WHEN** пользователь тапает по другому non-vocab слову той же реплики (не обязательно смежному)
- **THEN** предыдущий якорь сбрасывается; выделен только новый одиночный токен; CTA доступна

#### Scenario [RT-09]: Selection stays within one cue
- **GIVEN** в реплике A есть активный single-token selection
- **WHEN** пользователь тапает по non-vocab слову в другой реплике B оригинала
- **THEN** selection в A очищается и начинается новое одиночное выделение только в B

#### Scenario [RT-10]: Phrase surface is contiguous cue substring
- **GIVEN** Reader открыт, токены готовы
- **WHEN** пользователь взаимодействует с токенами оригинала для Add
- **THEN** multi-token / phrase selection для Add недоступна; contiguous phrase surface для сохранения не формируется (phrase save — later)

#### Scenario [RT-14]: Multi-token Add not available
- **GIVEN** Reader открыт, токены готовы
- **WHEN** пользователь пытается набрать contiguous multi-token selection для Add
- **THEN** multi-token selection / phrase Add недоступны; сохранение фразы из Reader — later

#### Scenario [RT-15]: In-vocab tap does not show Add CTA
- **GIVEN** токен оригинала с `is_in_vocab` = true
- **WHEN** пользователь тапает по нему
- **THEN** bottom CTA Add для этого жеста не показывается (открывается edit-flow `reader/highlight`)

### Requirement: Bottom sheet to add selection to dictionary
По нажатию bottom CTA система SHALL открывать bottom sheet с: (1) текстом выбранного **одного** токена (read-only surface), (2) полем перевода (обязательное), (3) полем контекста, предварительно заполненным полным текстом текущей реплики и **editable**. Подтверждение с непустым переводом MUST сохранять запись через общий dictionary persist (`dictionary/entries`) с identity = **morph lemma_key** выбранного токена (и surface_key = casefold surface по контракту словаря). Пустой перевод после trim MUST NOT создавать запись и MUST показать ошибку валидации. После успешного save система MUST закрыть sheet, очистить selection (bottom CTA скрывается) и инициировать обновление highlight (`reader/highlight`).

#### Scenario [RT-05]: CTA opens sheet with prefilled fields
- **GIVEN** есть одиночный non-vocab selection в одной реплике оригинала
- **WHEN** пользователь нажимает bottom CTA
- **THEN** открывается bottom sheet: surface = этот токен, перевод пуст, context = полный текст этой реплики и доступен для редактирования

#### Scenario [RT-06]: Save from sheet adds dictionary entry
- **GIVEN** sheet открыт для одиночного токена (например surface «Running»)
- **WHEN** пользователь вводит непустой перевод и подтверждает сохранение
- **THEN** запись появляется в словаре с morph lemma identity (для EN wordform — морфологический ключ, напр. «run»), указанным переводом и context; sheet закрыт; selection очищен

#### Scenario [RT-07]: Reject empty translation from Reader sheet
- **GIVEN** sheet открыт
- **WHEN** пользователь подтверждает сохранение с пустым переводом (после trim)
- **THEN** запись не создаётся и показывается ошибка валидации
