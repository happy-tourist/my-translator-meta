# reader/split (delta) — EPUB chapter sync

Связано с `reader/chapters`, `library/documents`, `parse/epub`. Subtitle time-sync остаётся; для book-режима добавляется sync по индексу главы и ограничение формата перевода.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RS-04 | Load translation shows cues in bottom | subtitle path unchanged |
| RS-13 | Load EPUB translation shows chapters in bottom | Flutter: epub original + epub translation → book panes |
| RS-07 | Delete translation returns to empty | unchanged intent (both kinds) |
| RS-08 | Failed load shows error in bottom flow | also incompatible format for epub |
| RS-05 | Sync after load by nearest start time | subtitle-only unchanged |
| RS-06 | Scroll sync time then pixel on settle | subtitle-only unchanged |
| RS-14 | Chapter pager syncs both panes | Flutter: change chapter on top → bottom same index |
| RS-15 | Assume matching chapter counts | Flutter/unit: equal N; document product assumption |
| RS-09 | Reopen restores translation in split | epub translation restores in book mode |
| RS-12 | Reopen resumes at saved original anchor | Flutter: subtitle → cue time; epub → chapter index (RC-06 / LD-28) |
| RS-16 | Mismatched chapter counts on translation load | Flutter: unequal N → clamp + visible error; equal N still RS-14/15 |

## MODIFIED Requirements

### Requirement: Load translation into bottom pane
В пустой нижней панели система SHALL предоставлять действие загрузки файла перевода с устройства. Для **subtitle**-оригинала допустимы те же subtitle-типы, что при import в Library; при успехе система SHALL показать cues перевода (время start→end и текст) и сохранить linked translation (`library/documents`). Для **EPUB**-оригинала система SHALL принимать **только** `.epub`; при успехе SHALL показать book-режим перевода (главы/абзацы, как верх) и сохранить linked EPUB translation. Несовместимый формат MUST дать заметную ошибку без привязки перевода. Замена файла без удаления не требуется: пока перевод загружен, система SHALL предоставлять удаление в **верхней зоне нижней панели**, не в AppBar. При ошибке загрузки/парсинга система MUST показать заметную ошибку без показа содержимого перевода; верхняя панель оригинала остаётся доступной.

#### Scenario [RS-04]: Load translation shows cues in bottom
- **GIVEN** split включён на subtitle-оригинале, нижняя панель пуста
- **WHEN** пользователь успешно загружает валидный файл субтитров с одной или более репликами
- **THEN** в нижней панели отображаются cues этого файла (время и текст), аналогично верхней панели

#### Scenario [RS-13]: Load EPUB translation shows chapters in bottom
- **GIVEN** split включён на EPUB-оригинале, нижняя панель пуста
- **WHEN** пользователь успешно загружает валидный `.epub` перевод
- **THEN** в нижней панели отображается book-режим перевода (главы/абзацы), аналогично верхней панели; файл распарсен через `parse_epub` (не `parse_subtitles`), а `attachTranslation` сохранил format `epub`

#### Scenario [RS-07]: Delete translation returns to empty
- **GIVEN** в нижней панели отображается перевод (субтитры или EPUB) или доступно удаление связанного перевода
- **WHEN** пользователь нажимает удаление в верхней зоне нижней панели
- **THEN** нижняя панель возвращается в пустое состояние с действием загрузки, связанный переводный документ больше не привязан к оригиналу, и действие удаления не расположено в AppBar

#### Scenario [RS-08]: Failed load shows error in bottom flow
- **GIVEN** split включён, нижняя панель пуста
- **WHEN** пользователь выбирает файл, который не удаётся загрузить или распарсить, либо формат несовместим с оригиналом (например субтитры к EPUB)
- **THEN** пользователь видит заметную ошибку в контексте Reader / нижней панели, содержимое перевода не показывается, и верхняя панель оригинала остаётся доступной

### Requirement: Synchronized scroll by time then pixel settle
В режиме split для **subtitle**-документов при наличии cues в обеих панелях система SHALL синхронизировать прокрутку так: (1) определить якорную реплику панели, получившей жест пользователя (или верхнюю панель при первом выравнивании после загрузки / включения split), (2) найти в другой панели реплику с ближайшим временем начала (`CueDto.startMs`, сравнение как `BigInt`), (3) прокрутить другую панель к этой реплике, (4) выполнить пиксельную подстройку только после settle (окончание жеста / programmatic jump), без непрерывного зеркалирования offset во время fling. Сразу после успешной первой загрузки перевода система MUST выполнить такое выравнивание относительно текущей позиции верхней панели. Для **EPUB** book-режима это требование MUST NOT применяться к chapter pager (см. chapter-index sync ниже); прокрутка абзацев внутри главы MAY оставаться независимой.

