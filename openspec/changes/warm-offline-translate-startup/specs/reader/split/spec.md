# Delta: reader/split

Базовый контракт: `openspec/specs/reader/split/spec.md`. Убрать one-shot sync; preserve/persist низ; text-only bottom; Delete у split в AppBar.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RS-04…RS-24 | see scenarios below | Flutter widget + Drift as noted |

## MODIFIED Requirements

### Requirement: Toggle horizontal split on Reader
Система SHALL позволять включать/выключать горизонтальный split (верх — оригинал, низ — перевод). По умолчанию — одна панель. При выключении split перевод остаётся в сессии до delete/exit. При **включении и выключении** split система MUST сохранять позицию прокрутки **верхней** панели оригинала и, если перевод загружен, позицию **нижней** панели (без sync низа к верху). Persist split on/off между сессиями не требуется.

#### Scenario [RS-01]: Toggle split shows empty bottom pane
- **GIVEN** Reader открыт для оригинального документа без привязанного перевода
- **WHEN** пользователь включает режим split
- **THEN** экран делится горизонтально: сверху оригинал, снизу — пустое состояние с действием загрузки файла

#### Scenario [RS-02]: Hide split returns to single pane
- **GIVEN** split включён и в нижней панели уже загружен перевод
- **WHEN** пользователь выключает split
- **THEN** показывается только одна панель с оригиналом; при повторном включении split перевод снова доступен без повторной загрузки в той же сессии

#### Scenario [RS-10]: Show split preserves original scroll
- **GIVEN** Reader в режиме одной панели, оригинал прокручен не к началу
- **WHEN** пользователь включает режим split
- **THEN** верхняя панель продолжает показывать ту же область оригинала, а не сбрасывается к началу; низ не one-shot sync к верху

#### Scenario [RS-11]: Hide split preserves original scroll
- **GIVEN** Reader в режиме split, верх и (при наличии перевода) низ прокручены не к началу
- **WHEN** пользователь выключает режим split, затем включает снова
- **THEN** единственная/верхняя панель оригинала сохраняет область; при повторном Show низ восстанавливает **свою** сохранённую позицию, не sync к верху

### Requirement: Load translation into bottom pane
В пустой нижней панели система SHALL предоставлять действие загрузки файла перевода с устройства. Для **subtitle**-оригинала допустимы те же subtitle-типы, что при import в Library; при успехе система SHALL показать cues перевода и сохранить linked translation (`library/documents`) **без** one-shot sync к верху. Для **EPUB**-оригинала система SHALL принимать **только** `.epub`; при успехе SHALL показать book-режим перевода и сохранить linked EPUB translation **без** one-shot align к верху. Несовместимый формат MUST дать заметную ошибку без привязки. Пока перевод загружен/linked, система SHALL предоставлять удаление **в AppBar рядом со split**, MUST NOT в верхней зоне нижней панели. Когда перевод показан, нижняя панель MUST быть text-only вплотную к divider (без title/pager/delete bar). При ошибке загрузки/парсинга — заметная ошибка без содержимого перевода; верх остаётся доступным.

#### Scenario [RS-04]: Load translation shows cues in bottom
- **GIVEN** split включён на subtitle-оригинале, нижняя панель пуста
- **WHEN** пользователь успешно загружает валидный файл субтитров с одной или более репликами
- **THEN** в нижней панели отображаются cues этого файла (время и текст), без one-shot выравнивания к якорю верха

#### Scenario [RS-13]: Load EPUB translation shows chapters in bottom
- **GIVEN** split включён на EPUB-оригинале, нижняя панель пуста
- **WHEN** пользователь успешно загружает валидный `.epub` перевод
- **THEN** в нижней панели отображается book-режим перевода (абзацы text-only); файл через `parse_epub`; `attachTranslation` сохранил format `epub`; без one-shot align к верху

