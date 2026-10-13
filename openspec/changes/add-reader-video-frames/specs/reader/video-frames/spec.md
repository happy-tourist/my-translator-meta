# reader/video-frames Specification

## Purpose

Позволяет на экране Reader привязать видео к открытому документу, показывать скриншоты по секундам над списком cues (по 3 на страницу) и листать страницы, не меняя поведение tap / highlight / списка cues.

**Слой (намеренно без Rust):** decode кадров и probe длительности — только Dart (native FFmpeg-kit / web canvas interop) за одним фасадом; **нового** Rust/FRB video API нет; `rust/` и FRB codegen для video не трогаем. Это FE-only compute exception (аналог platform-исключения web translate), не gap NLP/parse. Persist привязки и JPEG-кэша — Drift / platform storage (`library/documents`), не Rust SQLite. Существующие FRB `parse_subtitles` / `tokenize_cues` / mark / translate — без изменений из‑за полосы.

Связанные capabilities: список cues — `reader/cues`; split — `reader/split`; persist привязки — `library/documents`.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| VF-01 | Load video shows frame strip | Flutter widget: pick ok → strip visible with pager |
| VF-02 | Page shows three second slots | Flutter widget/unit: 1-based page N → seconds `3*(N-1)..3*(N-1)+2` (or fewer on last) |
| VF-03 | Pager advances and shows counter | Flutter widget: next/prev + page/total |
| VF-04 | Cue text under frame when interval covers second | unit: second inside cue → text; outside → empty |
| VF-05 | Empty text under frame is allowed | Flutter widget: slot without covering cue shows frame without text |
| VF-06 | Frame page change scrolls original cues | Flutter widget: change page → original list jumps near page time |
| VF-07 | Show split scrolls translation to top | Flutter widget: with video, enable split → bottom near top/frame time |
| VF-08 | Cue scroll does not move frame strip | Flutter widget: scroll cues → strip page unchanged |
| VF-09 | Remove video clears strip | Flutter widget: remove → strip gone; cues unchanged |
| VF-10 | Failed frame slot shows placeholder | Flutter widget: extract fail → placeholder in slot, others ok |
| VF-11 | Unsupported or failed pick shows error | Flutter widget: bad pick/probe → visible error; strip not shown |
| VF-12 | Tap and highlight still work with strip | Flutter widget: with strip, token tap / highlight still work on cues |

## ADDED Requirements

### Requirement: Load and remove video from Reader chrome
Система SHALL позволять на экране Reader выбрать видеофайл с устройства (поддерживаемые контейнеры включают MP4 и MKV) через действие в верхней зоне chrome рядом с управлением split, и SHALL позволять убрать привязанное видео отдельным действием в той же верхней зоне. После успешной привязки система MUST показать полосу кадров над списком cues. После удаления система MUST скрыть полосу и MUST NOT оставлять привязку в сессии. Повторный выбор при уже привязанном видео SHALL заменить привязку (новый файл) и сбросить кэш кадров предыдущего файла для этого документа. Воспроизведение аудио и полноценный плеер не требуются.

#### Scenario [VF-01]: Load video shows frame strip
- **GIVEN** Reader открыт для документа без привязанного видео
- **WHEN** пользователь успешно выбирает поддерживаемый видеофайл
- **THEN** над списком cues появляется полоса кадров с пагинацией

#### Scenario [VF-09]: Remove video clears strip
- **GIVEN** Reader открыт с привязанным видео и видимой полосой кадров
- **WHEN** пользователь выполняет действие удаления видео
- **THEN** полоса кадров скрывается, список cues и tap/highlight продолжают работать как до загрузки видео

#### Scenario [VF-11]: Unsupported or failed pick shows error
- **GIVEN** Reader открыт без видео (или с видео)
- **WHEN** пользователь выбирает файл, который система не может использовать как видео для кадров (неподдерживаемый тип или сбой probe длительности)
- **THEN** пользователь видит заметную ошибку, полоса кадров для этого выбора не показывается (прежняя привязка не ломается без явной замены)

### Requirement: Paginated frame strip by whole-video seconds
Система SHALL строить страницы кадров по **каждой целой секунде** длительности видео от `0` до `floor(durationSeconds) - 1` (или эквивалентной дискретизации на целые секунды покрытия файла). Каждая страница MUST показывать до **трёх** слотов подряд (секунда `3·(page-1)`, `+1`, `+2`); последняя страница MAY содержать меньше трёх слотов. Система SHALL показывать элементы перехода к предыдущей/следующей странице и счётчик вида текущая/всего. Извлечение кадров MUST выполняться лениво для видимой страницы (не обязательно для всех секунд сразу); уже извлечённые секунды MAY браться из кэша (см. `library/documents`).

