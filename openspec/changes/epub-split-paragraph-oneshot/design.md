## Context

См. `proposal.md` — Why. **Baseline:** book `_toggleSplit` только flip `_splitEnabled` + `_alignTranslationChapterToOriginal()` (глава); paragraph `ListView` без контроллера → remount Show/Hide сбрасывает offset верха в 0; низ стартует с абзаца 0. Субтитры уже: capture `ItemPositionsListener` → `initialScrollIndex` / post-frame `jumpTo` + `_syncPanes` по `startMs`. Skills: `work-with-reader` / `split-scroll.md`, `chapters.md`.

## Goals / Non-Goals

**Goals:**
- Preserve mid-chapter scroll оригинала на Show **и** Hide (book), по аналогии с RS-10/11 субтитров.
- One-shot: глава + индекс абзаца (+ alignment) при Show / load EPUB перевода.
- После one-shot — независимый скролл абзацев (как RS-06 для cues).
- Не ломать subtitle one-shot / preserve.

**Non-Goals:**
- Realtime mirror абзацев / pager.
- Persist paragraph offset между сессиями.
- Rust/FRB / parse changes.
- Перевод book panes на другой UI-фреймворк.

### Rust / FRB boundary (explicit)
- Только Flutter `lib/`; pairing абзацев — по индексу в UI, без нового FRB API.
- **Достаточность текущего API:** `parse_epub` → `EpubChapterDto.paragraphs: Vec<String>` уже отдаёт упорядоченные абзацы главы (`rust/src/api/epub.rs`). One-shot = `clamp(sourceParagraphIndex, 0, translationChapter.paragraphs.len()-1)` + `jumpTo` — compute-only на стороне UI. Не нужны: pairing API, align FRB, изменение parse/strip, новые DTO-поля.

## Decisions

### D1 — Book panes: index scrollables (как cues)
- **Выбор:** для paragraph lists оригинала и перевода в book-режиме использовать `scrollable_positioned_list` (`ItemScrollController` + `ItemPositionsListener`) на state экрана Reader — отдельные контроллеры от cue-панелей или переиспользование паттерна с book-specific instances.
- **Почему:** top-visible index + `itemLeadingEdge` + `jumpTo` уже канон для split; голый `ListView` + `ScrollController.offset` хуже переживает remount и разную высоту строк.
- **Альтернатива:** `ScrollController` + pixels — отклонено (хрупко при разной высоте абзацев / смене высоты pane).
- **RC-02:** смена главы по-прежнему remount списка → старт главы (ключ с chapter index); one-shot Show **не** меняет главу оригинала.

### D2 — Capture / restore на book Show и Hide
- **Выбор:** в book-ветке `_toggleSplit` перед `setState` захватить top-visible paragraph index + leading edge оригинала; после layout — `jumpTo` на remounted original list (как subtitle path). Hide — тот же capture/restore.
- **Почему:** смена `body` (`_buildBookPane` ↔ `_buildBookSplitBody`) dispose'ит Element; local `Key` не сохраняет `ScrollPosition`.

### D3 — One-shot: chapter then paragraph
- **Выбор:** при enabling + translation chapters / при успешном load EPUB перевода при split on:
  1. `_translationChapterIndex = clamp(_originalChapterIndex, …)` (как сейчас);
  2. после attach перевода: `jumpTo` (или `scrollTo`+`jumpTo`) на translation list к `clamp(sourceParagraphIndex, 0, translationChapterLen-1)` с alignment источника.
- **Источник якоря:** захваченный при Show (не перечитывать listener сразу после remount — риск «ещё 0», тот же урок что у cues).
- **Неравное число абзацев:** silent clamp (RS-22); отдельный error UI не требуется (в отличие от mismatched chapter N).
- **Альтернатива:** sync только fraction viewport — отклонено (индекс абзаца ближе к cue-index паттерну и tap/highlight).

### D4 — Нет settle-sync для book paragraphs
- **Выбор:** `_onScrollNotification` по-прежнему не вызывает sync панелей (D1 из `root-import-oneshot-split-sync`); book one-shot только из toggle enabling / load translation.
- **Почему:** продуктовый канон — one-shot only.

### D5 — Skills / meta
- `split-scroll.md` / `chapters.md` / `work-with-reader/SKILL.md` / `work-with-pages/reader.md` согласованы с delta (one-shot EPUB = chapter + paragraph; preserve mid-chapter на Show/Hide). При apply — сверить ещё раз.

### FE hook map (current HEAD → target)

Книжный и субтитровый режимы **взаимоисключающи** (`_isBookMode`), но для book — **отдельные** `ItemScrollController` / `ItemPositionsListener` (не шарить cue-контроллеры: проще и безопаснее для тестов/dispose).

| Hook | HEAD | Target |
|------|------|--------|
| `_toggleSplit` book early-return | flip `_splitEnabled` + `_alignTranslationChapterToOriginal()` only | Show **и** Hide: capture top-visible paragraph (`_topVisible` pattern) → seed `initialScrollIndex`/`initialAlignment` → post-frame `jumpTo` оригинала; при Show + translation chapters — chapter align **+** paragraph `jumpTo` перевода из **захваченного** якоря |
| `_buildBookPaneContent` list | bare `ListView.builder` (+ chapter key) | `ScrollablePositionedList.builder` + book controllers/listeners; сохранить keys `readerBookParagraphList_$index` / translation twin (RC-02 remount на смену главы) |
| `_loadTranslation` book + `_splitEnabled` | chapter clamp in `setState` only | после attach — post-frame one-shot paragraph `jumpTo` (как subtitle `_syncPanes` после load); якорь — текущий top-visible оригинала |
| `_alignTranslationChapterToOriginal` | chapter only | оставить; добавить paragraph jump helper (или расширить) с `clamp(p, 0, len-1)` |
| Divider (RS-17) | `ValueNotifier` heights | без изменений intent — scroll controllers на `State`, не пересоздавать на drag |
| Widget tests | `test/features/reader/reader_epub_split_test.dart` | RS-10/11/19/20/21/22; subtitle regression в `reader_screen_test.dart` |

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Remount race: listener ещё пуст → restore 0 | Seed initialScrollIndex/alignment + post-frame jumpTo; retry attach как у `_syncPanes` |
| Разная длина абзацев → «середина» визуально не совпадает | Index-first + optional alignment; не требовать pixel-identity |
| Регрессия subtitle path | Не трогать cue controllers без нужды; прогнать RS-10/18 widget tests |
| Divider drag (RS-17) | Сохранить ValueNotifier высот; контроллеры scroll живут на state, не пересоздавать на drag |

## Open Questions

_(нет — D1 explore закрыт: one-shot глава + абзац.)_
