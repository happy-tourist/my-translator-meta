## Why

Reader уже показывает оригинал субтитров одним списком, но пользователю не с чем сверять готовый перевод. Нужен режим разделённого экрана: сверху оригинал, снизу догруженный переводный файл с синхронным скроллом, чтобы ориентироваться при чтении.

## What Changes

- На экране Reader — включение/выключение горизонтального разделения (сверху оригинал, снизу перевод) и изменение положения разделителя.
- Нижняя панель: пустое состояние с загрузкой файла → отображение cues как сверху; удаление перевода; ошибки загрузки/парсинга заметны внизу (как при open в Library).
- Синхронизация скролла: сначала по ближайшему таймингу (`start`), затем пиксельная подстройка на settle.
- Переводный файл сохраняется в той же папке, что и оригинал, восстанавливается при повторном открытии; в списке папки виден только оригинал (перевод из нижней панели скрыт); удаление оригинала удаляет и перевод.

## Scope

- **Capability ID:** `reader/split` (новое) — split UI, load/delete перевода в сессии Reader, sync scroll (time → pixel settle).
- **Capability ID:** `library/documents` (изменяемое) — скрытый satellite-документ перевода в папке оригинала, связь original↔translation, фильтрация списка, cascade delete, restore при open.
- **Слои:** Flutter UI (Reader / split) + Drift persist документов; parse перевода через уже существующий FRB `parse_subtitles` / `parseSubtitles` (capability `parse/subtitles` без delta; без нового compute-API и без правок `rust/`).
- **Форматы перевода:** те же, что у import/open (SRT, VTT, ASS/SSA) — тот же Dart `CueDto.startMs` (`BigInt`) для time-sync.

## Out of scope

- Тап по слову / карточка / lemma highlight / офлайн-перевод слова.
- Замена перевода без удаления (только delete → снова загрузить).
- Выравнивание по индексу cue как единственный режим (канон skills `split-scroll.md` остаётся target-reference; в этом change — time-first на Flutter).
- Pairing / align / parallel-lists API в Rust; FRB codegen для этого change; видеоплеер; облачный sync.
- Persist позиции разделителя и позиции скролла между сессиями приложения.

## Capabilities

### New Capabilities

- `reader/split`: режим split на Reader (toggle, resize, empty/load/delete нижней панели, sync по таймингу затем пиксели на settle).

### Modified Capabilities

- `library/documents`: хранение скрытого переводного документа, привязка к оригиналу, список папки без перевода из Reader, cascade при удалении оригинала, восстановление перевода при open оригинала.

## Impact

- Flutter: Reader получает режим split и оркестрацию load/parse/persist перевода (inject `DocumentRepository` + optional parse fn); Library list фильтрует скрытые translation-docs; Drift schema/repos — связь original↔translation; sync helper `nearestCueIndex` на `CueDto.startMs` (`BigInt`).
- Rust/FRB: **нет задач** — повторный вызов существующего `parse_subtitles`; FE-only Drift/UI без нового BE ожидаемы и не блокируют реализуемость.
- Main specs (после sync): новый `reader/split`; обновлённый `library/documents` (не `parse/subtitles`).

## References

- Sibling app: `../my_translator_app/AGENTS.md` (target: split / sync scroll).
- Meta: `docs/projects-map.md`; skills `work-with-reader` (split-scroll.md), `work-with-pages` (reader.md), `work-with-drift`.
- Explore / design decisions D1–D6 (empty+load, persist hidden in folder, cascade, nearest `startMs` → pixel settle-only, delete unlinks+removes file, Library-like errors).
