## Why

При чтении субтитров пользователю не хватает визуального контекста сцены. Нужна возможность привязать к документу видеофайл, показать скриншоты по секундам над списком cues и листать их страницами, не ломая tap / highlight / split.

## What Changes

- На Reader — действия загрузить / убрать видео рядом со split; после загрузки сверху полоса из трёх кадров текущей страницы, стрелки и счётчик страниц.
- Кадры берутся на **каждую секунду** длительности видео; под кадром текст cue только если секунда попадает в интервал реплики (пусто — нормально).
- Список cues и жесты изучения слов работают как сейчас; полоса кадров не реагирует на скролл cues / нижней панели split.
- Смена страницы кадров и открытие split прокручивают соответствующие панели субтитров к времени текущих кадров / текущего верха.
- Привязка видео к документу и кэш уже извлечённых кадров переживают выход и повторный вход (durable link + JPEG посещённых секунд; при мёртвой ссылке — повторный выбор файла, сохранённые кадры остаются).

## Scope

- **Capability ID:** `reader/video-frames` (новое) — UI полосы кадров на Reader, пагинация по 3, извлечение кадров по секундам (native + web), опциональный текст под кадром, односторонняя sync к списку cues / split, load/remove видео в chrome Reader.
- **Capability ID:** `library/documents` (изменяемое) — persist привязки видео к оригинальному документу (durable platform link, не обязательная копия всего файла), кэш JPEG посещённых секунд, restore при open, cascade при удалении документа, re-pick при недоступной ссылке.
- **Слои:** Flutter UI (Reader) + Drift / platform storage для метаданных и кэша кадров; decode кадров на native через Dart FFmpeg-kit, на web через Dart canvas / web interop. **Без** нового Rust/FRB video API и без линковки FFmpeg в `rust/`. Намеренный FE-only compute exception (не gap NLP/parse); persist video — Drift, как у `library/documents`.

## Out of scope

- Полноценный видеоплеер, scrubber, аудиодорожка.
- Копирование всего видеофайла в sandbox «как байты субтитров» (только durable link + JPEG-кэш).
- Извлечение / хранение кадров в Rust; WebAssembly / WebCodecs-стеки в crate.
- Изменение поведения tap → vocab, lemma highlight, offline translate.
- Синхронизация полосы кадров от скролла cues или нижней панели split (низ → верх запрещён).
- Облачный sync видео и кадров.
- Обязательный decode всех секунд при первой загрузке (только lazy по странице + кэш).

## Capabilities

### New Capabilities

- `reader/video-frames`: полоса кадров на Reader (3 на страницу, pager), сэмплинг по секундам, текст под кадром по покрытию cue, load/remove, platform decode (native FFmpeg-kit / web canvas), sync сверху вниз при смене страницы и при Show split.

### Modified Capabilities

- `library/documents`: persist video attachment (durable link + frame JPEG cache) у оригинала; restore; cascade delete; re-attach при мёртвой ссылке без потери уже сохранённых кадров.

## Impact

- Flutter: Reader chrome + strip UI (strip вне remount Show/Hide split); оркестрация pick → usable path / probe / extract page / cache; Drift schema/repos для video link и frame cache; web facade извлечения кадра (по аналогии с `lib/data/translate/` platform split).
- Зависимости: native FFmpeg-kit (Dart); web — `package:web` / canvas interop; без изменений публичного Rust FRB API для video. Native extract только по filesystem path (sandbox copy при `content://`).
- Main specs (после sync): новый `reader/video-frames`; обновлённый `library/documents`.

## References

- Sibling app: `../my_translator_app/AGENTS.md`.
- Meta: `docs/projects-map.md`; skills `work-with-pages` (reader), `work-with-reader` (split-scroll), `work-with-drift`, `work-with-env-deploy`.
- Explore: решения D1–D10, P1 (JPEG-кэш + durable link); sync кадры/Show split → cues, не наоборот.
