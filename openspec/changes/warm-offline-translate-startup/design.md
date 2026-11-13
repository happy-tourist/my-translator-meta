## Context

См. `proposal.md` — Why / Scope. Уже в коде: process-wide ensure/warmUp, **fullscreen gate** в `AppShell`, native **extract → app-local dir → `init_offline_translate_from_dir`** (D8 revised / D13; bytes FRB MAY for tests), Web worker warmup, Reader zero-sync / bottom anchor / top-linked pager / AppBar delete.

Explore (до path-fix): на **native** gate-spinner был «мёртвый» — UI isolate занят ~228MB load/copy + Dart→Rust encode. Пользователь: **оставить лоадер**; вариант B — доработать cold path. Path/file init сильнее, чем ещё меньше chunk yields при полном FRB bytes.

Explore (Reader, 2026-11-13): при reopen низ EPUB помнит mid-chapter, верх — глава с абзаца 0 (LD-28 only). Решение: persist верха **как низа** (абзац + alignment); убрать fixed `chapter.title` chrome; pager оставить. Не sync и не «подскролл» на open — только запомнить и открыть на месте.

Платформы: Web (Chrome), Windows, Android. Фокус freeze-fix: **native**. Reader delta: Flutter/Drift.

## Goals / Non-Goals

**Goals:**

- Fullscreen loading после Drift open до ready/fail; затем shell (**gate остаётся**).
- Shared process-wide ensure; Translate wait + spinner.
- **Native:** во время gate `CircularProgressIndicator` остаётся живым (кадры) — не блокировать UI isolate полным bytes transfer модели.
- Reader: zero sync; bottom persist; **original EPUB mid-chapter persist/restore**; top-only linked pager; text-only bottom; **original без fixed title chrome**; Delete у split.

**Non-Goals:**

- Убрать gate / тихий-only product default.
- ORT внутри `init_app` / `RustLib.init`.
- Persist split/divider.
- Убирать pager / стрелки у оригинала.
- Realtime scroll mirror; video sync.
- Менять Web pipeline (Transformers.js worker) кроме регрессии.

## Decisions

### D1 — Fullscreen loading gate (подтверждено)

После Drift open — fullscreen loading до ensure ready/fail. Shell недоступен до снятия. Fail → dismiss без обязательного SnackBar. Escape hatch для тестов.

**Explore:** предложение убрать gate **отклонено** («оставь лоадер»).

### D2 — Shared init future

Без изменений: один process-wide Future на native+Web.

### D3 — Cold path yields (недостаточен один)

Прежний bytes+chunked yield **не** закрывает TO-11 на реальных ~200MB. Оставляем yields где ещё нужны (копия на диск), но главный фикс — D8/D13 (path), не «ещё мельче chunks» как единственная мера.

### D4 — Translate button UX

Без изменений: spinner на кнопке при wait.

### D5 — Точки врезки (обновлено)

| Зона | Файлы / роль |
|------|----------------|
| Gate | `lib/app/shell.dart` — fullscreen до ensure (без смены UX) |
| Native facade | `lib/data/translate/offline_translate_native.dart` (+ helpers): extract assets → app support/cache dir с yield; вызвать path/dir FRB init |
| FRB / Rust | `rust/src/api/translate.rs` + `translate/`: init from OS paths / model dir; ORT `Session` from files; codegen |
| Web | без обязательных правок worker |
| Reader / Drift | D9–D12 (уже) + **D14 mid-chapter original** + **D15 title chrome** |
| Tests | TO-11; Rust path-init; shell gate; LD-33 / RS-12 / RC-13 |

### D6 — Warm-up / gate errors

Без изменений: fail → dismiss gate; Translate показывает kind.

### D7 — Тесты

- TO-09/13 gate; TO-10; **TO-11 живой индикатор на native cold path**; TO-14 path init; RS/RC/LD + **LD-33 / RS-12 mid-chapter original / RC-13**.

### D8 — Rust / FRB (**REVISED**)

**Было:** verify-only; только `Vec<{name,bytes}>`; запрет OS-path.

**Стало:** для **native product cold path** публичный FRB SHALL принимать **пути к файлам модели** (или один directory + фиксированные basenames), чтобы не гонять ~200MB через Dart UI isolate. Bytes-init (`init_offline_translate` с bytes) MAY остаться для unit-тестов / compat, но **product ensure/warm-up MUST NOT** полагаться на полный bytes transfer как единственный путь.

Не класть ORT в `init_app`. Web по-прежнему не вызывает FRB translate.

### D9–D12 — Reader (zero-sync / bottom / pager / delete)

