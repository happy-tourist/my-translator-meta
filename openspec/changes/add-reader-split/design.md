## Context

См. `proposal.md` — Why. Сейчас в sibling `my_translator_app`: single-pane `ReaderScreen` (`lib/features/reader/`) принимает `title` + `List<CueDto>`; Library open (`library_screen.dart`) → `DocumentRepository.loadBytes` → FRB `parseSubtitles` → push Reader. Drift `Documents` без связи original↔translation; список папки показывает все документы папки. Skills: `work-with-reader` / `split-scroll.md` (**MVP = time-first** на Flutter; index+pairing = target later), `work-with-drift`, `work-with-pages` (reader.md), `work-with-frb`.

## Goals / Non-Goals

**Goals:**
- Flutter split UI (toggle, resize, empty/load/delete) на Reader.
- Sync: nearest `start_ms` → pixel settle-only (чисто Flutter helper на уже распарсенных `CueDto`).
- Drift: скрытый translation document + link; list filter; cascade; restore on open.
- Повторное использование существующего FRB `parse_subtitles` / Dart `parseSubtitles` (без нового Rust API, без правок `rust/`).

**Non-Goals:**
- Word tap / highlight / offline translate card.
- Любой новый Rust/FRB surface: pairing / align / parallel-lists API; index-only sync как единственный режим.
- Persist divider ratio / scroll position across app restarts.
- UI «заменить файл» без delete.
- Задачи `cargo check` / `cargo test` / FRB codegen для этого change (codegen не нужен — API не меняется).

## Rust / FRB boundary (reuse only)

Этот change **не требует** правок `my_translator_app/rust/` и **не добавляет** capability delta на `parse/subtitles`.

| Claim | Evidence in app HEAD | Классификация |
|-------|----------------------|---------------|
| Парс перевода теми же форматами SRT/VTT/ASS | [`rust/src/api/parse.rs`](../../../my_translator_app/rust/src/api/parse.rs) `parse_subtitles(bytes, format_hint) → Result<ParsedSubtitles, ParseErrorKind>`; domain `rust/src/parse/` | **aligns** |
| Таймкоды для sync | Dart FRB `CueDto { index, startMs, endMs, textLines }` где `startMs`/`endMs` — **`BigInt`** (не `int`); nearest по `startMs` | **aligns** |
| Ошибки load/parse (в т.ч. zero cues) | `ParseErrorKind::{UnsupportedFormat, Encoding, Empty, Malformed}` — map в UI как Library | **aligns** |
| Persist original↔translation | Drift во Flutter (`lib/data/`), не Rust SQLite | **FE-only** (ожидаемо) |
| Sync scroll / divider / split chrome | Flutter controllers + helper; **не** Rust | **FE-only** (ожидаемо) |

**Контракт вызова (как Library open):** `parseSubtitles(bytes:, formatHint:)` где `formatHint` — расширение/`doc.format` (`srt` \| `vtt` \| `ass`) при наличии; для file_picker перевода — hint из имени файла по тем же правилам import/open. Успех → `ParsedSubtitles.cues`; fail → catch `ParseErrorKind` (включая `Empty` для нуля cues — RS-08).

**Явно не делать в Rust:** pairing original↔translation по индексу, выравнивание списков, placeholder-cues, отдельный sync API. Канон skills `work-with-reader` / `split-scroll.md` / `work-with-pages` (`reader.md`): **MVP = time-first на Flutter**; index + Rust pairing остаются **target (later)**.

## Decisions

### D1 — Split как режим Reader, не отдельный route
- **Выбор:** расширить сессию Reader (`StatelessWidget` → **StatefulWidget**): `splitEnabled`, `dividerRatio` (default `0.5`), `translationCues` / `translationDocumentId` optional.
- **Почему:** back из Library уже один push; skills допускают mode под `lib/features/reader/` (или `features/split/` widgets рядом).
- **Альтернатива:** отдельный `SplitReaderScreen` route — лишний hop при toggle.

**Точки врезки UI:**
- `lib/features/reader/reader_screen.dart` — AppBar action toggle; body: single list **или** `Column` (top pane / drag handle / bottom pane). Cue rows: вынести общий widget (timestamp + text_lines) для обеих панелей — не дублировать разметку.
- Новые виджеты рядом: pane list (`scrollable_positioned_list` или эквивалент), empty+load, divider handle (без обязательного отдельного feature-folder, если файлов мало).
- **DI (как Library):** передать в Reader `DocumentRepository` (обязательно для load/persist/delete перевода) и optional `ParseSubtitlesFn` для тестов (default → generated `parseSubtitles`). Не вызывать FRB/Drift из «голых» leaf-виджетов без inject.

### D2 — Drift: link + скрытость
- **Выбор:** на `Documents` добавить nullable `translationDocumentId` (FK → `Documents.id`) у **оригинала**; переводный ряд — обычный document в той же `folderId`, не попадающий в list query.
- **Фильтр списка:** `watch/list by folder` исключает id, которые являются чьим-то `translationDocumentId` (или явный флаг `isHiddenTranslation` — допустим, если FK-only join неудобен; предпочтение: без лишнего флага, если query простой).
- **Почему:** один слот перевода на оригинал; import остаётся visible; reader-load создаёт hidden satellite.
- **Альтернативы:** role enum на каждом doc; отдельная таблица pairs — тяжелее для MVP.

