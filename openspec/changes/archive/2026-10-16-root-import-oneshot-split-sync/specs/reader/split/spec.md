# reader/split (delta) — one-shot pane sync

Связано с `reader/video-frames`, `reader/chapters`. Realtime scroll / chapter-pager sync снимается; остаётся одноразовое выравнивание при Show split и при успешной загрузке перевода.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RS-05 | Sync after load by nearest start time | Flutter widget: load translation → bottom near original start once |
| RS-06 | Scroll sync time then pixel on settle | Flutter widget: scroll settle → other pane **unchanged** (no realtime; title kept for MODIFIED identity) |
| RS-14 | Chapter pager syncs both panes | Flutter widget: top next → bottom **stays** (independent; title kept for MODIFIED identity) |
| RS-15 | Assume matching chapter counts | Flutter widget: after Show align, panes may diverge via pager |
| RS-16 | Mismatched chapter counts on translation load | Flutter: unequal N → clamp + visible error; one-shot still runs |
| RS-17 | Split divider drag does not rebuild paragraph lists | unchanged intent |
| RS-18 | Show split one-shot syncs translation | Flutter widget: Show with cues → bottom aligns once |
| RS-19 | Show split one-shot aligns EPUB chapters | Flutter widget: Show → bottom chapter = top once |
| RS-20 | Load EPUB translation one-shot aligns chapters | Flutter widget: load epub → bottom chapter matches top once |
| RS-12 | Reopen resumes at saved original anchor | resume unchanged; no auto sync until Show |

## MODIFIED Requirements

### Requirement: Synchronized scroll by time then pixel settle
В режиме split для **subtitle**-документов при наличии cues в обеих панелях система SHALL выполнять **одноразовое** выравнивание нижней панели к верхней по ближайшему `CueDto.startMs` (`BigInt`) **только** когда: (1) пользователь **включает** (Show) split при уже доступных cues перевода, или (2) пользователь **успешно загружает** перевод при уже включённом split. После этого выравнивания система MUST NOT синхронизировать панели при жестах прокрутки, settle, fling или programmatic scroll (кроме самого one-shot jump). Пиксельная подстройка на one-shot MAY применяться один раз. Для **EPUB** book-режима это требование MUST NOT применяться (см. one-shot chapter align ниже). Смена video page MUST NOT вызывать sync (`reader/video-frames`).

#### Scenario [RS-05]: Sync after load by nearest start time
- **GIVEN** верхняя панель subtitle-оригинала прокручена так, что якорная реплика имеет известный `start`, split включён, нижняя панель пуста
- **WHEN** пользователь успешно загружает перевод с несколькими cues с разными `start`
- **THEN** нижняя панель один раз выравнивается к реплике с ближайшим `start` к якорю верха

#### Scenario [RS-06]: Scroll sync time then pixel on settle
- **GIVEN** обе панели содержат subtitle cues, split включён, one-shot уже выполнен или панели уже видны
- **WHEN** пользователь прокручивает одну панель и жест завершается (settle)
- **THEN** позиция другой панели не меняется из‑за этого жеста (realtime sync отсутствует)

#### Scenario [RS-18]: Show split one-shot syncs translation
- **GIVEN** Reader в одной панели, оригинал прокручен не к началу, в сессии уже есть cues перевода
- **WHEN** пользователь включает split
- **THEN** нижняя панель один раз выравнивается к ближайшему `start` якоря верха; последующий скролл любой панели не двигает другую

### Requirement: Resume reading position when reopening original
Если у оригинального документа сохранён якорь чтения (persist — `library/documents`), система SHALL восстановить позицию **по формату**:
- **subtitle:** прокрутить панель оригинала к реплике с ближайшим `CueDto.startMs` к сохранённому якорю (`BigInt`) и применить alignment при наличии; выравнивание нижней панели перевода MUST NOT выполняться при одном лишь open — только при последующем Show split / load перевода (см. one-shot выше).
- **EPUB / book-режим:** показать главу с 0-based индексом из якоря (`lastReadStartMs` format-gated; `reader/chapters` RC-06 / LD-28); MUST NOT вызывать time-nearest по абзацам.
Если якоря нет или контент пуст — открытие с начала (первая глава / начало списка). Persist режима split и divider не требуется. Сбой якоря MUST NOT блокировать open.

#### Scenario [RS-12]: Reopen resumes at saved original anchor
- **GIVEN** пользователь ранее читал оригинал не с начала, якорь сохранён, приложение снова открывает этот оригинал в Reader
- **WHEN** Reader показывает оригинал
- **THEN** для subtitle список cues показывает область сохранённого якоря; для EPUB — главу с сохранённым chapter index (не обязательно первую), без интерпретации якоря как cue `startMs`; нижняя панель не выравнивается, пока пользователь не включит split

### Requirement: Synchronize EPUB panes by chapter index
В book-режиме split при наличии EPUB-содержимого в обеих панелях система SHALL выполнять **одноразовое** выравнивание индекса главы нижней панели к верхней **только** при Show split (если перевод уже в сессии) или при успешной загрузке EPUB-перевода при включённом split. После one-shot каждая панель MUST иметь **независимый** индекс главы: pager prev/next на одной панели MUST NOT менять главу другой. При **неравном** N на load перевода система MUST показать заметную ошибку и MAY clamp индекс в допустимый диапазон каждой панели. Скролл списка абзацев внутри главы не обязан зеркалироваться.

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
- **GIVEN** EPUB оригинал на главе k, в сессии есть EPUB перевод, split выключен
- **WHEN** пользователь включает split
- **THEN** нижняя панель один раз показывает главу k (или clamp, если у перевода меньше глав); дальнейшее переключение pager наверху не меняет низ

#### Scenario [RS-20]: Load EPUB translation one-shot aligns chapters
- **GIVEN** split включён, EPUB оригинал на главе k, нижняя панель пуста
- **WHEN** пользователь успешно загружает валидный EPUB-перевод
- **THEN** нижняя панель один раз открывается на главе k (или clamp); последующий pager одной панели не двигает другую

### Requirement: Book split chrome without per-drag paragraph rebuild
В book-режиме split перетаскивание divider MUST изменять высоты панелей без полного пересоздания списков абзацев на каждый pointer-move. Независимые индексы глав и независимый scroll абзацев сохраняются. Subtitle split divider MAY сохранить прежнее поведение, если book path изолирован.

#### Scenario [RS-17]: Split divider drag does not rebuild paragraph lists
- **GIVEN** EPUB split с загруженным переводом, обе панели показывают абзацы
- **WHEN** пользователь перетаскивает split divider
- **THEN** высоты панелей меняются, а списки абзацев не проходят полный rebuild на каждый шаг жеста (допустима лёгкая подстройка layout)
