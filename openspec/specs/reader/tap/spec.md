# reader/tap Specification

## Purpose

Даёт пользователю в Reader выделять слова и смежные фразы в тексте оригинала тапами по токенам, собирать contiguous selection и через bottom sheet сохранять выбранный фрагмент в личный словарь с ручным переводом и контекстом реплики.

Связанные capabilities: отображение cues — `reader/cues` / split — `reader/split`; persist словаря — `dictionary/entries`. Морфологическая лемматизация, offline translate и lemma highlight — отдельные future changes.

## Requirements

### Requirement: Batch tokenize original cues for Reader
При открытии Reader система SHALL один раз вычислить токены для всех реплик **оригинала** документа (batch) и держать их доступными для отрисовки и hit-test. Система MUST NOT выполнять отдельный tokenize-вызов на каждый rebuild строки списка. Пока batch не завершён, реплики оригинала MAY отображаться как plain text без token selection. При сбое вызова tokenize (FRB/bridge/orchestration) система MUST сохранить читаемый Reader (plain text) и показать user-visible сообщение об ошибке; MUST NOT требовать закрытия Library. Панель перевода MUST NOT получать token selection в этом change.

#### Scenario [RT-01]: Batch tokens ready for original cues
- **GIVEN** документ оригинала успешно открыт в Reader с одной или более cues
- **WHEN** пользователь просматривает список оригинала после успешного batch tokenize
- **THEN** у реплик оригинала доступны токены/spans для hit-test без повторной токенизации каждой строки при обычном rebuild списка

#### Scenario [RT-13]: Tokenize call failure degrades Reader
- **GIVEN** документ оригинала открыт в Reader, но batch tokenize завершился ошибкой вызова
- **WHEN** пользователь остаётся на экране Reader
- **THEN** список оригинала остаётся читаемым как plain text, token selection / bottom CTA для словаря недоступны, и показано user-visible сообщение об ошибке (SnackBar)

#### Scenario [RT-08]: Translation pane has no token selection
- **GIVEN** Reader открыт в split с загруженным переводом
- **WHEN** пользователь взаимодействует с текстом на панели перевода
- **THEN** не возникает selection слов/фраз для добавления в словарь (панель перевода остаётся без token-select UX этого change)

### Requirement: Tap selection of contiguous tokens in one cue
Система SHALL позволять выделять слова оригинала **только тапом** по токенам внутри **одной** реплики. Первый тап выбирает слово-якорь и показывает bottom CTA. Тап по **смежному** снаружи текущего диапазона слову той же реплики расширяет contiguous selection. Тап по слову **внутри** уже выделенного диапазона длины ≥ 2 сужает selection до этого слова (новый одиночный якорь). Повторный тап по единственному уже выбранному слову MUST NOT менять selection и MUST оставить CTA. Тап по **несмежному** слову той же реплики MUST сбросить selection к новому якорю и дать краткий визуальный фидбек сброса (~300ms). Selection MUST NOT объединять токены из разных реплик: тап в другой реплике начинает новое выделение только там. Drag-select в этом change не требуется.

#### Scenario [RT-02]: Single tap shows selection and CTA
- **GIVEN** Reader открыт, токены оригинала доступны, selection пуст
- **WHEN** пользователь тапает по одному слову в реплике оригинала
- **THEN** это слово выделено и видна bottom CTA для добавления в словарь

#### Scenario [RT-03]: Contiguous tap extends selection
- **GIVEN** в реплике уже выделено одно слово
- **WHEN** пользователь тапает по соседнему (смежному) слову той же реплики
- **THEN** selection становится contiguous диапазоном обоих слов, CTA остаётся доступной

#### Scenario [RT-04]: Non-contiguous tap resets anchor
- **GIVEN** в реплике выделены слова, образующие contiguous диапазон
- **WHEN** пользователь тапает по слову той же реплики с gap относительно текущего диапазона
- **THEN** предыдущий диапазон сбрасывается; выбран только новый якорь; пользователь видит краткий визуальный фидбек, что selection перезапущен; CTA доступна для нового якоря

#### Scenario [RT-09]: Selection stays within one cue
- **GIVEN** в реплике A есть активный selection
- **WHEN** пользователь тапает по слову в другой реплике B оригинала
- **THEN** selection в A очищается и начинается новое выделение только в B

#### Scenario [RT-10]: Phrase surface is contiguous cue substring
- **GIVEN** в одной реплике выделены два или более смежных слова
- **WHEN** система формирует текст фразы для sheet / сохранения
- **THEN** текст соответствует contiguous фрагменту текста реплики от начала первого до конца последнего выбранного токена (включая исходные разделители между ними)

