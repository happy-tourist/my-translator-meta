# reader/video-frames Specification

## Purpose

Позволяет на экране Reader привязать видео к открытому документу и **заменить** список cues оригинала страницами из **трёх вертикальных** скриншотов (по одной целой секунде), с опциональным текстом cue под кадром по правилу «округлянный конец реплики», pager у **низа** панели, прокруткой к первому кадру при смене страницы и метками времени субтитров **из файла** (когда текст есть).

**Слой (намеренно без Rust):** decode кадров и probe — только Dart (native FFmpeg-kit / web canvas); persist — Drift (`library/documents`).

Связанные capabilities: cues — `reader/cues`; split — `reader/split`; persist — `library/documents`.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| VF-01 | Load video replaces original with paged column of 3 | Flutter widget: 3 stacked frames + bottom pager |
| VF-02 | Model has one frame slot per whole-video second | unit: duration N → `0..N-1`; UI 3 per page |
| VF-03 | Empty cue text still shows frame without timestamp | Flutter widget: no ending cue → frame + empty text, no time label |
| VF-04 | Cue text under frame at rounded end second with file times | unit/widget: end-second attach; under text show start→end from cue |
| VF-05 | Load jumps to page for current reading time | Flutter widget: page contains floor(T/1000) |
| VF-06 | Original page change syncs translation when split | Flutter widget: next/prev → bottom near same time |
| VF-07 | Bottom scroll does not move original when video | Flutter widget: translation scroll → page unchanged |
| VF-08 | Remove video restores cue list at same time | Flutter widget: remove → cues near T |
| VF-09 | Frames only in original pane | Flutter widget: no frames in bottom |
| VF-10 | Failed frame slot shows placeholder | Flutter widget: extract fail → placeholder |
| VF-11 | Unsupported or failed pick shows error | Flutter widget: bad pick → error; no paged UI for that choice |
| VF-12 | Tap and highlight still work on cue text under frames | Flutter widget: token tap under frame |
| VF-13 | Open/restore jumps to reading time or last page/second | Flutter widget: restore not forced to page 0 |
| VF-14 | Pager chrome is at bottom of original pane | Flutter widget: prev/next below frame list |
| VF-15 | Page change scrolls to first frame | Flutter widget: after next/prev, first slot of page visible |

## ADDED Requirements

### Requirement: Load and remove video from Reader chrome
Система SHALL позволять на экране Reader выбрать видеофайл (MP4/MKV) через действие в верхней зоне chrome рядом со split, и SHALL позволять убрать привязанное видео отдельным действием там же. После успешной привязки система MUST **заменить** список cues оригинала paged vertical triple (см. ниже). После удаления система MUST вернуть обычный cue list на временном якоре текущей секунды / страницы и MUST NOT оставлять привязку в сессии. Повторный выбор SHALL заменить привязку и сбросить кэш кадров предыдущего файла. Аудиоплеер не требуется.

#### Scenario [VF-01]: Load video replaces original with paged column of 3
- **GIVEN** Reader открыт без привязанного видео
- **WHEN** пользователь успешно выбирает поддерживаемый видеофайл
- **THEN** панель оригинала показывает три кадра друг под другом и pager вместо cue list (не горизонтальную полосу над cues и не infinite list)

#### Scenario [VF-08]: Remove video restores cue list at same time
- **GIVEN** Reader с video pages около секунды T
- **WHEN** пользователь удаляет видео
- **THEN** обычный cue list прокручен к репликам около T

#### Scenario [VF-11]: Unsupported or failed pick shows error
- **GIVEN** Reader открыт
- **WHEN** выбран неподдерживаемый файл или probe fails
- **THEN** заметная ошибка; paged frames для этого выбора не показываются

### Requirement: Paged vertical frames by whole second
Система SHALL моделировать кадры по каждой целой секунде от `0` до `floor(durationSeconds) - 1`. UI MUST показывать одновременно не более трёх кадров **вертикально** и MUST предоставлять pager размера 3. Элементы переключения страниц (стрелки / счётчик) MUST располагаться у **низа** панели оригинала (под списком кадров), а не над ним. Извлечение кадров MUST быть ленивым для текущей страницы; кэш MAY использоваться. При `durationSeconds < 1` → VF-11.

