## Why

Realtime sync панелей в split мешает свободному чтению (особенно со скриншотами видео и EPUB): перевод постоянно «догоняет» оригинал. Одновременно нельзя быстро положить файл в библиотеку — обязательна папка, хотя папки нужны лишь для опциональной группировки.

## What Changes

- Sync панелей split — **только один раз** при Show split и при успешной загрузке перевода; settle-scroll, video page→перевод и EPUB pager↔pager sync убираются.
- После one-shot align панели независимы (субтитры: независимый скролл; EPUB: свой индекс главы на панель).
- Import субтитров/EPUB **в корне** Library без создания папки; в корне и в папке — смешанный список папок и документов.

## Scope

- **Capability ID:** `reader/split` — one-shot sync при Show и при load перевода; удаление realtime sync scroll (субтитры) и chapter-pager sync (EPUB).
- **Capability ID:** `reader/video-frames` — при video pages нет sync перевода на смену страницы; one-shot остаётся только через `reader/split` (Show / load перевода).
- **Capability ID:** `reader/chapters` — pager двигает только свою панель; ссылка на общий индекс глав в split снимается в пользу независимых индексов после one-shot.
- **Capability ID:** `library/documents` — документы в корне библиотеки (без обязательной папки); list/open/delete/translation attach для корневых docs.
- **Capability ID:** `library/folders` — корень/папка показывают смешанный список (папки + документы); empty root с Import и Create folder.
- **Слои:** Flutter UI (Library, Reader split / video / book) + Drift persist; Rust/FRB **без** изменений.

## Out of scope

- Облачный sync / multi-device.
- Persist режима split on/off и позиции разделителя.
- Замена перевода без delete → load.
- Новый FRB / parse API; изменения vocabulary / tap / highlight / offline translate.
- Auto-создание «Inbox»-папки вместо корневых документов (канон — документ без папки в корне).
- Sync при Load video (без Show / load перевода) — не требуется.
- Сортировка сложнее «папки сверху, затем документы» (имя) — не в этом change.

## Capabilities

### New Capabilities

_(нет)_

### Modified Capabilities

- `reader/split`: one-shot sync панелей при Show split и при успешной загрузке перевода; без realtime scroll/chapter sync.
- `reader/video-frames`: убрать требование sync перевода при смене video page (VF-06); сохранить независимость низа от скролла оригинала при video.
- `reader/chapters`: pager меняет только текущую панель; split chapter-index realtime sync снимается (align — через `reader/split` one-shot).
- `library/documents`: разрешить import/list/open корневых документов; отменить reject «без папки».
- `library/folders`: UI корня/папки — смешанный список папок и документов; empty root с обоими CTA.

## Impact

- Flutter: Library (import в корне, смешанный список); Drift schema/repos (`folderId` nullable / root); Reader — убрать settle listeners sync, video page sync, общий EPUB chapter index → независимые индексы + one-shot align.
- Rust/FRB: нет изменений; open/load перевода по-прежнему через существующие `parse_subtitles` / `parse_epub` (без pairing API).
- Main specs (после sync): обновлённые перечисленные capabilities.

## References

- Sibling app: `../my_translator_app/AGENTS.md`.
- Meta: `docs/projects-map.md`; skills `work-with-reader` (split-scroll, video-frames, chapters), `work-with-drift`, `work-with-pages`.
- Explore (эта сессия): D1 полный off realtime; D2 корень в БД; D3 смешанный список; D4 sync при load перевода; D5 убрать EPUB pager sync; D6 независимые индексы после one-shot; defaults: нет sync на load video; папки сверху; empty root Import + Create folder.
