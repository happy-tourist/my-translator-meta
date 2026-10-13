## Why

При чтении субтитров пользователю не хватает визуального контекста сцены. Нужна возможность привязать к документу видеофайл и **вплести** скриншоты по секундам в поток чтения оригинала (кадр под кадром), с переходом к текущей позиции чтения, не ломая tap / highlight и не дублируя кадры в панели перевода.

## What Changes

- На Reader — действия загрузить / убрать видео рядом со split.
- После успешной загрузки список cues оригинала **заменяется** вертикальным потоком: на **каждую целую секунду** видео — кадр, под ним опциональный текст cue (если секунда покрыта интервалом реплики; пусто — нормально, кадр всё равно в списке) и временная метка.
- Сразу после Load (и при restore) поток **открывается на секунде, соответствующей текущей позиции чтения** (время видимого верха cues / якоря до transform).
- Remove video — снова обычный cue list на **том же** временном якоре.
- Кадры **только** в панели оригинала; нижняя панель split без кадров.
- При video + split: скролл interleaved-оригинала двигает низ (перевод); скролл низа **не** двигает оригинал.
- Привязка видео и кэш JPEG посещённых секунд переживают выход/вход (durable link; при мёртвой ссылке — re-pick, кэш остаётся).

**BREAKING (относительно первой реализации strip):** убирается полоса «3 кадра в ряд над cues» и pager страниц по 3; оригинал при video преобразуется, а не дополняется верхней панелью.

## Scope

- **Capability ID:** `reader/video-frames` — interleaved second timeline в original pane, load/remove, lazy extract, jump-on-load, sync rules с split, tap/highlight на тексте под кадром.
- **Capability ID:** `library/documents` — persist привязки и JPEG-кэша (без обязательной копии всего видео); restore; cascade; re-pick; якорь секунды вместо page-of-3 при необходимости.
- **Слои:** Flutter UI + Drift / platform storage; decode native FFmpeg-kit / web canvas за существующим Dart-фасадом. **Без** нового Rust/FRB video API.

## Out of scope

- Полноценный видеоплеер, scrubber, аудиодорожка.
- Копирование всего видеофайла в sandbox.
- Извлечение / хранение кадров в Rust; WASM WebCodecs в crate.
- Кадры в нижней панели split.
- Обязательный decode всех секунд при первой загрузке (только lazy вокруг viewport + кэш).
- Облачный sync видео и кадров.
- Изменение семантики offline translate / vocab CRUD вне текстыпод-кадром tap.

## Capabilities

### New Capabilities

- `reader/video-frames`: вертикальный поток «секунда → кадр → текст|пусто»; replace original; jump к текущему времени при load/restore; remove → cue list; sync original→translation при video, не наоборот; load/remove в chrome; platform decode.

### Modified Capabilities

- `library/documents`: persist video attachment + JPEG cache; restore в Reader как interleaved timeline; cascade; re-attach; optional `last_second` (вместо page-of-3).

## Impact

- Flutter: заменить `VideoFrameStrip` (3-в-ряд) на virtualized second list в original body; оркестрация load → jump; remove → restore cues; split sync rules; существующий extract/facade/Drift cache переиспользовать.
- Зависимости: уже выбранный `ffmpeg_kit_flutter_new_video` + `package:web`; без новых Rust/FRB video API.
- Main specs (после sync): обновить `reader/video-frames` и `library/documents`.

## References

- Sibling app: `../my_translator_app/AGENTS.md`.
- Meta: `docs/projects-map.md`; skills `work-with-reader` / `video-frames.md`, `work-with-pages`, `work-with-drift`.
- Explore (revision): D1 A секунды вертикально; replace; jump-on-load; frames only original; original→bottom sync, bottom↛original.
