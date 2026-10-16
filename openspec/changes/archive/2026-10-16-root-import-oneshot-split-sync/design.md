## Context

См. `proposal.md` — Why. **Baseline до этого change:** `LibraryScreen` (import только при `folderId != null`, LD-02 reject), Drift `Documents.folderId` NOT NULL (schema 6), `ReaderScreen` с settle-sync субтитров (RS-06), video page→перевод (VF-06), общий `_chapterIndex` для EPUB split (RS-14). **После apply:** schema 7 + nullable `folderId` / root import + mixed list; one-shot sync только Show / load перевода; dual EPUB chapter indices; meta skills (task 5.1) приведены к канону. Skills: `work-with-reader` (split-scroll / video-frames / chapters), `work-with-drift`, `work-with-pages`.

## Goals / Non-Goals

**Goals:**
- One-shot sync панелей только при Show split и при успешной загрузке перевода; убрать realtime scroll / video-page / EPUB pager sync.
- Независимые индексы глав EPUB после one-shot.
- Корневые документы: nullable `folderId`, import/list/open в корне, смешанный UI-список.
- Только Flutter `lib/` + Drift; без Rust/FRB.

**Non-Goals:**
- Скрытая Inbox-папка вместо NULL.
- Sync при Load video.
- Persist split on/off / divider.
- FRB codegen / правки `rust/`.

### Rust / FRB boundary (explicit)
- Library root / nullable `folderId` / mixed list — **Drift + Flutter only**; в `rust/` нет user persist (нет sqlite/diesel/folder API).
- Open документа и load перевода продолжают вызывать **уже существующие** FRB `parse_subtitles` / `parse_epub` (`CueDto.start_ms` / главы) **без** изменения сигнатур и без нового pairing API.
- One-shot time-nearest — Dart `nearestCueIndex` + scroll jump; EPUB chapter one-shot — Flutter индексы панелей. Realtime sync снимается в UI, не в Rust.

## Decisions

### D1 — One-shot sync, no settle listeners
- **Выбор:** оставить helper `nearestCueIndex` + `_syncTranslationToTimeMs` / `_syncPanes` **только** как явные вызовы из:
  1. `_toggleSplit` при **enabling** + есть перевод (субтитры: time-nearest; video: текущая секунда → перевод; EPUB: align chapter — см. D2);
  2. успешный `_loadTranslation` при уже включённом split (субтитры **и** EPUB — см. D2; сейчас EPUB load early-return без align из‑за shared index).
- **Убрать sync из (полный список call sites в `reader_screen.dart`):**
  - `_onScrollNotification` → `_syncPanes` на `ScrollEndNotification` (persist reading anchor на settle оригинала **оставить**; `_userInitiatedScroll` arm можно оставить только для persist);
  - `_onVideoPageChanged` → `_syncTranslationToTimeMs`;
  - `_loadVideo` post-frame sync после успешного attach (Non-Goals / Out of scope: **нет** sync при Load video);
  - `_removeVideo` post-frame `_syncPanes` при restore cues;
  - `_restoreInitialReadingAnchor` → sync низа (RS-12: open восстанавливает только original; низ — при следующем Show / load перевода).
- **Почему:** продуктовый канон explore — realtime лишний; one-shot достаточно для старта сверки.
- **Альтернатива:** оставить VF-06 / RS-06 — отклонено.

**Точки врезки:** `lib/features/reader/reader_screen.dart` (`_onScrollNotification`, `_syncPanes`, `_onVideoPageChanged`, `_toggleSplit`, `_loadTranslation`, `_loadVideo`, `_removeVideo`, `_restoreInitialReadingAnchor`).

### D2 — EPUB: два индекса главы
- **Выбор:** заменить общий `_chapterIndex` на `_originalChapterIndex` + `_translationChapterIndex` (0-based). Pager каждой панели вызывает `_goToChapter(pane, index)` только для своей панели (сейчас `_goToChapter(int)` двигает оба через shared state). One-shot: `_translationChapterIndex = clamp(_originalChapterIndex, 0, translationN-1)` при Show / load.
- **Book `_toggleSplit`:** сейчас только flip `_splitEnabled` (shared index = «уже выровнено»). После dual indices при **enabling** + есть translation chapters MUST выставить translation index (RS-19).
- **EPUB `_loadTranslation`:** после attach MUST one-shot align translation chapter (RS-20); mismatched N — по-прежнему error + clamp (RS-16).
- **Resume (RS-12 / RC-06):** восстанавливать только `_originalChapterIndex`; translation выравнивается при следующем Show (или сразу при load, если split уже on). Persist `lastReadStartMs` — только original chapter.
- **Почему:** иначе «убрать sync» невозможно при shared state.
- **Альтернатива:** shared index — отклонено (D6 explore).

