## Context

См. `proposal.md` — Why. В sibling `my_translator_app` реализовано: `ReaderScreen` (Stateful) с split toggle, Drift `Documents.translationDocumentId`, sync time-first, preserve scroll на Show/Hide (D7), reading anchor persist/restore (D9–D11). Skills: `work-with-reader` / `split-scroll.md`, `work-with-drift`, `work-with-pages` (reader.md), `work-with-frb`.

## Goals / Non-Goals

**Goals:**
- Flutter split UI (toggle, resize, empty/load/delete) на Reader.
- Sync: nearest `start_ms` → pixel settle-only (чисто Flutter helper на уже распарсенных `CueDto`).
- Drift: скрытый translation document + link; list filter; cascade; restore on open.
- Preserve scroll оригинала на **Show и Hide** split (session remount).
- Drift: persist якоря чтения оригинала; restore при повторном open.
- Повторное использование существующего FRB `parse_subtitles` / Dart `parseSubtitles` (без нового Rust API, без правок `rust/`).

**Non-Goals:**
- Word tap / highlight / offline translate card.
- Любой новый Rust/FRB surface: pairing / align / parallel-lists API; index-only sync как единственный режим.
- Persist режима split on/off и divider ratio между сессиями.
- UI «заменить файл» без delete.
- Задачи `cargo check` / `cargo test` / FRB codegen для этого change (codegen не нужен — API не меняется).

## Rust / FRB boundary (reuse only)

Этот change **не требует** правок `my_translator_app/rust/` и **не добавляет** capability delta на `parse/subtitles`.

| Claim | Evidence in app HEAD | Классификация |
|-------|----------------------|---------------|
| Парс перевода теми же форматами SRT/VTT/ASS | [`rust/src/api/parse.rs`](../../../my_translator_app/rust/src/api/parse.rs) `parse_subtitles(bytes, format_hint) → Result<ParsedSubtitles, ParseErrorKind>`; domain `rust/src/parse/` | **aligns** |
| Таймкоды для sync / resume | Dart FRB `CueDto { index, startMs, endMs, textLines }` где `startMs`/`endMs` — **`BigInt`**; nearest по `startMs` | **aligns** |
| Ошибки load/parse (в т.ч. zero cues) | `ParseErrorKind::{UnsupportedFormat, Encoding, Empty, Malformed}` — map в UI как Library | **aligns** |
| Persist original↔translation + reading anchor | Drift во Flutter (`lib/data/`), не Rust SQLite | **FE-only** (ожидаемо) |
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
- **DI (как Library):** передать в Reader `DocumentRepository` (обязательно для load/persist/delete перевода **и** reading anchor) и optional `ParseSubtitlesFn` для тестов (default → generated `parseSubtitles`). Не вызывать FRB/Drift из «голых» leaf-виджетов без inject.

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
  -> load reading anchor (lastReadStartMs / lastReadAlignment) if present
  -> push Reader(originalCues, translationCues?, initialAnchor?)
Reader after first layout
  -> if initialAnchor: nearestCueIndex + jumpTo(+alignment); if translation cues: syncPanes
Reader split ON, no translation
  -> empty + file_picker
    -> bytes + formatHint from picked name (как import)
    -> parseSubtitles (тот же FRB, без нового API)
    -> on success: DocumentRepository.attachTranslation(folder of original; store format like import)
    -> set translationCues; syncOnce(time->pixel)
    -> on fail: SnackBar / inline error (map ParseErrorKind как Library; Empty = zero cues)
Delete bottom
  -> detachAndDeleteTranslation; clear cues; empty state
Scroll settle (original) / Reader dispose|pop
  -> DocumentRepository.saveReadingAnchor(originalId, startMs, alignment?)
