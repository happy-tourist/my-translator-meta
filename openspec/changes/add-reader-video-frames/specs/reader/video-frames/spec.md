# reader/video-frames Specification

## Purpose

Позволяет на экране Reader привязать видео к открытому документу и **заменить** список cues оригинала вертикальным потоком скриншотов по **каждой целой секунде** (кадр под кадром), с опциональным текстом cue под кадром и переходом к текущей позиции чтения.

**Слой (намеренно без Rust):** decode кадров и probe длительности — только Dart (native FFmpeg-kit / web canvas) за одним фасадом; нового Rust/FRB video API нет. Persist привязки и JPEG-кэша — Drift (`library/documents`).

Связанные capabilities: cues — `reader/cues`; split — `reader/split`; persist — `library/documents`.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| VF-01 | Load video replaces original with second timeline | Flutter widget: pick ok → interleaved list visible (not 3-across strip) |
| VF-02 | Timeline has one row per whole-video second | unit/widget: duration N sec → N rows for seconds `0..N-1` |
| VF-03 | Empty cue text still shows frame row | Flutter widget: second without covering cue → frame + empty text |
| VF-04 | Cue text under frame when interval covers second | unit: second inside cue → text; outside → empty |
| VF-05 | Load jumps to current reading time | Flutter widget: scrolled cues ~T → after load list shows around second floor(T) |
| VF-06 | Original scroll syncs translation when split | Flutter widget: scroll interleaved original → bottom near same time |
| VF-07 | Bottom scroll does not move original when video | Flutter widget: scroll translation → original second unchanged |
| VF-08 | Remove video restores cue list at same time | Flutter widget: remove → cues back; position near same second |
| VF-09 | Frames only in original pane | Flutter widget: split on → no frame widgets in bottom pane |
| VF-10 | Failed frame row shows placeholder | Flutter widget: extract fail → placeholder; neighbors ok |
| VF-11 | Unsupported or failed pick shows error | Flutter widget: bad pick/probe → error; timeline not shown |
| VF-12 | Tap and highlight still work on cue text under frames | Flutter widget: token tap / highlight on text under frame |
| VF-13 | Open/restore jumps to reading time or last second | Flutter widget: restore attachment → jump without forcing second 0 |

## ADDED Requirements

### Requirement: Load and remove video from Reader chrome
Система SHALL позволять на экране Reader выбрать видеофайл с устройства (поддерживаемые контейнеры включают MP4 и MKV) через действие в верхней зоне chrome рядом с управлением split, и SHALL позволять убрать привязанное видео отдельным действием в той же верхней зоне. После успешной привязки система MUST **заменить** список cues оригинала вертикальным second-timeline (см. ниже). После удаления система MUST вернуть обычный список cues на временном якоре текущей секунды timeline и MUST NOT оставлять привязку в сессии. Повторный выбор при уже привязанном видео SHALL заменить привязку и сбросить кэш кадров предыдущего файла для этого документа. Воспроизведение аудио и полноценный плеер не требуются.

#### Scenario [VF-01]: Load video replaces original with second timeline
- **GIVEN** Reader открыт для документа без привязанного видео
- **WHEN** пользователь успешно выбирает поддерживаемый видеофайл
- **THEN** панель оригинала показывает вертикальный поток кадров по секундам вместо обычного cue list (не полосу из трёх кадров над cues)

#### Scenario [VF-08]: Remove video restores cue list at same time
- **GIVEN** Reader открыт с video timeline на секунде T
- **WHEN** пользователь удаляет видео
- **THEN** отображается обычный cue list, прокрученный к репликам около времени T; tap/highlight работают как до загрузки видео

#### Scenario [VF-11]: Unsupported or failed pick shows error
- **GIVEN** Reader открыт без видео (или с видео)
- **WHEN** пользователь выбирает файл, который система не может использовать (неподдерживаемый тип или сбой probe)
- **THEN** пользователь видит заметную ошибку; timeline для этого выбора не показывается (прежняя привязка не ломается без явной замены)

### Requirement: Vertical second timeline in original pane
Система SHALL строить элементы timeline по **каждой целой секунде** длительности видео от `0` до `floor(durationSeconds) - 1`. Каждый элемент MUST показывать: JPEG-кадр (или плейсхолдер загрузки/ошибки), временную метку секунды и текстовую область cue. Элементы MUST идти **друг под другом** (один столбец), без сетки «три в ряд» и без отдельного pager страниц по 3. Извлечение кадров MUST быть ленивым для видимой области viewport (не обязательно для всех секунд сразу); уже извлечённые секунды MAY браться из кэша (`library/documents`). При `durationSeconds < 1` после probe → VF-11 (timeline не показывать).