#### Scenario [RS-07]: Delete translation returns to empty
- **GIVEN** в нижней панели отображается перевод (субтитры или EPUB) или доступно удаление связанного перевода
- **WHEN** пользователь нажимает удаление **рядом со split в AppBar**
- **THEN** нижняя панель возвращается в пустое состояние с действием загрузки, связанный переводный документ больше не привязан к оригиналу, и действие удаления **не** расположено в нижней панели

#### Scenario [RS-08]: Failed load shows error in bottom flow
- **GIVEN** split включён, нижняя панель пуста
- **WHEN** пользователь выбирает файл, который не удаётся загрузить или распарсить, либо формат несовместим с оригиналом (например субтитры к EPUB)
- **THEN** пользователь видит заметную ошибку в контексте Reader / нижней панели, содержимое перевода не показывается, и верхняя панель оригинала остаётся доступной

### Requirement: Synchronized scroll by time then pixel settle
В режиме split для **subtitle**-документов система MUST NOT выполнять выравнивание нижней панели к верхней по `CueDto.startMs`, индексу cue или pixel alignment при Show split, Load translation, open/reopen, scroll settle/fling или programmatic scroll (кроме restore **собственного** якоря низа). После жеста прокрутки одной панели другая MUST NOT двигаться из‑за sync. Для **EPUB** это требование MUST NOT вводить time-based sync. Смена video page MUST NOT вызывать sync (`reader/video-frames`).

#### Scenario [RS-05]: Sync after load by nearest start time
- **GIVEN** верхняя панель subtitle-оригинала прокручена так, что якорная реплика имеет известный `start`, split включён, нижняя панель пуста
- **WHEN** пользователь успешно загружает перевод с несколькими cues с разными `start`
- **THEN** нижняя панель показывает перевод **без** выравнивания к ближайшему `start` якоря верха

#### Scenario [RS-06]: Scroll sync time then pixel on settle
- **GIVEN** обе панели содержат subtitle cues, split включён
- **WHEN** пользователь прокручивает одну панель и жест завершается (settle)
- **THEN** позиция другой панели не меняется из‑за этого жеста (realtime sync отсутствует)

#### Scenario [RS-18]: Show split one-shot syncs translation
- **GIVEN** Reader в одной панели, оригинал прокручен не к началу, в сессии уже есть cues перевода
- **WHEN** пользователь включает split
- **THEN** нижняя панель **не** выравнивается к якорю верха; при наличии сохранённой позиции низа она восстанавливается; последующий скролл любой панели не двигает другую

### Requirement: Resume reading position when reopening original
Если у оригинального документа сохранён якорь чтения (persist — `library/documents`), система SHALL восстановить позицию **оригинала** по формату (subtitle time+alignment; EPUB chapter index LD-28). При наличии linked translation система SHALL также восстановить якорь **нижней** панели из persist перевода **без** one-shot sync низа к верху. Show/Hide split MUST сохранять позиции оригинала и низа независимо. Persist режима split и divider не требуется. Сбой якоря MUST NOT блокировать open.

#### Scenario [RS-12]: Reopen resumes at saved original anchor
- **GIVEN** пользователь ранее читал оригинал не с начала, якорь сохранён, приложение снова открывает этот оригинал в Reader
- **WHEN** Reader показывает оригинал
- **THEN** для subtitle список cues показывает область сохранённого якоря; для EPUB — главу с сохранённым chapter index; нижняя панель при наличии перевода восстанавливается из **своего** якоря, без sync к верху

### Requirement: Synchronize EPUB panes by chapter index
В book-режиме split система MUST NOT выполнять one-shot выравнивание низа к верху по главе/абзацу при Show split или Load EPUB translation. Индекс главы низа меняется только когда пользователь нажимает prev/next на **верхнем** pager (`reader/chapters`): тогда низ = clamp(глава верха), оба списка абзацев — с начала главы. Прокрутка абзацев одной панели MUST NOT менять другую. При **неравном** N глав на load система MUST показать заметную ошибку и MAY clamp; MUST NOT one-shot paragraph align к верху. Realtime зеркалирование абзацев MUST NOT выполняться.

