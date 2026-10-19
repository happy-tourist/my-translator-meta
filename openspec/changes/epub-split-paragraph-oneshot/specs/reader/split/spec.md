# reader/split (delta) — EPUB mid-chapter one-shot

Связано с `reader/chapters`. Субтитры one-shot по `startMs` без изменений intent. EPUB: one-shot = глава **и** абзац; preserve mid-chapter scroll оригинала на Show/Hide.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RS-10 | Show split preserves original scroll | Flutter widget: EPUB mid-chapter → Show → top still mid (not para 0); subtitle cue path regression |
| RS-11 | Hide split preserves original scroll | Flutter widget: EPUB split top mid → Hide → single pane still mid |
| RS-19 | Show split one-shot aligns EPUB chapters | Flutter widget: Show → bottom chapter **and** paragraph align to top once |
| RS-20 | Load EPUB translation one-shot aligns chapters | Flutter widget: load epub → bottom chapter **and** paragraph match top once |
| RS-21 | After EPUB one-shot paragraph scroll independent | Flutter widget: scroll one pane paragraphs → other unchanged |
| RS-22 | Unequal paragraph counts clamp on one-shot | Flutter widget: fewer translation paras → clamp index; no crash |

## MODIFIED Requirements

### Requirement: Toggle horizontal split on Reader
Система SHALL позволять на экране Reader включать и выключать горизонтальное разделение экрана: верхняя область — cues оригинального документа **или** book-список абзацев текущей главы; нижняя — область перевода. По умолчанию Reader MUST открываться в режиме одной панели (без split). При выключении split система MUST сохранять загруженный перевод в рамках текущей сессии Reader (не сбрасывать cues/главы до явного удаления или выхода из Reader). При **включении и выключении** split система MUST сохранять позицию прокрутки верхней панели оригинала в рамках текущей сессии Reader: для субтитров — cue/якорь списка; для EPUB book-режима — **позицию внутри текущей главы** (не сбрасывать список абзацев к началу главы из‑за перестройки layout). Persist режима split on/off между сессиями приложения не требуется.

#### Scenario [RS-01]: Toggle split shows empty bottom pane
- **GIVEN** Reader открыт для оригинального документа без привязанного перевода
- **WHEN** пользователь включает режим split
- **THEN** экран делится горизонтально: сверху видны cues оригинала **или** book-список абзацев текущей главы, снизу — пустое состояние с действием загрузки файла

#### Scenario [RS-02]: Hide split returns to single pane
- **GIVEN** split включён и в нижней панели уже загружен перевод (subtitle cues **или** EPUB главы/абзацы)
- **WHEN** пользователь выключает split
- **THEN** показывается только одна панель с оригиналом, приложение не падает, и при повторном включении split перевод снова доступен без повторной загрузки в той же сессии

#### Scenario [RS-10]: Show split preserves original scroll
- **GIVEN** Reader в режиме одной панели: для субтитров список оригинала прокручен так, что видна не первая реплика; **или** для EPUB — текущая глава прокручена так, что виден не первый абзац
- **WHEN** пользователь включает режим split
- **THEN** верхняя панель продолжает показывать ту же (или эквивалентную по индексу/якорю) область оригинала, а не сбрасывается к началу списка / началу главы

#### Scenario [RS-11]: Hide split preserves original scroll
- **GIVEN** Reader в режиме split, верхняя панель прокручена так, что видна не первая реплика оригинала (субтитры) **или** не первый абзац текущей главы (EPUB)
- **WHEN** пользователь выключает режим split
- **THEN** единственная панель оригинала продолжает показывать ту же (или эквивалентную по индексу/якорю) область, а не сбрасывается к началу списка / началу главы

### Requirement: Synchronize EPUB panes by chapter index
В book-режиме split при наличии EPUB-содержимого в обеих панелях система SHALL выполнять **одноразовое** выравнивание нижней панели к верхней **только** при Show split (если перевод уже в сессии) или при успешной загрузке EPUB-перевода при включённом split. One-shot MUST выставить: (1) индекс главы низа = clamp(индекс главы верха); (2) позицию списка абзацев низа к якорю верха — по **индексу абзаца** (и MAY пиксельному alignment) с clamp, если в главе перевода меньше абзацев. После one-shot каждая панель MUST иметь **независимый** индекс главы и **независимый** скролл абзацев: pager prev/next и прокрутка абзацев на одной панели MUST NOT менять другую. При **неравном** N глав на load перевода система MUST показать заметную ошибку и MAY clamp индекс главы (как ранее). Realtime зеркалирование абзацев MUST NOT выполняться.

#### Scenario [RS-14]: Chapter pager syncs both panes
- **GIVEN** split включён, обе панели показывают EPUB с одинаковым числом глав N ≥ 2, после one-shot обе на главе 1
- **WHEN** пользователь переключает верхнюю панель на главу 2
- **THEN** нижняя панель остаётся на главе 1 (realtime pager sync отсутствует)

#### Scenario [RS-15]: Assume matching chapter counts
- **GIVEN** EPUB оригинал и EPUB перевод с одинаковым числом глав, one-shot уже выставил обе панели на одну главу
- **WHEN** пользователь листает главы только в одной панели
- **THEN** другая панель сохраняет свой индекс; ошибки несовпадения структуры нет

#### Scenario [RS-16]: Mismatched chapter counts on translation load
- **GIVEN** split включён на EPUB-оригинале с N глав, пользователь загружает валидный EPUB-перевод с M глав, M ≠ N
- **WHEN** загрузка/привязка перевода завершается
- **THEN** пользователь видит заметную ошибку о несовпадении структуры; перевод MAY остаться доступным с clamp индекса one-shot, но MUST NOT молча притворяться, что N = M

#### Scenario [RS-19]: Show split one-shot aligns EPUB chapters
- **GIVEN** EPUB оригинал на главе k, список абзацев прокручен к непервому абзацу p, в сессии есть EPUB перевод, split выключен
- **WHEN** пользователь включает split
- **THEN** нижняя панель один раз показывает главу k (или clamp) **и** область абзаца, эквивалентную якорю p (или clamp, если у перевода в главе меньше абзацев); дальнейшее переключение pager или скролл абзацев наверху не двигает низ

#### Scenario [RS-20]: Load EPUB translation one-shot aligns chapters
- **GIVEN** split включён, EPUB оригинал на главе k с видимым непервым абзацем p, нижняя панель пуста
- **WHEN** пользователь успешно загружает валидный EPUB-перевод
- **THEN** нижняя панель один раз открывается на главе k (или clamp) **и** выравнивается к абзацу, эквивалентному p (или clamp); последующий pager или скролл абзацев одной панели не двигает другую

#### Scenario [RS-21]: After EPUB one-shot paragraph scroll independent
- **GIVEN** EPUB split с переводом, one-shot уже выполнен, обе панели на одной главе
- **WHEN** пользователь прокручивает список абзацев только верхней панели
- **THEN** позиция списка абзацев нижней панели не меняется из‑за этого жеста

#### Scenario [RS-22]: Unequal paragraph counts clamp on one-shot
- **GIVEN** EPUB оригинал на главе k прокручен к абзацу с индексом p, перевод в сессии на главе k имеет M абзацев, M ≤ p, split выключен
- **WHEN** пользователь включает split
- **THEN** нижняя панель показывает главу k (или clamp по главам) и видимую область около последнего доступного абзаца (clamp), приложение не падает