#### Scenario [RS-05]: Sync after load by nearest start time
- **GIVEN** верхняя панель subtitle-оригинала прокручена так, что якорная реплика имеет известный `start`, и нижняя панель пуста
- **WHEN** пользователь успешно загружает перевод с несколькими cues с разными `start`
- **THEN** нижняя панель выравнивается к реплике с ближайшим `start` к якорю верха

#### Scenario [RS-06]: Scroll sync time then pixel on settle
- **GIVEN** обе панели содержат subtitle cues и split включён
- **WHEN** пользователь прокручивает одну панель и жест завершается (settle)
- **THEN** другая панель оказывается на реплике с ближайшим `start` к якорю прокрученной панели, с пиксельной подстройкой после settle, без требования 1:1 копирования offset во время fling

### Requirement: Restore translation when reopening original
Если у оригинального документа есть сохранённый связанный перевод, система SHALL при открытии Reader сделать перевод доступным в нижней панели после включения split (без обязательной повторной загрузки с устройства в той же установке приложения). Для EPUB оригинал/перевод MUST открываться в book-режиме. Поведение хранения связи — capability `library/documents`.

#### Scenario [RS-09]: Reopen restores translation in split
- **GIVEN** оригинал ранее получил связанный перевод через нижнюю панель, приложение снова открывает этот оригинал в Reader
- **WHEN** пользователь включает split
- **THEN** нижняя панель показывает сохранённый перевод (cues для субтитров или главы/абзацы для EPUB), не пустое состояние загрузки

### Requirement: Resume reading position when reopening original
Если у оригинального документа сохранён якорь чтения (persist — `library/documents`), система SHALL восстановить позицию **по формату**:
- **subtitle:** прокрутить панель оригинала к реплике с ближайшим `CueDto.startMs` к сохранённому якорю (`BigInt`) и применить alignment при наличии; при cues перевода в сессии SHOULD выровнять низ по времени (как после load).
- **EPUB / book-режим:** показать главу с 0-based индексом из якоря (`lastReadStartMs` format-gated; `reader/chapters` RC-06 / LD-28); MUST NOT вызывать `nearestCueIndex` / time-nearest по абзацам.
Если якоря нет или контент пуст — открытие с начала (первая глава / начало списка). Persist режима split и divider не требуется. Сбой якоря MUST NOT блокировать open.

#### Scenario [RS-12]: Reopen resumes at saved original anchor
- **GIVEN** пользователь ранее читал оригинал не с начала, якорь сохранён, приложение снова открывает этот оригинал в Reader
- **WHEN** Reader показывает оригинал
- **THEN** для subtitle список cues показывает область сохранённого якоря; для EPUB — главу с сохранённым chapter index (не обязательно первую), без интерпретации якоря как cue `startMs`

## ADDED Requirements

### Requirement: Synchronize EPUB panes by chapter index
В book-режиме split при наличии EPUB-содержимого в обеих панелях система SHALL синхронизировать **индекс текущей главы**: смена главы в одной панели (pager prev/next или эквивалент) MUST установить ту же 0-based (или согласованную 1-based UI) главу в другой панели, когда индекс существует в обеих. Продуктовая предпосылка: готовые переводы имеют **одинаковое** число глав. При **неравном** N на load перевода система MUST показать заметную ошибку и MAY clamp индекс в допустимый диапазон каждой панели (не ломая RS-14/15 при равном N). Скролл списка абзацев внутри главы не обязан зеркалироваться пиксель-в-пиксель.

#### Scenario [RS-14]: Chapter pager syncs both panes
- **GIVEN** split включён, обе панели показывают EPUB с одинаковым числом глав N ≥ 2, текущая глава 1
- **WHEN** пользователь переключает верхнюю панель на главу 2
- **THEN** нижняя панель тоже показывает главу 2

#### Scenario [RS-15]: Assume matching chapter counts
- **GIVEN** EPUB оригинал и EPUB перевод с одинаковым числом глав
- **WHEN** пользователь листает главы в пределах 1…N
- **THEN** обе панели остаются на одном и том же индексе главы без ошибки несовпадения структуры

#### Scenario [RS-16]: Mismatched chapter counts on translation load
- **GIVEN** split включён на EPUB-оригинале с N глав, пользователь загружает валидный EPUB-перевод с M глав, M ≠ N
- **WHEN** загрузка/привязка перевода завершается
- **THEN** пользователь видит заметную ошибку о несовпадении структуры; перевод MAY остаться доступным с clamp индекса, но MUST NOT молча притворяться, что N = M