#### Scenario [RS-14]: Chapter pager syncs both panes
- **GIVEN** split включён, обе панели показывают EPUB с одинаковым числом глав N ≥ 2, обе на главе 1
- **WHEN** пользователь переключает верхнюю панель на главу 2 стрелками сверху
- **THEN** нижняя панель также показывает главу 2; оба списка абзацев с начала главы

#### Scenario [RS-15]: Assume matching chapter counts
- **GIVEN** EPUB оригинал и EPUB перевод с одинаковым числом глав, обе панели на одной главе после явного top pager (не one-shot)
- **WHEN** пользователь прокручивает абзацы только в одной панели
- **THEN** другая панель сохраняет свой scroll абзацев; ошибки несовпадения структуры нет

#### Scenario [RS-16]: Mismatched chapter counts on translation load
- **GIVEN** split включён на EPUB-оригинале с N глав, пользователь загружает валидный EPUB-перевод с M глав, M ≠ N
- **WHEN** загрузка/привязка перевода завершается
- **THEN** пользователь видит заметную ошибку о несовпадении структуры; перевод MAY остаться доступным с clamp при последующих top arrows, но MUST NOT молча притворяться, что N = M, и MUST NOT one-shot align абзаца к верху

#### Scenario [RS-19]: Show split one-shot aligns EPUB chapters
- **GIVEN** EPUB оригинал на главе k, список абзацев прокручен к непервому абзацу p, в сессии есть EPUB перевод, split выключен
- **WHEN** пользователь включает split
- **THEN** нижняя панель **не** one-shot выравнивается к главе/абзацу верха; восстанавливается сохранённая позиция низа при наличии

#### Scenario [RS-20]: Load EPUB translation one-shot aligns chapters
- **GIVEN** split включён, EPUB оригинал на главе k с видимым непервым абзацем p, нижняя панель пуста
- **WHEN** пользователь успешно загружает валидный EPUB-перевод
- **THEN** нижняя панель показывает перевод **без** one-shot align к главе/абзацу p верха

#### Scenario [RS-21]: After EPUB one-shot paragraph scroll independent
- **GIVEN** EPUB split с переводом, обе панели видимы
- **WHEN** пользователь прокручивает список абзацев только верхней панели
- **THEN** позиция списка абзацев нижней панели не меняется из‑за этого жеста

#### Scenario [RS-22]: Unequal paragraph counts clamp on one-shot
- **GIVEN** EPUB оригинал на главе k прокручен к абзацу с индексом p, перевод в сессии на главе k имеет M абзацев, M ≤ p, split выключен
- **WHEN** пользователь включает split
- **THEN** приложение не падает; низ не one-shot прыгает к p; при restore собственного якоря низа применяется clamp

## ADDED Requirements

### Requirement: Persist and restore bottom pane position
Система SHALL запоминать позицию нижней панели при settle скролла низа, Hide split и выходе из Reader, и SHALL восстанавливать её при Show/reopen при наличии перевода. Persist — `library/documents` (якорь перевода). Subtitle: cue/`startMs` + alignment; EPUB: индекс абзаца + alignment.

#### Scenario [RS-24]: Persist and restore bottom position
- **GIVEN** пользователь прокрутил низ не к началу и якорь перевода сохранён
- **WHEN** приложение перезапускается и документ снова открыт с переводом
- **THEN** нижняя панель показывает сохранённую область (clamp при несовпадении длины)

#### Scenario [RS-23]: Bottom pane text-only under divider
- **GIVEN** split с загруженным переводом (subtitle или EPUB)
- **WHEN** пользователь смотрит нижнюю панель
- **THEN** под divider сразу текст перевода; title/pager/delete chrome в низу отсутствуют