Zero one-shot sync; top-linked chapter; translation reading anchor; bottom text-only + Delete у split — уже в коде. Intent без изменений; mid-chapter **оригинала** и title chrome — D14/D15.

### D13 — Native extract then path init

**Выбор:**

1. Flutter (после первого paint gate) последовательно читает/копирует required model files из AssetBundle в app-local dir (`getApplicationSupportDirectory` / эквивалент), с **chunked write + yields** (или нативный copy API, если уже в deps — не плодить лишние пакеты без нужды).
2. Если целевые файлы уже на месте и валидны (size/mtime или marker) — **пропустить** повторную полную копию (лёгкий disk reuse в рамках сессии/установок; не отдельный «product disk-cache» feature).
3. Вызвать FRB init с paths/dir → Rust ORT load from filesystem (async FRB worker; без гигантского Dart→Rust encode).
4. Shared ensure по-прежнему один; gate ждёт тот же Future.

**Отклонено как единственный фикс:** только уменьшать `_kAssetCopyChunkBytes` при полном FRB bytes — не снимает encode stall.

**Альтернатива (отклонена сейчас):** убрать gate и пустить shell под jank — пользователь оставил лоадер.

### D14 — Original EPUB mid-chapter persist (NEW)

**Проблема:** низ уже пишет paragraph + alignment в якорь translation-документа; оригинал EPUB пишет только chapter в `lastReadStartMs` (`alignment: null`) → reopen = глава k, абзац 0.

**Выбор (как низ по данным скролла):**

1. На settle / pop оригинала book-pane: capture top-visible paragraph index + `itemLeadingEdge` (зеркало `_captureBookTranslationAnchor`).
2. Persist на **оригинальном** документе: **chapter** остаётся в `lastReadStartMs` (LD-28); **paragraph index** — в новой nullable Drift-колонке (например `lastReadParagraphIndex`); **alignment** — в существующем `lastReadAlignment` (0…1), как у низа.
3. `ReadingAnchor` / `DocumentRepository` save/load расширить optional paragraph index; Library open передаёт полный якорь в `ReaderScreen.initialReadingAnchor`.
4. `initState` book-mode: besides chapter restore, seed `_bookOriginalInitialScrollIndex` / `_bookOriginalInitialAlignment` из якоря (clamp); список открывается на месте через `initialScrollIndex` / `initialAlignment` (и jump-retry только если уже есть паттерн для remount — **не** отдельный «scroll to start» на open).
5. Смена главы через pager (`_goToChapter`) по-прежнему rewind к 0 (RC-12) и пишет новый chapter (+ paragraph 0).

**Отклонено:** pack chapter+paragraph в один `lastReadStartMs` (хрупко); класть paragraph в `lastReadAlignment` (ломает 0…1 leading edge).

**Миграция:** schema bump + nullable column; старые ряды без paragraph → open глава с начала списка (как сейчас).

### D15 — Убрать fixed chapter title на оригинале (NEW)

В `_buildBookPaneContent` для `fromOriginal: true` убрать `Padding` + `Text(chapter.title)` над списком. Оставить `Expanded(list)` + SafeArea pager (prev / `n / N` / next). Низ уже text-only. Title в контенте абзацев EPUB не трогать.

Точки: `lib/features/reader/reader_screen.dart`; виджет-тесты RC-01/RC-13 (нет key `readerBookChapterTitle_*` / аналог).

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| FRB API break / dual init | Явный path API или расширенный контракт; bytes для тестов; codegen + skills update |
| Disk space ~200MB в support dir | Accept; reuse files; document |
| Copy всё ещё jankит | Chunked write + yields; измерять vs bytes+FRB encode |
| Gate всё ещё долгий | Честный loader; живая анимация важнее короткого dead screen |
| Web регрессия | Не трогать worker без нужды; smoke `:5555`+COOP/COEP |
| Drift migration | Nullable column; old anchors degrade gracefully |
| Title was only chapter cue | Pager `n / N` остаётся; content may include heading paragraphs |

## Migration Plan

- Существующий gate/ensure оставить; заменить native `_loadAndInit` bytes-path на extract→path.
- Старые installs: первая сессия после обновления — extract; далее reuse.
- Rollback: временно вернуть bytes init (хуже UI).
- Reader zero-sync / bottom — уже в коде.
- **D14:** Drift migrate + Reader seed; старые EPUB якоря без paragraph → paragraph 0.
- **D15:** UI-only; без data migration.

## Open Questions

Нет — gate оставлен; path init выбран; mid-chapter original = как низ (D14); title chrome снят, pager оставлен (D15).
