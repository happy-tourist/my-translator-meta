## Why

При чтении субтитров пользователю не хватает визуального контекста сцены. Нужна возможность привязать к документу видеофайл и **заменить** список cues оригинала страницами из **трёх вертикальных** скриншотов (по одной секунде), с текстом cue под кадром по правилу «конец реплики», pager внизу экрана и метками времени у субтитров из файла.

## What Changes

- На Reader — действия загрузить / убрать видео рядом со split.
- После успешной загрузки список cues оригинала **заменяется** колонкой из **трёх** кадров (секунды `t…t+2`); переключение остальных — **стрелками у низа** original pane (не над кадрами).
- При смене страницы список кадров MUST прокрутиться к **первому** кадру новой страницы.
- Кадры по **каждой целой секунде** видео; под кадром — текст cue **только если** `round(endMs/1000)` этой реплики равен секунде кадра; несколько cue с одним концом — оба текста под одним кадром.
- Если под кадром есть текст cue — показать временну́ю метку **из файла** (`startMs → endMs` cue, как в обычном Reader). Если текста нет — кадр без метки времени.
- Сразу после Load / restore — открыть **страницу**, содержащую секунду текущего места чтения.
- Remove video — снова обычный cue list на том же временном якоре.
- Кадры **только** в панели оригинала; video + split: смена страницы двигает низ; скролл низа **не** двигает оригинал.
- Привязка видео и кэш JPEG переживают выход/вход.

**BREAKING (относительно infinite list / horizontal strip):** paged column of 3 + bottom pager.

**BREAKING (относительно «никогда нет timestamp»):** метка времени **у субтитра из файла**, когда текст есть; под пустым кадром метки нет.

## Scope

- **Capability ID:** `reader/video-frames` — paged vertical triple, bottom pager, scroll-to-first on page change, end-second text, cue time range from file when text present, jump-on-load, split sync, tap/highlight.
- **Capability ID:** `library/documents` — persist attachment + JPEG cache; restore; cascade; re-pick; last page/second anchor.
- **Слои:** Flutter + Drift; decode Dart FFmpeg-kit / web canvas. **Без** Rust/FRB video API.

## Out of scope

- Видеоплеер, scrubber, аудио.
- Rust/WASM video decode.
- Копия видео **в sandbox приложения / Drift media** — **in scope** (durable Web reopen); не в отдельный «облачный» storage.
- Кадры в нижней панели split.
- Горизонтальная полоса 3-в-ряд над cue list; infinite all-seconds list.
- Обязательный decode всех секунд сразу; облачный sync.

## Capabilities

### New Capabilities

- `reader/video-frames`: replace → 3 vertical frames + **bottom** arrows; scroll to first frame on page change; text by `round(endMs/1000)`; timestamp from cue file when text present; jump/remove/split sync; platform decode.

### Modified Capabilities

- `library/documents`: persist video + JPEG cache; restore paged UI; cascade; re-attach; last page/second.

## Impact

- Flutter: polish `VideoFramePages` (pager bottom, scroll controller jump, `CueRow.showTimestamp: true` when embedding cues); specs/skills; extract/Drift unchanged.
- Main specs after sync: `reader/video-frames`, `library/documents`.

## References

- Sibling app: `../my_translator_app/AGENTS.md`.
- Meta: `docs/projects-map.md`; `work-with-reader` / `video-frames.md`.
- Explore (2026-10-13): bottom pager; scroll-to-first; timestamps from cue file under text.
