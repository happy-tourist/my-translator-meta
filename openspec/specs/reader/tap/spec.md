# reader/tap Specification

## Purpose

Даёт пользователю в Reader выделять слова и смежные фразы в тексте оригинала тапами по токенам, собирать contiguous selection и через bottom sheet сохранять выбранный фрагмент в личный словарь с ручным переводом и контекстом реплики.

Связанные capabilities: отображение cues — `reader/cues` / split — `reader/split`; persist словаря — `dictionary/entries`; highlight / edit in_vocab — `reader/highlight`.

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
Система SHALL позволять выделять word-токены оригинала **только тапом** внутри **одной** реплики для пути Add. Первый тап по токену с `is_in_vocab` = false выбирает якорь и показывает bottom CTA. Тап по **смежному** снаружи текущего диапазона non-vocab слову той же реплики MUST расширять contiguous selection. Тап по слову **внутри** уже выделенного диапазона длины ≥ 2 MUST сужать selection до этого слова (новый одиночный якорь). Повторный тап по единственному уже выбранному non-vocab токену MUST снять selection и скрыть CTA. Тап вне токена MUST NOT снимать selection. Тап по **несмежному** non-vocab слову той же реплики MUST сбросить selection к новому якорю и дать краткий визуальный фидбек сброса (~300ms). Selection MUST NOT объединять токены из разных реплик. Drag-select не требуется. Тап по `is_in_vocab` = true — см. `reader/highlight` (edit, без Add CTA); Add selection при этом MUST очищаться. При открытии Add/Edit bottom sheet система SHOULD не давать списку оригинала визуально прыгать к началу (`resizeToAvoidBottomInset: false` и/или restore scroll якоря).

#### Scenario [RT-02]: Single tap shows selection and CTA
- **GIVEN** Reader открыт, selection пуст, целевой токен не в словаре
- **WHEN** пользователь тапает по одному такому слову
- **THEN** слово выделено и видна bottom CTA Add

#### Scenario [RT-03]: Contiguous tap extends selection
- **GIVEN** в реплике выделен один non-vocab токен
- **WHEN** пользователь тапает по соседнему non-vocab слову той же реплики
- **THEN** selection становится contiguous диапазоном обоих слов; CTA доступна

#### Scenario [RT-04]: Non-contiguous tap resets anchor
- **GIVEN** в реплике есть contiguous selection
- **WHEN** пользователь тапает по несмежному non-vocab слову той же реплики
- **THEN** selection сбрасывается к новому одиночному якорю; краткий visual flash ~300ms; CTA доступна

#### Scenario [RT-16]: Shrink multi selection to single
- **GIVEN** в реплике выделен contiguous диапазон длины ≥ 2
- **WHEN** пользователь тапает по токену внутри этого диапазона
- **THEN** selection сужается до этого одиночного токена; CTA остаётся

#### Scenario [RT-17]: Retap sole selected clears
- **GIVEN** выбран единственный non-vocab токен и видна Add CTA
- **WHEN** пользователь снова тапает по этому же токену
- **THEN** selection снят; CTA скрыта

#### Scenario [RT-09]: Selection stays within one cue
- **GIVEN** в реплике A есть активный selection
- **WHEN** пользователь тапает по non-vocab слову в реплике B
- **THEN** selection в A очищается; новое выделение только в B

#### Scenario [RT-10]: Phrase surface is token-join
- **GIVEN** contiguous multi-token selection в одной реплике
- **WHEN** формируется surface для Add-sheet / persist key
- **THEN** surface/key строится как casefold(join выбранных token surfaces через один пробел), не как обязательный cue substring

#### Scenario [RT-14]: Contiguous multi-token Add available
- **GIVEN** Reader открыт, токены готовы
- **WHEN** пользователь набирает contiguous multi-token selection и сохраняет через CTA/sheet
- **THEN** phrase Add доступен; запись создаётся с phrase identity (см. RT-06 / `dictionary/entries`)

#### Scenario [RT-15]: In-vocab tap does not show Add CTA
- **GIVEN** токен с `is_in_vocab` = true
- **WHEN** пользователь тапает по нему
- **THEN** bottom CTA Add для этого жеста не показывается (edit-flow `reader/highlight`)

### Requirement: Bottom sheet to add selection to dictionary
По нажатию bottom CTA система SHALL открывать bottom sheet с: (1) read-only surface выбранного фрагмента (один токен или phrase join), (2) обязательным translation, (3) context = полный текст реплики, editable. Подтверждение с непустым translation MUST сохранять через `dictionary/entries`: **single-token** → identity = morph `lemma_key`, `surface_key` = casefold(surface); **multi-token** → identity = casefold(join token surfaces), `surface_key` = тот же ключ (или casefold join — эквивалент). Пустой translation после trim MUST NOT персистить и MUST показать ошибку. После успешного save — закрыть sheet, очистить selection, обновить highlight (`reader/highlight`).

#### Scenario [RT-05]: CTA opens sheet with prefilled fields
- **GIVEN** non-empty non-vocab selection в одной реплике
- **WHEN** пользователь нажимает bottom CTA
- **THEN** sheet: surface = selection (single или join), перевод пуст, context = текст реплики editable

#### Scenario [RT-06]: Save from sheet adds dictionary entry
- **GIVEN** sheet открыт для single «Running» или для phrase из нескольких токенов
- **WHEN** пользователь вводит непустой перевод и подтверждает
- **THEN** для single — запись с morph lemma; для phrase — запись с casefold(join); sheet закрыт; selection очищен

#### Scenario [RT-07]: Reject empty translation from Reader sheet
- **GIVEN** sheet открыт
- **WHEN** сохранение с пустым translation после trim
- **THEN** запись не создаётся; показывается ошибка валидации

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
| RT-02 | Single tap shows selection and CTA | Flutter: tap non-vocab → selected + Add CTA |
| RT-03 | Contiguous tap extends selection | Flutter: tap adjacent non-vocab → expand contiguous range |
| RT-04 | Non-contiguous tap resets anchor | Flutter: gap tap → only new word; brief ~300ms flash |
| RT-05 | CTA opens sheet with prefilled fields | Flutter: CTA → surface = selection (single or phrase join); translation empty; context = cue |
| RT-06 | Save from sheet adds dictionary entry | Flutter: single → morph lemma; phrase → casefold(join tokens) |
| RT-07 | Reject empty translation from Reader sheet | Flutter: empty translation → no persist, validation |
| RT-08 | Translation pane has no token selection | Flutter widget: split with translation → taps do not create word selection on bottom pane |
| RT-09 | Selection stays within one cue | Flutter: tap other cue → selection only there |
| RT-10 | Phrase surface is token-join | Flutter: multi-token surface = casefold(join token surfaces with single space) |
| RT-11 | Tokenize Cyrillic and apostrophe words | Rust `cargo test`: Cyrillic words + `don't` as single word tokens with valid char spans |
| RT-12 | Span surface invariant | Rust `cargo test`: each word token surface equals cue[char_start..char_end] |
| RT-13 | Tokenize call failure degrades Reader | Flutter widget: failing batch tokenize → cues readable as plain text, no token CTA, user-visible SnackBar |
| RT-14 | Contiguous multi-token Add available | Flutter: can form multi-token selection and save phrase |
| RT-15 | In-vocab tap does not show Add CTA | Flutter: tap in_vocab → no Add CTA (edit via highlight) |
| RT-16 | Shrink multi selection to single | Flutter: tap inside range length≥2 → shrink to that token |
| RT-17 | Retap sole selected clears | Flutter: retap lone non-vocab → selection gone, CTA hidden |