**Repos / schema:**
- `lib/data/database/tables.dart` + migration bump `schemaVersion` в `app_database.dart`.
- `DocumentRepository`: `attachTranslation`, `detachAndDeleteTranslation`, `deleteDocument` cascade, `listVisibleByFolder`, `loadLinkedTranslation`.

**Cascade:** delete original → delete linked translation bytes/row (как folder cascade уже чистит docs — расширить на link). Delete translation from Reader → `detachAndDeleteTranslation` only.

### D3 — Open / load orchestration
```
Library tap original
  -> load+parse original (существующий flow: loadBytes → parseSubtitles)
  -> if translationDocumentId: loadBytes + parseSubtitles(formatHint: translation.format) best-effort
  -> push Reader(originalCues, translationCues?)
Reader split ON, no translation
  -> empty + file_picker
    -> bytes + formatHint from picked name (как import)
    -> parseSubtitles (тот же FRB, без нового API)
    -> on success: DocumentRepository.attachTranslation(folder of original; store format like import)
    -> set translationCues; syncOnce(time->pixel)
    -> on fail: SnackBar / inline error (map ParseErrorKind как Library; Empty = zero cues)
Delete bottom
  -> detachAndDeleteTranslation; clear cues; empty state
```

**Reader constructor / args:** `title`, `originalCues`, `originalDocumentId`, optional `translationCues` / `translationDocumentId`, `documentRepository`, optional `parseSubtitles`. Точка push: `library_screen.dart` `_onDocumentTap` (после best-effort load+parse translation). Парс всегда через существующий generated Dart API — **не** трогать `rust/src/api/`.

### D4 — Sync scroll: time then pixel settle
- **Выбор:** два независимых списка с programmatic scroll-to-index (предпочтительно `scrollable_positioned_list` **или** эквивалент на `ScrollablePositionedList` / `ItemScrollController`). На `ScrollEndNotification` / после programmatic jump:
  1. `anchorStartMs` (`BigInt`) из видимой/якорной cue панели-источника;
  2. `nearestIndex` в другой панели по абсолютной разнице `|startMs - anchor|` (сравнение `BigInt`, без потери точности через `toInt()`);
  3. `scrollTo(index)`;
  4. опциональная pixel fine-adjust после layout settle.
- **Не использовать** `SyncScrollControllerGroup` как единственный sync (pixel-mirror ломает разную высоту строк / разную длину списков).
- **Guard:** флаг `_syncing`, чтобы не зациклить listener A↔B.
- **После load / включения split с уже загруженным переводом:** один `syncOnce` от верхней панели.

**Helper:** `int nearestCueIndex(List<CueDto> cues, BigInt startMs)` (пусто → не sync / no-op). Unit-тест на ties: фиксировать детерминированный выбор (например меньший index при равной дистанции).

**Зависимость:** добавить в `pubspec.yaml` только пакет, нужный для scroll-to-index (например `scrollable_positioned_list`); не тянуть `sync_scroll_controller`, если он не нужен для settle pixel step.

### D5 — Ошибки и UX copy
- Переиспользовать **тот же** map `ParseErrorKind` → EN SnackBar / заметная ошибка, что Library open-flow (`UnsupportedFormat` / `Encoding` / `Empty` / `Malformed` / load fail); ключ/аналог для Reader bottom (например `readerTranslationErrorSnack`). Не вводить отдельную семантику ошибок для перевода.
- Fail перевода при open оригинала: Reader всё равно открывается; split empty или ошибка при попытке показать низ — не блокировать оригинал.
- Пока в нижней панели уже есть cues перевода, UI показывает **удаление**, не кнопку «заменить»; повторная загрузка — только после delete (detach+remove).

### D6 — Тесты
- Drift unit: attach / list hides translation / delete translation / cascade.
- Widget: RS-01…RS-09 по возможности с fake parse/repos (как текущий `reader_screen_test` / library inject `parseSubtitles` + `DocumentRepository`).
- Unit helper: `nearestCueIndex(cues, startMs: BigInt)`.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Time sync «прыгает» при плохом align файлов | nearest `start_ms` зафиксирован в spec; позже можно порог — out of scope |
| Feedback loop sync A↔B | `_syncing` + sync primarily on settle |
| Migration Drift на Web/native | одна schema bump; тесты репозитория |
| Skills канон index vs time-first | MVP time-first зафиксирован здесь + в `split-scroll.md`; target index+pairing — later |

## Migration Plan

1. Bump Drift schema: nullable `translationDocumentId` (+ indices/FK по conventions drift skill).
2. Deploy: существующие документы без link — поведение Reader как сейчас.
3. Rollback: откат app+meta; orphan risk минимален до первой attach.

## Open Questions

Нет блокирующих. Default: divider ratio `0.5`; title скрытого перевода — имя выбранного файла; AppBar toggle split.