```

**Reader constructor / args:** `title`, `originalCues`, `originalDocumentId`, optional `translationCues` / `translationDocumentId`, optional `initialReadingAnchor` (`startMs` + `alignment`), `documentRepository`, optional `parseSubtitles`. Точка push: `library_screen.dart` `_onDocumentTap`. Парс всегда через существующий generated Dart API — **не** трогать `rust/src/api/`.

### D4 — Sync scroll: time then pixel settle
- **Выбор:** два независимых списка с programmatic scroll-to-index (предпочтительно `scrollable_positioned_list` **или** эквивалент на `ScrollablePositionedList` / `ItemScrollController`). На `ScrollEndNotification` / после programmatic jump:
  1. `anchorStartMs` (`BigInt`) из видимой/якорной cue панели-источника;
  2. `nearestIndex` в другой панели по абсолютной разнице `|startMs - anchor|` (сравнение `BigInt`, без потери точности через `toInt()`);
  3. `scrollTo(index)`;
  4. опциональная pixel fine-adjust после layout settle.
- **Не использовать** `SyncScrollControllerGroup` как единственный sync (pixel-mirror ломает разную высоту строк / разную длину списков).
- **Guard:** флаг `_syncing`, чтобы не зациклить listener A↔B.
- **После load / включения split с уже загруженным переводом / restore resume:** один `syncOnce` от верхней панели.

**Helper:** `int nearestCueIndex(List<CueDto> cues, BigInt startMs)` (пусто → не sync / no-op). Unit-тест на ties: фиксировать детерминированный выбор (например меньший index при равной дистанции).

**Зависимость:** добавить в `pubspec.yaml` только пакет, нужный для scroll-to-index (например `scrollable_positioned_list`); не тянуть `sync_scroll_controller`, если он не нужен для settle pixel step.

### D5 — Ошибки и UX copy
- Переиспользовать **тот же** map `ParseErrorKind` → EN SnackBar / заметная ошибка, что Library open-flow (`UnsupportedFormat` / `Encoding` / `Empty` / `Malformed` / load fail); ключ/аналог для Reader bottom (например `readerTranslationErrorSnack`). Не вводить отдельную семантику ошибок для перевода.
- Fail перевода при open оригинала: Reader всё равно открывается; split empty или ошибка при попытке показать низ — не блокировать оригинал.
- Пока в нижней панели уже есть cues перевода, UI показывает **удаление**, не кнопку «заменить»; повторная загрузка — только после delete (detach+remove).

### D6 — Тесты
- Drift unit: attach / list hides translation / delete translation / cascade; **save/load reading anchor (LD-17)**.
- Widget: RS-01…RS-12 по возможности с fake parse/repos (как текущий `reader_screen_test` / library inject `parseSubtitles` + `DocumentRepository`).
- Unit helper: `nearestCueIndex(cues, startMs: BigInt)`.

### D7 — Preserve original scroll on Show **and Hide** split (RS-10, RS-11)
- **Проблема:** `body: split ? splitBody : singlePane` снимает `ScrollablePositionedList` оригинала с дерева → позиция сбрасывается в начало при любом направлении toggle.
- **Выбор:** перед **обоими** переходами (`false→true` и `true→false`) сохранить якорь верхней/единственной панели оригинала (`ItemPositionsListener` → top-visible `index` + `itemLeadingEdge`); после `setState` / post-frame — `jumpTo` на `_originalScrollController` (когда attached). После Show с translation cues — `_syncPanes(fromOriginal: true)`.
- **Почему не A (один виджет всегда в дереве):** больше рефакторинга layout; capture+jumpTo достаточно.
- **Связь с resume:** session capture/jumpTo — in-memory; persist якоря — D9/D10 (отдельно).

### D8 — Delete chrome в верхней зоне нижней панели (RS-07)
- **Выбор:** убрать `IconButton` Delete из `AppBar.actions`; в режиме split при `_hasTranslationCues || _hasLinkedTranslation` показать полоску/header **сверху нижней панели** (над cue list или над empty+Load) с действием Delete (`readerTranslationDelete` / empty-variant key).
- **Toggle split** остаётся в AppBar.
- Stuck-link (link без cues): Delete в той же верхней зоне нижней панели; Load — в empty content ниже header (или только Delete в header, пока link не снят).

### D9 — Drift: reading anchor на оригинале (LD-17)
- **Выбор:** на `Documents` nullable поля якоря чтения **оригинала**:
  - `lastReadStartMs` — `IntColumn` nullable (миллисекунды; subtitle timestamps укладываются в i64; в API/Reader сравнивать с `CueDto.startMs` как `BigInt`);
  - `lastReadAlignment` — `RealColumn` nullable (`0.0…1.0`, `itemLeadingEdge`).
- Bump `schemaVersion` + migration (отдельно от уже применённого bump translation FK, если schema уже выше — ещё один bump).
- **Не** хранить якорь на hidden translation-doc; **не** persist split/divider.
- `DocumentRepository`: `saveReadingAnchor(documentId, startMs, alignment?)`, чтение якоря при `get`/open path (или отдельный `loadReadingAnchor`).

### D10 — Когда писать и как восстанавливать якорь (RS-12)
- **Write:** (1) после user-scroll settle на панели оригинала; (2) при dispose / Route pop Reader — best-effort save текущего top-visible якоря (если attached).
- **Не** писать на каждый pixel tick / programmatic sync с `_syncing == true` (избегать шума и лишних write от sync).
- **Restore:** Library/open передаёт якорь в Reader; после первого layout — `nearestCueIndex` + `jumpTo(index, alignment)`; если есть translation cues — затем `_syncPanes`; пустые cues / null якорь → начало списка.
- Fail save якоря MUST NOT блокировать UI (best-effort; silent или log).

### D11 — Explore decisions (зафиксировано)
| ID | Решение |
|----|---------|
| Resume при reopen | **да** |
| Change | дописать **add-reader-split** (не новый каталог) |
| Хранить | `startMs` + alignment оригинала |
| Когда писать | settle + pop |
| Restore | только оригинал; низ — time sync; split/divider не persist |
| Cues сдвинулись | nearest по `startMs` |
| Hide split | preserve как Show (RS-11) |

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Time sync «прыгает» при плохом align файлов | nearest `start_ms` зафиксирован в spec; позже можно порог — out of scope |
| Feedback loop sync A↔B | `_syncing` + sync primarily on settle |
| Migration Drift на Web/native | schema bump(s); тесты репозитория |
| Skills канон index vs time-first | MVP time-first зафиксирован здесь + в `split-scroll.md`; target index+pairing — later |
| Remount list сбрасывает scroll при Show/Hide | D7: capture + post-frame `jumpTo` оба направления |
| Delete в AppBar неочевиден для нижней панели | D8: chrome только в bottom pane top |
| Частые Drift write на settle | throttle optional later; MVP: settle + pop; skip while `_syncing` |
| Flash позиции 0 до post-frame jumpTo | допустим краткий; не stuck wrong state |

## Migration Plan

1. Bump Drift schema: nullable `translationDocumentId` (+ indices/FK) — уже в MVP path.
2. Bump Drift schema: nullable `lastReadStartMs` + `lastReadAlignment`.
3. Deploy: существующие документы без link/якоря — Reader с начала списка, как сейчас.
4. Rollback: откат app+meta; orphan risk минимален.

## Open Questions

Нет блокирующих. Defaults: divider `0.5`; AppBar только toggle; Delete — D8; resume — D9–D11.