#### Scenario [VF-02]: Timeline has one row per whole-video second
- **GIVEN** привязано видео длительностью не менее трёх секунд
- **WHEN** timeline отображается
- **THEN** в потоке есть отдельные строки для секунд 0, 1, 2, … (вертикально), а не три слота одной «страницы»

#### Scenario [VF-03]: Empty cue text still shows frame row
- **GIVEN** ни одна cue оригинала не покрывает секунду `t`
- **WHEN** строка секунды `t` видна
- **THEN** кадр (или плейсхолдер) показан, текстовая область пуста, строка остаётся в списке без ошибки

#### Scenario [VF-10]: Failed frame row shows placeholder
- **GIVEN** привязано видео, extract для одной секунды завершается ошибкой
- **WHEN** эта строка отображается
- **THEN** в ошибочной строке виден плейсхолдер кадра, соседние строки и жесты на тексте остаются доступны

### Requirement: Optional cue text under each frame
Для каждой секунды `t` система SHALL показывать под кадром текст cue **только если** существует реплика оригинала, чей полуинтервал `[startMs, endMs)` покрывает момент `t·1000` мс (при нескольких — первая cue в порядке списка оригинала). Display-текст MUST быть join `text_lines` через `\n` (как cues-first в Reader). Если покрывающей реплики нет, система MUST оставить текстовую область пустой (допустимо, не ошибка) и MUST всё равно показать строку кадра.

#### Scenario [VF-04]: Cue text under frame when interval covers second
- **GIVEN** оригинал имеет cue с `startMs`/`endMs`, покрывающими секунду `t`
- **WHEN** строка секунды `t` отображается
- **THEN** под кадром виден display-текст этой реплики (`text_lines` через `\n`)

### Requirement: Jump to current reading time on load and restore
При успешной загрузке видео (и при restore привязки при open, когда timeline показывается) система SHALL определить текущее время чтения **до** transform (время видимого верха списка cues / reading anchor / эквивалент) и MUST прокрутить second-timeline к секунде `floor(timeMs/1000)` (clamp в `[0, durationSeconds-1]`). Система MUST NOT оставлять пользователя на секунде 0, если до Load позиция чтения была существенно позже.

#### Scenario [VF-05]: Load jumps to current reading time
- **GIVEN** Reader на обычном cue list прокручен к области около 10 минут
- **WHEN** пользователь успешно загружает видео
- **THEN** interleaved timeline открывается около секунды ~600, а не с начала файла

#### Scenario [VF-13]: Open/restore jumps to reading time or last second
- **GIVEN** оригинал имеет сохранённую привязку видео; при open доступен reading anchor или сохранённый `last_second`
- **WHEN** Reader восстанавливает timeline
- **THEN** список открывается около соответствующего времени (не принудительно с секунды 0, если якорь/last_second известен)

### Requirement: Split sync with video timeline
Кадры MUST отображаться **только** в панели оригинала; нижняя панель split MUST NOT содержать кадры video timeline. При включённом split и активном video timeline: скролл interleaved-оригинала SHALL прокручивать нижнюю панель перевода к ближайшей cue того же времени (reuse `nearestCueIndex`); скролл нижней панели MUST NOT изменять позицию second-timeline оригинала. При **включении** split система SHALL выровнять низ к текущему верху оригинала / текущей секунде и MUST NOT сбрасывать секунду timeline из‑за remount.

#### Scenario [VF-06]: Original scroll syncs translation when split
- **GIVEN** video timeline и split включены, перевод загружен
- **WHEN** пользователь прокручивает interleaved-оригинал к другой секунде
- **THEN** нижняя панель показывает область около того же времени

#### Scenario [VF-07]: Bottom scroll does not move original when video
- **GIVEN** video timeline на секунде T, split включён
- **WHEN** пользователь прокручивает нижнюю панель перевода
- **THEN** original timeline остаётся на секунде T

#### Scenario [VF-09]: Frames only in original pane
- **GIVEN** video timeline и split включены
- **WHEN** обе панели видимы
- **THEN** кадры присутствуют только в верхней (original) панели

### Requirement: Frame extract errors and learning gestures unchanged
Пока video timeline виден, система MUST сохранять существующее поведение tap → vocab / highlight на тексте cue под кадрами (`reader/tap`, `reader/highlight`) без требования жестов на самих кадрах.

#### Scenario [VF-12]: Tap and highlight still work on cue text under frames
- **GIVEN** Reader с video timeline и токенизируемым текстом cue под кадром
- **WHEN** пользователь тапает слово в тексте под кадром
- **THEN** поведение выделения / add-vocab / highlight соответствует `reader/tap` и `reader/highlight` без регрессии из‑за timeline