#### Scenario [VF-02]: Page shows three second slots
- **GIVEN** привязано видео длительностью не менее девяти секунд
- **WHEN** пользователь находится на первой странице полосы
- **THEN** видны слоты для секунд 0, 1 и 2 (три кадра или плейсхолдеры загрузки/ошибки)

#### Scenario [VF-03]: Pager advances and shows counter
- **GIVEN** привязано видео с числом секунд, дающим более одной страницы
- **WHEN** пользователь переходит на следующую страницу
- **THEN** отображаются следующие до трёх секунд, а счётчик страниц обновляется соответственно

### Requirement: Optional cue text under each frame
Для каждой секунды `t` на странице система SHALL показывать под кадром текст cue **только если** существует реплика оригинала, чей полуинтервал `[startMs, endMs)` покрывает момент `t·1000` мс (при нескольких — первая cue в порядке списка оригинала). Display-текст MUST быть join `text_lines` через `\n` (как cues-first в Reader). Если покрывающей реплики нет, система MUST оставить текстовую область пустой (это допустимое состояние, не ошибка).

#### Scenario [VF-04]: Cue text under frame when interval covers second
- **GIVEN** оригинал имеет cue с `startMs`/`endMs`, покрывающими секунду `t`
- **WHEN** страница полосы показывает слот секунды `t`
- **THEN** под кадром виден display-текст этой реплики (`text_lines` через `\n`)

#### Scenario [VF-05]: Empty text under frame is allowed
- **GIVEN** ни одна cue оригинала не покрывает секунду `t`
- **WHEN** страница полосы показывает слот секунды `t`
- **THEN** кадр (или плейсхолдер) показан без текста реплики, без сообщения об ошибке из‑за отсутствия текста

### Requirement: One-way scroll sync from frames and split show
При смене страницы кадров система SHALL прокрутить список оригинальных cues к области, соответствующей времени текущей страницы (якорь по времени первой секунды страницы или ближайшей cue через существующий nearest-`startMs` helper). Если режим split включён, система SHALL также выровнять нижнюю панель перевода к тому же временному якорю (как при sync split). При **включении** split система SHALL прокрутить нижнюю панель к текущему верху (позиция оригинала / время полосы кадров) и MUST NOT сбрасывать страницу полосы из‑за remount списка cues. Скролл списка cues или нижней панели split MUST NOT изменять страницу полосы кадров.

#### Scenario [VF-06]: Frame page change scrolls original cues
- **GIVEN** Reader с привязанным видео, список оригинала не на начале
- **WHEN** пользователь переключает страницу кадров на область с другим временем
- **THEN** список оригинала прокручивается к репликам около времени новой страницы

#### Scenario [VF-07]: Show split scrolls translation to top
- **GIVEN** Reader с привязанным видео и загруженным переводом, split выключен, полоса на странице не с начала файла
- **WHEN** пользователь включает split
- **THEN** нижняя панель перевода показывает область около текущего верха / времени полосы, а полоса кадров не сбрасывается из‑за включения split

#### Scenario [VF-08]: Cue scroll does not move frame strip
- **GIVEN** Reader с привязанным видео, полоса на странице P
- **WHEN** пользователь прокручивает список cues (или нижнюю панель split)
- **THEN** полоса кадров остаётся на странице P

### Requirement: Frame extract errors and learning gestures unchanged
Если извлечение кадра для одной секунды не удалось, система SHALL показать плейсхолдер в этом слоте и MUST NOT блокировать остальные слоты страницы и список cues. Пока полоса кадров видна, система MUST сохранять существующее поведение tap → vocab / highlight на оригинальных cues (`reader/tap`, `reader/highlight`) без требования жестов на самих кадрах.

#### Scenario [VF-10]: Failed frame slot shows placeholder
- **GIVEN** привязано видео, для одной из секунд текущей страницы extract завершается ошибкой
- **WHEN** страница отображается
- **THEN** в ошибочном слоте виден плейсхолдер, соседние слоты и cues остаются доступны

#### Scenario [VF-12]: Tap and highlight still work with strip
- **GIVEN** Reader открыт с полосой кадров и токенизированными cues оригинала
- **WHEN** пользователь тапает слово на cue под полосой
- **THEN** поведение выделения / add-vocab / highlight соответствует `reader/tap` и `reader/highlight` без регрессии из‑за полосы
