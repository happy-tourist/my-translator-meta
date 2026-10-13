# reader/video-frames Specification

## Purpose

Позволяет на экране Reader привязать видео к открытому документу и **заменить** список cues оригинала страницами из **трёх вертикальных** скриншотов (по одной целой секунде), с опциональным текстом cue под кадром по правилу «округлянный конец реплики», без метки времени и без горизонтальной полосы над cues.

**Слой (намеренно без Rust):** decode кадров и probe длительности — только Dart (native FFmpeg-kit / web canvas) за одним фасадом; нового Rust/FRB video API нет. Persist привязки и JPEG-кэша — Drift (`library/documents`).

Связанные capabilities: cues — `reader/cues`; split — `reader/split`; persist — `library/documents`.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| VF-01 | Load video replaces original with paged column of 3 | Flutter widget: pick ok → 3 stacked frames + pager (not cue list, not infinite all-seconds list, not horizontal strip) |
| VF-02 | Model has one frame slot per whole-video second | unit: duration N → seconds `0..N-1`; UI shows 3 per page |
| VF-03 | Empty cue text still shows frame | Flutter widget: second with no ending cue → frame + empty text, **no** timestamp |
| VF-04 | Cue text under frame at rounded end second | unit: `round(endMs/1000)==t` → text under t; start/middle → empty; multi-cue same end → both |
| VF-05 | Load jumps to page for current reading time | Flutter widget: cues ~T → after load page contains floor(T/1000) second |
| VF-06 | Original page change syncs translation when split | Flutter widget: next/prev page → bottom near same time |
| VF-07 | Bottom scroll does not move original when video | Flutter widget: scroll translation → page/second unchanged |
| VF-08 | Remove video restores cue list at same time | Flutter widget: remove → cues back near same second |
| VF-09 | Frames only in original pane | Flutter widget: split on → no frames in bottom |
| VF-10 | Failed frame slot shows placeholder | Flutter widget: extract fail → placeholder; neighbors ok |
| VF-11 | Unsupported or failed pick shows error | Flutter widget: bad pick/probe → error; paged UI not shown for that choice |
| VF-12 | Tap and highlight still work on cue text under frames | Flutter widget: token tap / highlight on text under frame |
| VF-13 | Open/restore jumps to reading time or last page/second | Flutter widget: restore → not forced to page 0 / second 0 when anchor known |

## ADDED Requirements

### Requirement: Load and remove video from Reader chrome
Система SHALL позволять на экране Reader выбрать видеофайл с устройства (поддерживаемые контейнеры включают MP4 и MKV) через действие в верхней зоне chrome рядом с управлением split, и SHALL позволять убрать привязанное видео отдельным действием в той же верхней зоне. После успешной привязки система MUST **заменить** список cues оригинала paged vertical triple (см. ниже). После удаления система MUST вернуть обычный список cues на временном якоре текущей секунды / страницы и MUST NOT оставлять привязку в сессии. Повторный выбор при уже привязанном видео SHALL заменить привязку и сбросить кэш кадров предыдущего файла для этого документа. Воспроизведение аудио и полноценный плеер не требуются.

#### Scenario [VF-01]: Load video replaces original with paged column of 3
- **GIVEN** Reader открыт для документа без привязанного видео
- **WHEN** пользователь успешно выбирает поддерживаемый видеофайл
- **THEN** панель оригинала показывает три кадра **друг под другом** и элементы переключения страниц вместо обычного cue list (не горизонтальную полосу над cues и не бесконечный список всех секунд)

#### Scenario [VF-08]: Remove video restores cue list at same time
- **GIVEN** Reader открыт с video pages на секунде / странице около T
- **WHEN** пользователь удаляет видео
- **THEN** отображается обычный cue list, прокрученный к репликам около времени T; tap/highlight работают как до загрузки видео

#### Scenario [VF-11]: Unsupported or failed pick shows error
- **GIVEN** Reader открыт без видео (или с видео)
- **WHEN** пользователь выбирает файл, который система не может использовать (неподдерживаемый тип или сбой probe)
- **THEN** пользователь видит заметную ошибку; paged frames для этого выбора не показываются (прежняя привязка не ломается без явной замены)

### Requirement: Paged vertical frames by whole second
Система SHALL моделировать кадры по **каждой целой секунде** длительности видео от `0` до `floor(durationSeconds) - 1`. UI MUST показывать одновременно **не более трёх** таких кадров, расположенных **вертикально** (столбец), и MUST предоставлять переключение на остальные секунды стрелками / pager страниц размера 3. Каждый видимый слот MUST показывать JPEG-кадр (или плейсхолдер загрузки/ошибки) и текстовую область cue; система MUST NOT показывать под кадром метку времени (timestamp). Извлечение кадров MUST быть ленивым для текущей страницы; уже извлечённые секунды MAY браться из кэша (`library/documents`). При `durationSeconds < 1` после probe → VF-11 (paged UI не показывать).

