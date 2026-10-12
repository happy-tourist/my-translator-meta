Связь: highlight / edit in_vocab — `reader/highlight`; persist — `dictionary/entries`. Delta восстанавливает contiguous phrase Add (как tap-vocab) и фиксирует ключи save: morph для single, casefold(join) для phrase.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RT-02 | Single tap shows selection and CTA | Flutter: tap non-vocab → selected + Add CTA |
| RT-03 | Contiguous tap extends selection | Flutter: tap adjacent non-vocab → expand contiguous range |
| RT-04 | Non-contiguous tap resets anchor | Flutter: gap tap → only new word; brief ~300ms flash |
| RT-05 | CTA opens sheet with prefilled fields | Flutter: CTA → surface = selection (single or phrase join); translation empty; context = cue |
| RT-06 | Save from sheet adds dictionary entry | Flutter: single → morph lemma; phrase → casefold(join tokens) |
| RT-07 | Reject empty translation from Reader sheet | Flutter: empty translation → no persist, validation |
| RT-09 | Selection stays within one cue | Flutter: tap other cue → selection only there |
| RT-10 | Phrase surface is token-join | Flutter: multi-token surface = casefold(join token surfaces with single space) |
| RT-14 | Contiguous multi-token Add available | Flutter: can form multi-token selection and save phrase |
| RT-15 | In-vocab tap does not show Add CTA | Flutter: tap in_vocab → no Add CTA (edit via highlight) |
| RT-16 | Shrink multi selection to single | Flutter: tap inside range length≥2 → shrink to that token |

## MODIFIED Requirements

### Requirement: Tap selection of contiguous tokens in one cue
Система SHALL позволять выделять word-токены оригинала **только тапом** внутри **одной** реплики для пути Add. Первый тап по токену с `is_in_vocab` = false выбирает якорь и показывает bottom CTA. Тап по **смежному** снаружи текущего диапазона non-vocab слову той же реплики MUST расширять contiguous selection. Тап по слову **внутри** уже выделенного диапазона длины ≥ 2 MUST сужать selection до этого слова (новый одиночный якорь). Повторный тап по единственному уже выбранному non-vocab токену MUST NOT менять selection и MUST оставить CTA. Тап по **несмежному** non-vocab слову той же реплики MUST сбросить selection к новому якорю и дать краткий визуальный фидбек сброса (~300ms). Selection MUST NOT объединять токены из разных реплик. Drag-select не требуется. Тап по `is_in_vocab` = true — см. `reader/highlight` (edit, без Add CTA); Add selection при этом MUST очищаться.

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