### Requirement: Bottom sheet to add selection to dictionary
По нажатию bottom CTA система SHALL открывать bottom sheet с: (1) текстом выбранного слова/фразы (read-only surface), (2) полем перевода (обязательное), (3) полем контекста, предварительно заполненным полным текстом текущей реплики и **editable** (можно изменить или очистить). Подтверждение с непустым переводом MUST сохранять запись через общий dictionary persist (`dictionary/entries`, upsert по нормализованному ключу). Пустой перевод после trim MUST NOT создавать запись и MUST показать ошибку валидации. После успешного save система MUST закрыть sheet и очистить selection (bottom CTA скрывается). Автоперевод и подсветка lemma после save в этом change не требуются.

#### Scenario [RT-05]: CTA opens sheet with prefilled fields
- **GIVEN** есть contiguous selection в одной реплике оригинала
- **WHEN** пользователь нажимает bottom CTA
- **THEN** открывается bottom sheet: surface/phrase заполнены из selection, перевод пуст, context = полный текст этой реплики и доступен для редактирования

#### Scenario [RT-06]: Save from sheet adds dictionary entry
- **GIVEN** sheet открыт с непустым selection surface
- **WHEN** пользователь вводит непустой перевод и подтверждает сохранение
- **THEN** запись появляется в словаре (доступна через list/lookup Vocabulary) с ключом = нормализованный surface/фраза, указанным переводом и (опционально отредактированным) context; sheet закрыт; selection очищен (bottom CTA скрыта)

#### Scenario [RT-07]: Reject empty translation from Reader sheet
- **GIVEN** sheet открыт
- **WHEN** пользователь подтверждает сохранение с пустым переводом (после trim)
- **THEN** запись не создаётся и показывается ошибка валидации

### Requirement: Token spans for hit-test
Rust tokenize compute SHALL принимать для каждой реплики тот же display-string, что рисует Reader (`text_lines` / `textLines`, соединённые одним `\n`), и возвращать упорядоченные **word**-токены с `surface` и **char** span (`char_start` / `char_end`, exclusive end) такими, что `surface` совпадает с соответствующим срезом этого текста. Публичный FRB batch — `tokenize_cues(texts: Vec<String>) -> Vec<CueTokensDto>` (по одному результату на вход, поле `cue_index` = индекс во входном векторе). Токенизация MUST корректно обрабатывать кириллицу и сохранять типичные EN-формы с апострофом (например `don't`) как одно слово, где tokenizer это предусматривает. Byte-only индексы без char-индексов для Flutter hit-test MUST NOT быть единственным контрактом. Морфологическая лемма и Rust-persist словаря в этом requirement не требуются.

#### Scenario [RT-11]: Tokenize Cyrillic and apostrophe words
- **GIVEN** display-текст реплики (lines joined by `\n`) содержит кириллические слова и/или форму с апострофом вроде `don't`
- **WHEN** выполняется batch `tokenize_cues` для оригинала
- **THEN** кириллические слова выделяются как отдельные word-токены, а `don't` — как один word-токен (не разбитый на части по апострофу)

#### Scenario [RT-12]: Span surface invariant
- **GIVEN** произвольная непустая реплика с словами (тот же display-string, что для paint)
- **WHEN** `tokenize_cues` / tokenize возвращает word-токены
- **THEN** для каждого word-токена срез текста реплики по `char_start..char_end` равен `surface`

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RT-01 | Batch tokens ready for original cues | Flutter: after Reader open, original cues have token spans without per-row tokenize on rebuild |
| RT-02 | Single tap shows selection and CTA | Flutter widget: tap word → selected + bottom CTA visible |
| RT-03 | Contiguous tap extends selection | Flutter widget: tap adjacent word → both selected, CTA stays |
| RT-04 | Non-contiguous tap resets anchor | Flutter widget: tap with gap → only new word selected; brief visual feedback |
| RT-05 | CTA opens sheet with prefilled fields | Flutter widget: CTA → sheet: surface/phrase, empty translation, context = cue text (editable) |
| RT-06 | Save from sheet adds dictionary entry | Flutter widget/integration: confirm with translation → entry in dictionary persist; sheet closed; selection cleared |
| RT-07 | Reject empty translation from Reader sheet | Flutter widget: empty translation → no persist, validation error |
| RT-08 | Translation pane has no token selection | Flutter widget: split with translation → taps do not create word selection on bottom pane |
| RT-09 | Selection stays within one cue | Flutter widget: selecting in cue A then tapping cue B starts new selection only in B |
| RT-10 | Phrase surface is contiguous cue substring | Flutter: multi-word contiguous selection uses cue substring spanning first→last token (not naive space-join only) |
| RT-11 | Tokenize Cyrillic and apostrophe words | Rust `cargo test`: Cyrillic words + `don't` as single word tokens with valid char spans |
| RT-12 | Span surface invariant | Rust `cargo test`: each word token surface equals cue[char_start..char_end] |
| RT-13 | Tokenize call failure degrades Reader | Flutter widget: failing batch tokenize → cues readable as plain text, no token CTA, user-visible SnackBar |