#### Scenario [VF-02]: Model has one frame slot per whole-video second
- **GIVEN** привязано видео длительностью не менее трёх секунд
- **WHEN** первая страница отображается
- **THEN** видны кадры секунд 0, 1 и 2 в столбце; остальные секунды доступны через переключение страниц

#### Scenario [VF-03]: Empty cue text still shows frame
- **GIVEN** ни одна cue оригинала не имеет `round(endMs/1000) == t`
- **WHEN** слот секунды `t` виден
- **THEN** кадр (или плейсхолдер) показан, текстовая область пуста, **метки времени нет**, слот остаётся без ошибки

#### Scenario [VF-10]: Failed frame slot shows placeholder
- **GIVEN** привязано видео, extract для одной секунды на странице завершается ошибкой
- **WHEN** эта страница отображается
- **THEN** в ошибочном слоте виден плейсхолдер кадра, соседние слоты и жесты на тексте остаются доступны

### Requirement: Cue text under frame by rounded end second
Для каждой секунды `t` система SHALL показывать под кадром текст **всех** реплик оригинала, для которых `round(endMs / 1000) = t` (обычное арифметическое округление), в порядке списка cues. Display-текст каждой cue MUST быть join `text_lines` через `\n`; при нескольких cue на одной секунде система MUST показать **все** их тексты под этим кадром. Система MUST NOT размещать текст cue под кадрами секунд начала или середины интервала той же реплики. Если подходящих cue нет, текстовая область MUST быть пустой (кадр остаётся). Система MUST NOT показывать timestamp под кадром.

#### Scenario [VF-04]: Cue text under frame at rounded end second
- **GIVEN** оригинал имеет cue с `endMs`, для которого `round(endMs/1000) = t`, и опционально вторую cue с тем же округлянным концом
- **WHEN** слот секунды `t` отображается
- **THEN** под кадром виден display-текст этой (этих) реплик; под кадрами других секунд той же первой cue текста этой cue нет

### Requirement: Jump to page for current reading time on load and restore
При успешной загрузке видео (и при restore привязки при open) система SHALL определить текущее время чтения **до** transform и MUST открыть страницу, содержащую секунду `T = clamp(floor(timeMs/1000), 0, durationSeconds-1)` (или сохранённый якорь last second / page). Система MUST NOT оставлять пользователя на первой странице / секунде 0, если до Load позиция чтения была существенно позже.

#### Scenario [VF-05]: Load jumps to page for current reading time
- **GIVEN** Reader на обычном cue list прокручен к области около 10 минут
- **WHEN** пользователь успешно загружает видео
- **THEN** paged UI открывается на странице около секунды ~600, а не с начала файла

#### Scenario [VF-13]: Open/restore jumps to reading time or last page/second
- **GIVEN** оригинал имеет сохранённую привязку видео; при open доступен reading anchor или сохранённый last second / page
- **WHEN** Reader восстанавливает paged frames
- **THEN** открывается страница около соответствующего времени (не принудительно с секунды 0 / первой страницы, если якорь известен)

### Requirement: Split sync with video pages
Кадры MUST отображаться **только** в панели оригинала; нижняя панель split MUST NOT содержать кадры. При включённом split и активном video UI: смена страницы оригинала SHALL прокручивать нижнюю панель перевода к ближайшей cue времени текущей секунды страницы (reuse `nearestCueIndex`); скролл нижней панели MUST NOT изменять страницу / секунду оригинала. При **включении** split система SHALL выровнять низ к текущей секунде и MUST NOT сбрасывать страницу из‑за remount.

#### Scenario [VF-06]: Original page change syncs translation when split
- **GIVEN** video pages и split включены, перевод загружен
- **WHEN** пользователь переключает страницу оригинала
- **THEN** нижняя панель показывает область около того же времени

#### Scenario [VF-07]: Bottom scroll does not move original when video
- **GIVEN** video page на секунде T, split включён
- **WHEN** пользователь прокручивает нижнюю панель перевода
- **THEN** original page / second остаётся на T

#### Scenario [VF-09]: Frames only in original pane
- **GIVEN** video pages и split включены
- **WHEN** обе панели видимы
- **THEN** кадры присутствуют только в верхней (original) панели

### Requirement: Frame extract errors and learning gestures unchanged
Пока paged frames видны, система MUST сохранять существующее поведение tap → vocab / highlight на тексте cue под кадрами (`reader/tap`, `reader/highlight`) без требования жестов на самих кадрах.

#### Scenario [VF-12]: Tap and highlight still work on cue text under frames
- **GIVEN** Reader с video pages и токенизируемым текстом cue под кадром
- **WHEN** пользователь тапает слово в тексте под кадром
- **THEN** поведение выделения / add-vocab / highlight соответствует `reader/tap` и `reader/highlight` без регрессии из‑за video UI
