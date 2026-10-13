## Why

При чтении субтитров пользователю не хватает визуального контекста сцены. Нужна возможность привязать к документу видеофайл и **заменить** список cues оригинала страницами из **трёх вертикальных** скриншотов (по одной секунде), с текстом cue под кадром по правилу «конец реплики», без полосы 3-в-ряд и без бесконечного списка всех секунд сразу.

## What Changes

- На Reader — действия загрузить / убрать видео рядом со split.
- После успешной загрузки список cues оригинала **заменяется** колонкой из **трёх** кадров (секунды `t…t+2`), переключение остальных — **стрелками** (pager).
- Кадры по **каждой целой секунде** видео; под кадром — текст cue **только если** `round(endMs/1000)` этой реплики равен секунде кадра; несколько cue с одним концом — оба текста под одним кадром; **без** метки времени под кадром.
- Сразу после Load / restore — открыть **страницу**, содержащую секунду текущего места чтения (не обязательно с начала файла).
- Remove video — снова обычный cue list на том же временном якоре.
- Кадры **только** в панели оригинала; при video + split: смена страницы / секунды оригинала двигает низ (перевод); скролл низа **не** двигает оригинал.
- Привязка видео и кэш JPEG посещённых секунд переживают выход/вход.

**BREAKING (относительно infinite interleaved list):** убирается прокрутка всех секунд подряд; снова pager по 3, но кадры **столбцом** (не горизонтальная полоса над cues).

**BREAKING (относительно interval-cover текста):** текст привязан к **округлянному концу** cue, не к покрытию `[start, end)`.

## Scope

- **Capability ID:** `reader/video-frames` — paged vertical triple (3 секунды в столбец + стрелки), replace original, текст по end-second, без timestamp, jump-on-load, sync с split, tap/highlight на тексте под кадром.
- **Capability ID:** `library/documents` — persist привязки и JPEG-кэша; restore; cascade; re-pick; якорь страницы/секунды для restore.
- **Слои:** Flutter UI + Drift / platform storage; decode native FFmpeg-kit / web canvas за существующим Dart-фасадом. **Без** нового Rust/FRB video API.

## Out of scope

- Полноценный видеоплеер, scrubber, аудиодорожка.
- Копирование всего видеофайла в sandbox.
- Извлечение / хранение кадров в Rust; WASM WebCodecs в crate.
- Кадры в нижней панели split.
- Горизонтальная полоса 3-в-ряд над неизменным cue list.
- Бесконечный virtualized list всех секунд без pager.
- Обязательный decode всех секунд при первой загрузке (только lazy для текущей страницы + кэш).
- Облачный sync видео и кадров.
- Изменение семантики offline translate / vocab CRUD вне текста под кадром.

## Capabilities

### New Capabilities

- `reader/video-frames`: replace original → страница из 3 вертикальных кадров + стрелки; текст по `round(endMs/1000)`; без timestamp; jump к странице reading time; remove → cue list; one-way split sync; load/remove в chrome; platform decode.

### Modified Capabilities

- `library/documents`: persist video attachment + JPEG cache; restore в Reader как paged triple column; cascade; re-attach; якорь last page / second.

## Impact

- Flutter: переписать presentation с infinite `VideoSecondTimeline` на paged column of 3 + arrows; правило текста end-second; убрать timestamp; extract/facade/Drift cache переиспользовать.
- Зависимости: уже выбранный `ffmpeg_kit_flutter_new_video` + `package:web`; без новых Rust/FRB video API.
- Main specs (после sync): обновить `reader/video-frames` и `library/documents`.

## References

- Sibling app: `../my_translator_app/AGENTS.md`.
- Meta: `docs/projects-map.md`; skills `work-with-reader` / `video-frames.md`, `work-with-pages`, `work-with-drift`.
- Explore (2026-10-13): столбец по 3 + стрелки; текст по концу cue; без timestamp; replace cue list.