#### Scenario [VF-02]: Model has one frame slot per whole-video second
- **GIVEN** видео ≥ 3 секунд
- **WHEN** первая страница отображается
- **THEN** видны кадры 0, 1, 2 в столбце; остальные через pager

#### Scenario [VF-14]: Pager chrome is at bottom of original pane
- **GIVEN** video pages отображаются
- **WHEN** панель оригинала видна
- **THEN** элементы prev/next (и счётчик страниц, если есть) находятся ниже области кадров

#### Scenario [VF-03]: Empty cue text still shows frame without timestamp
- **GIVEN** ни одна cue не имеет `round(endMs/1000) == t`
- **WHEN** слот `t` виден
- **THEN** кадр показан, текстовая область пуста, метки времени нет

#### Scenario [VF-10]: Failed frame slot shows placeholder
- **GIVEN** extract для одной секунды на странице fails
- **WHEN** страница видна
- **THEN** placeholder в ошибочном слоте; соседи и жесты на тексте доступны

### Requirement: Scroll to first frame on page change
При переключении на другую страницу (prev/next или programmatic jump на другую page) система MUST прокрутить область кадров так, чтобы **первый** кадр новой страницы был видим (эквивалент scroll-to-top списка кадров страницы).

#### Scenario [VF-15]: Page change scrolls to first frame
- **GIVEN** пользователь прокрутил список кадров текущей страницы вниз
- **WHEN** пользователь нажимает next (или prev) и страница меняется
- **THEN** виден первый кадр новой страницы (не середина/низ предыдущего скролла)

### Requirement: Cue text under frame by rounded end second
Для секунды `t` система SHALL показывать под кадром текст всех реплик с `round(endMs/1000) = t`, в порядке списка. Display-текст cue MUST быть join `text_lines` через `\n`; несколько cue на одной секунде MUST все быть видны. Текст MUST NOT появляться под start/middle секундами той же реплики. Пустой слот → кадр без текста.

Когда под кадром показан текст cue, система MUST показать временну́ю метку этой реплики **из субтитрового файла** в том же виде, что обычный Reader cue row: `startMs → endMs` (форматированные). Система MUST NOT подставлять метку «секунда кадра» вместо времён cue. Когда текста нет, система MUST NOT показывать метку времени в слоте.

#### Scenario [VF-04]: Cue text under frame at rounded end second with file times
- **GIVEN** cue с `round(endMs/1000) = t` (и опционально вторая с тем же концом)
- **WHEN** слот `t` отображается
- **THEN** под кадром виден текст реплик и их метки `start → end` из файла; под другими секундами первой cue её текста нет

### Requirement: Jump to page for current reading time on load and restore
При Load / restore система SHALL открыть страницу, содержащую `T = clamp(floor(timeMs/1000), 0, durationSeconds-1)` (или сохранённый якорь). MUST NOT принудительно оставлять на странице 0, если якорь позже.

#### Scenario [VF-05]: Load jumps to page for current reading time
- **GIVEN** cue list около 10 минут
- **WHEN** успешный Load video
- **THEN** страница около секунды ~600

#### Scenario [VF-13]: Open/restore jumps to reading time or last page/second
- **GIVEN** сохранённая привязка и якорь
- **WHEN** restore paged frames
- **THEN** страница около якоря, не принудительно первая

### Requirement: Split sync with video pages
Кадры только в original; bottom без кадров. Смена страницы → sync translation via `nearestCueIndex`; scroll bottom ↛ original page. Show split MUST NOT сбрасывать page.

#### Scenario [VF-06]: Original page change syncs translation when split
- **GIVEN** video + split + translation
- **WHEN** next/prev page
- **THEN** bottom около того же времени

#### Scenario [VF-07]: Bottom scroll does not move original when video
- **GIVEN** video на T, split on
- **WHEN** scroll translation
- **THEN** original page/second остаётся на T

#### Scenario [VF-09]: Frames only in original pane
- **GIVEN** video + split
- **WHEN** обе панели видны
- **THEN** кадры только сверху

### Requirement: Frame extract errors and learning gestures unchanged
Tap/highlight на тексте cue под кадрами MUST соответствовать `reader/tap` и `reader/highlight`.

#### Scenario [VF-12]: Tap and highlight still work on cue text under frames
- **GIVEN** текст cue под кадром токенизируем
- **WHEN** tap на слово
- **THEN** поведение как у обычного Reader без регрессии
