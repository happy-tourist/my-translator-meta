## Why

Reader уже показывает оригинал субтитров одним списком, но пользователю не с чем сверять готовый перевод. Нужен режим разделённого экрана: сверху оригинал, снизу догруженный переводный файл с синхронным скроллом. При чтении позиция не должна сбрасываться при переключении split и при повторном открытии того же файла — пользователь ожидает вернуться к месту, где остановился.

## What Changes

- На экране Reader — включение/выключение горизонтального разделения (сверху оригинал, снизу перевод) и изменение положения разделителя.
- Нижняя панель: пустое состояние с загрузкой файла → отображение cues как сверху; удаление перевода; ошибки загрузки/парсинга заметны внизу (как при open в Library).
- Синхронизация скролла: сначала по ближайшему таймингу (`start`), затем пиксельная подстройка на settle.
- Переводный файл сохраняется в той же папке, что и оригинал, восстанавливается при повторном открытии; в списке папки виден только оригинал (перевод из нижней панели скрыт); удаление оригинала удаляет и перевод.
- При **включении и выключении** split позиция прокрутки оригинала сохраняется в сессии (не сброс в начало из‑за remount списка).
- Удаление перевода — в верхней зоне нижней панели (не в AppBar).
- **Resume:** позиция чтения оригинала сохраняется между сессиями приложения; при повторном открытии файла Reader сразу показывает место остановки (якорь по `start` реплики).

## Scope

- **Capability ID:** `reader/split` (новое) — split UI, load/delete перевода в сессии Reader, sync scroll (time → pixel settle), preserve scroll на toggle Show/Hide, restore якоря при open.
- **Capability ID:** `library/documents` (изменяемое) — скрытый satellite-документ перевода; связь original↔translation; фильтрация списка; cascade delete; restore перевода при open; **persist якоря чтения** оригинала (`start` + alignment).
- **Слои:** Flutter UI (Reader / split) + Drift persist документов; parse перевода через уже существующий FRB `parse_subtitles` / `parseSubtitles` (capability `parse/subtitles` без delta; без нового compute-API и без правок `rust/`).
- **Форматы перевода:** те же, что у import/open (SRT, VTT, ASS/SSA) — тот же Dart `CueDto.startMs` (`BigInt`) для time-sync и resume.

## Out of scope

- Тап по слову / карточка / lemma highlight / офлайн-перевод слова.
- Замена перевода без удаления (только delete → снова загрузить).
- Выравнивание по индексу cue как единственный режим (канон skills `split-scroll.md` остаётся target-reference; в этом change — time-first на Flutter).
- Pairing / align / parallel-lists API в Rust; FRB codegen для этого change; видеоплеер; облачный sync.
- Persist режима split on/off и позиции разделителя между сессиями приложения.
- Дублирующая кнопка Delete в AppBar (только chrome нижней панели).

## Capabilities

### New Capabilities

- `reader/split`: режим split на Reader (toggle, resize, empty/load/delete нижней панели, sync по таймингу затем пиксели на settle; preserve scroll на Show/Hide; jump к сохранённому якорю при open).

### Modified Capabilities

- `library/documents`: хранение скрытого переводного документа, привязка к оригиналу, список папки без перевода из Reader, cascade при удалении оригинала, восстановление перевода при open оригинала, persist якоря чтения оригинала.

## Impact

- Flutter: Reader — split + оркестрация load/parse/persist перевода и reading resume; Library open передаёт/восстанавливает якорь; Drift schema/repos — связь original↔translation + поля якоря; sync helper `nearestCueIndex` на `CueDto.startMs` (`BigInt`).
- Rust/FRB: **нет задач** — повторный вызов существующего `parse_subtitles`; FE-only Drift/UI без нового BE ожидаемы и не блокируют реализуемость.
- Main specs (после sync): новый `reader/split`; обновлённый `library/documents` (не `parse/subtitles`).

## References

- Sibling app: `../my_translator_app/AGENTS.md` (target: split / sync scroll).
- Meta: `docs/projects-map.md`; skills `work-with-reader` (split-scroll.md), `work-with-pages` (reader.md), `work-with-drift`.
- Explore: resume при reopen (D1/D6); defaults D2–D5 (якорь `startMs`+alignment; write на settle+pop; restore оригинала + sync низа; nearest при сдвиге cues); D7 Hide-split preserve; design D1–D11.