**Точки:** `reader_screen.dart` book panes / `_goToChapter` / `_toggleSplit` (book branch) / `_loadTranslation` (book branch) / persist original chapter only.

### D3 — Video pages: page change без sync
- **Выбор:** `_onVideoPageChanged` только обновляет `_videoCurrentSecond` + persist `last_second`; **не** вызывает `_syncTranslationToTimeMs`. One-shot при Show / load **перевода** использует текущую секунду страницы. Load/remove video MUST NOT sync перевод (см. D1).
- **VF-07** уже соблюдён (bottom ↛ pages) — сохранить.
- **Альтернатива:** sync на page change — отклонено.

### D4 — Root documents: nullable `folderId`
- **Выбор:** Drift migration **schema 7**: `Documents.folderId` → `integer().nullable().references(Folders, #id)()` (NULL = корень). `importIntoFolder(folderId: null)` разрешён (убрать throw LD-02); insert через `Value(folderId)` / `Value(null)` после codegen.
- **List API (один канон):** `listVisible({int? folderId})` — `folderId == null` → `documents.folderId.isNull()`, иначе `equals(folderId)`; по-прежнему скрывать translation satellites. `listInFolder(id)` MAY остаться thin wrapper → `listVisible(folderId: id)`. Не плодить параллельные `listAtRoot` + `listInFolder` + `listVisible` без нужды.
- **Migration (SQLite):** `ALTER` не умеет снять NOT NULL с существующей колонки. Schema 7: **table recreate** (create new table → copy rows → drop old → rename) или эквивалентный `customStatement` в `onUpgrade` `from < 7`; **не** `migrator.addColumn` (колонка уже есть). Существующие ряды сохраняют ненулевой `folderId`. После правки tables — Drift codegen (`.g.dart`).
- **attachTranslation:** копировать `original.folderId` (включая NULL) на satellite — уже `folderId: original.folderId`; после nullable типов companion это продолжит работать.
- **folder cascade delete:** без изменений для корневых docs (они вне subtree / `folderId.isIn(subtreeIds)` их не задевает).
- **Почему:** «от корня в базе» без фантомной Inbox.
- **Альтернатива:** системная папка Inbox — отклонено (proposal Out of scope).

**Точки:** `lib/data/database/tables.dart`, `app_database.dart` (version 7 + recreate migration), `document_repository.dart`, тесты `test/data/document_repository_test.dart`.

### D5 — Library UI: смешанный список
- **Выбор:** в корне и в папке один `ListView`: сначала папки, затем видимые документы (UI уже так рисует; сейчас корень всегда `_documents = []` — подключить `listVisible(folderId: null)`). Import CTA в AppBar **и в корне**, и в empty root (рядом с Create folder). `_importFile`: убрать early-return при `_currentFolderId == null`; вызывать import с `folderId: _currentFolderId` (null = корень).
- **Empty root (LF-11):** Create folder + Import; empty folder (не корень) — Create folder (+ Import из AppBar при открытой папке).
- **Точки:** `lib/features/upload/library_screen.dart`; виджет-тесты `test/features/library_screen_test.dart`.

### D6 — Meta skills alignment (после кода или в том же change)
- Обновить канон в meta skills, чтобы не противоречили one-shot + root docs:
  - Reader: `split-scroll.md`, `video-frames.md`, `chapters.md`, краткие упоминания в `work-with-reader/SKILL.md`
  - Library: `work-with-pages/upload.md` (root import, mixed list, empty root Import+Create folder)
  - Drift: `work-with-drift/schema.md` (schema 7, nullable `folderId`), `repositories.md` (`listVisible` / import null folderId)
- До task 5.1 при apply: **delta specs + этот design перекрывают** устаревшие формулировки skills (не следовать realtime RS-06/VF-06/RS-14 sync и reject LD-02 из skills).
- Чеклист — `tasks.md`; детали не дублировать здесь.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| SQLite не снимает NOT NULL через ALTER | Schema 7: table recreate / `customStatement` (D4); unit-тест import root + upgrade 6→7 |
| Тесты RS-06 / RS-14 / VF-06 ожидают старый sync | Переписать под новые сценарии (RS-06 no-sync, RS-14 independent, VF-06 no page sync); добавить RS-18/19/20, LF-09/11, LD-02/06 root |
| Skills meta устареют относительно кода | D6 + task на правку skills в meta |
| Пользователь ждёт sync при листании video | Зафиксировано в specs; one-shot только Show/load |

## Migration Plan

1. Bump Drift schema 6 → 7, nullable `folderId`; существующие ряды остаются с ненулевым `folderId`.
2. UI/repo позволяют root import; rollback = не деплоить / откат миграции только через restore backup (как обычно для Drift user DB).

## Open Questions

Нет (explore D1–D6 закрыты; defaults: нет sync на load video; папки сверху; empty root Import + Create folder).

## Tasks

Чеклист реализации — `tasks.md`.
