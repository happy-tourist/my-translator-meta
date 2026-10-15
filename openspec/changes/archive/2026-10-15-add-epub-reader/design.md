## Context

См. `proposal.md` (Why / Scope). Базовый EPUB path (parse / Library / book UI / split) уже реализован в sibling app. На Web скролл абзацев и split jank: `CueRow` строит `WidgetSpan`+`GestureDetector` на каждый токен; batch tokenize/mark на всю главу; divider `setState` пересобирает обе панели. Follow-up — post-load Flutter perf без смены `parse_epub`.

## Goals / Non-Goals

**Goals:**

- Синхронный EPUB parse в Rust (thin FRB), вход **bytes**, выход главы → plain абзацы.
- Book-режим в существующем Reader: абзацы + нижний chapter pager; split EPUB↔EPUB sync по индексу главы.
- Import `.epub` в Drift без кэша глав (re-parse on open).
- **Perf follow-up:** book-mode lightweight hit-test; progressive tokenize + chapter token cache; split divider без per-drag rebuild списков.

**Non-Goals:**

- Rich HTML, картинки, сноски; запись EPUB; video для EPUB; смешанный EPUB↔SRT split.
- Отдельный top-level screen вне `features/reader`.
- Lazy parse тел глав / смена FRB `parse_epub` payload; переписывание subtitle WidgetSpan CueRow (кроме optional shared helper).

## Decisions

### D1 — Парсер: `rbook` (read-only)

- **Выбор:** `rbook` `0.7` (crates.io latest 0.7.x) с `default-features = false`, features `threadsafe` + `prelude` (без `write` для размера WASM).
- **Почему:** зрелее lexepub; docs заявляют `wasm32-unknown-unknown`; sync API совместим с текущим sync `parse_subtitles`; `Epub::read(impl Read + Seek)` / `Cursor<Vec<u8>>` = bytes-in (без filesystem path).
- **Альтернативы:** lexepub (async/Tokio, моложе) — отвергнут в explore; писать свой ZIP/XHTML — out of scope.

### D2 — FRB surface (locked names)

- **Модуль:** внутренний `rust/src/parse/epub/` (domain) + thin `rust/src/api/epub.rs`; в `lib.rs` — `mod parse` уже есть (добавить `parse::epub`); `api/mod.rs` — `pub mod epub;`. Не класть логику в `api/simple.rs`.
- **Контракт (sync, bytes-only, без path / Tokio):**

```rust
pub struct EpubChapterDto {
    pub index: u32,              // 0-based document order
    pub title: String,           // TOC label or stable fallback
    pub paragraphs: Vec<String>, // non-empty plain paragraphs
}

pub struct ParsedEpub {
    pub chapters: Vec<EpubChapterDto>,
}

pub enum EpubParseErrorKind {
    UnsupportedFormat, // not a readable EPUB / wrong container
    Malformed,         // EPUB opened but structure/content unusable mid-parse
    Empty,             // no chapter with ≥1 non-empty paragraph
}

pub fn parse_epub(bytes: Vec<u8>) -> Result<ParsedEpub, EpubParseErrorKind>;
```

- **Аналог:** тот же паттерн, что `api/parse.rs` → `parse_subtitles` / `ParseErrorKind` (отдельный enum — **не** переиспользовать `ParseErrorKind` субтитров).
- **Title fallback:** если у TOC-пункта пустой label — `"Chapter {n}"` где `n = index + 1` (стабильно, EN, без i18n в Rust).
- После API — `flutter_rust_bridge_codegen generate` (+ web pkg при необходимости).
- **Learning / tokenize:** нового FRB tokenize API не нужно — Flutter передаёт абзацы как `Vec<String>` в существующий `tokenize_cues` (как display-string cues).

### D3 — Главы из TOC

- Обход TOC (`Epub::toc().contents()` + дети / nested): каждый пункт с **разрешаемым ресурсом** (`TocEntry::resource()` = Some) и непустым plain-текстом после strip = глава (вложенные включены).
- Структурный узел без resource / без текста → не пустая страница; дети с контентом — да.
- Нет usable TOC (нет contents / ни одного content-bearing пункта) → ровно одна глава: весь извлечённый текст по spine/`reader()` order; title fallback `"Chapter 1"`.
- Тело главы: `Epub::read_resource_str(resource)` (или эквивалент) для href пункта TOC; **fragment (`#…`) в v1 игнорируется** — берётся весь XHTML ресурса. Если несколько TOC-пунктов указывают на один файл (разные fragment) — в v1 возможны главы с одинаковым телом (известное ограничение; fragment-scoped extract — out of scope).
- Strip markup → абзацы: см. D9.

### D9 — Plain-text strip (wasm-safe)

- **Без** C/native HTML-парсеров и без отдельного JS strip на Web.
- Допустимо: простой tag-strip + разбиение по block-тегам (`p`, `div`, `h1`–`h6`, `li`, `blockquote`, `br` → граница/перевод строки); схлопнуть whitespace; отбросить пустые сегменты; изображения/сноски опускать (не обязаны попадать в абзацы).
- Не тянуть crates с C-deps ради HTML; при сомнении — минимальный hand-rolled strip в `parse/epub/`.

### D4 — Flutter book-режим (locked session)

- Точки: `lib/features/upload/library_screen.dart` (format allowlist + open branch), `lib/data/library/document_format.dart` (+ repo reject messages), `lib/data/library/parse_error_messages.dart` (+ EPUB kinds), `lib/features/reader/reader_screen.dart` и соседние виджеты chapter pager / paragraph list.
- **Не** создавать `lib/screens/reader_screen.dart`.
- **Session contract (locked):** Library open для `format == epub` вызывает FRB `parse_epub` и пушит `ReaderScreen` в **book-режиме** с payload глав (`ParsedEpub` / эквивалент: список глав → абзацы), плюс `documentFormat: 'epub'` (или явный `ReaderMode.book`). Subtitle open остаётся `parse_subtitles` → `originalCues`. Не открывать EPUB через пустой `CueDto` list.
- **Paragraph → learning pipeline (locked minimal reuse):** тексты абзацев текущей главы → `tokenize_cues(texts: …)` / `mark_vocab_tokens` как сейчас для cue display-strings. UI-строка абзаца: `showTimestamp: false`; hit-test — **D11** (не WidgetSpan-на-слово). Допустим adapter `CueDto` только как внутренний display/tokenize носитель; **запрещено** использовать `startMs` абзацев для split-sync или reading-anchor book-пути. Progressive tokenize / cache — **D12**.
- Chrome pager: паттерн как у `VideoFramePages` (`[<] n/N [>]` снизу списка абзацев), единица = глава; при смене главы — заменить абзацы и сбросить scroll к началу.
- EPUB: **скрыть** AppBar Load/Replace/Remove video (`readerLoadVideo` / `readerRemoveVideo`); не вызывать video restore на open.

### D5 — Split

- Ветка по `format` / book-режиму: subtitle → текущий time sync; epub → **общее** состояние текущего chapter index (0-based); pager любой book-панели двигает индекс → обе панели показывают ту же главу (если индекс есть в обеих).
- Каждая book-панель (верх/низ при loaded translation) имеет chapter pager chrome; scroll абзацев внутри главы **не** обязан зеркалироваться.
- Picker / `attachTranslation` для EPUB-оригинала: только `.epub` (LD-26); subtitle allowlist для subtitle-оригинала не расширять смешанно.
- Load translation EPUB: `parse_epub`, не `parse_subtitles`; ошибки → `EpubParseErrorKind` UI copy (D10).
- Предпосылка продукта: одинаковое N глав; при расхождении на load — best-effort clamp индекса + заметная ошибка (RS-16; не ломая RS-14/15 для равных N).

### D6 — Persist / якорь (binary-safe + chapter)

- Format `epub` в allowlist (`document_format` / import / attach).
- **Native:** sandbox `storageKey` + `writeAsBytes` / `readAsBytes` — уже binary-safe; без изменений контракта.
- **Web (hard lock):** текущий `DocumentContentStore` Web (`utf8.decode` → `Documents.content` Text → `utf8.encode` в `loadBytes`) **коррумпирует ZIP/EPUB**. EPUB MUST NOT идти этим UTF-8 roundtrip.
  - **Locked v1:** для `format == epub` на Web хранить байты как **base64** в `Documents.content`; `DocumentRepository.loadBytes` / store path format-gated: epub → base64 decode/encode; subtitle → прежний UTF-8.
  - **Допустимая альтернатива** (если base64 отвергнут по размеру при apply): companion `BlobColumn` таблица по образцу `DocumentVideoMedia` (+ schema bump) — тот же bytes-in/out контракт для open/parse.
- Reading anchor **v1 (locked):** для epub в существующих колонках format-gated семантика — `lastReadStartMs` = **0-based chapter index** (не cue time); `lastReadAlignment` = optional viewport alignment абзацного списка **или** null. **Paragraph index resume — out of scope v1** (RC-06 = глава; paragraph MAY later). Subtitle: `lastReadStartMs` = cue `startMs` как сейчас. Save/restore ветвить по `documentFormat` / book-режиму; не вызывать `nearestCueIndex` для epub.
- Re-parse на каждый open (без таблицы кэша глав в v1).

### D7 — Web

- Тот же FRB wasm path, что субтитры; COOP/COEP уже канон `work-with-env-deploy` — не отдельный EPUB-этап.
- Не подключать отдельный JS/wasm-pack модуль rbook.
- Persist EPUB на Web — D6 (base64 / Blob), не «как текст субтитров».
- **rbook** заявляет `wasm32-unknown-unknown`; при early `cargo check --target wasm32-unknown-unknown` (task 1.6) возможны транзитивные сюрпризы (`zip` → `getrandom` без backend на `wasm32-unknown-unknown`). Митигация при сбое линковки (в порядке предпочтения): (1) target-specific `getrandom` feature/`rustflags` backend, совместимый с уже используемым FRB web build; (2) если RNG в runtime parse не нужен — `unsupported` backend только если доказано, что encrypt-пути zip не вызываются на наших fixtures; (3) не откатываться на отдельный JS EPUB-парсер. Spike зафиксировать в apply notes при первом зелёном wasm check.

### D10 — EPUB parse errors → UI copy

- Добавить mapping `EpubParseErrorKind` → пользовательские строки (рядом с `messageForParseError` / `parse_error_messages.dart`): минимум различать «неподдерживаемый/битый» (`UnsupportedFormat` \| `Malformed`) vs «пустой текст» (`Empty`) — PE-05 / LD-11.
- Library open и Reader load-translation ловят `EpubParseErrorKind` отдельно от `ParseErrorKind` субтитров.

### D8 — Prerequisites из explore (закрыты решением)

| ID | Решение |
|----|---------|
| Парсер | rbook |
| Learning | как субтитры |
| Страница | глава (TOC / nested) |
| Нет TOC | одна глава |
| Split | EPUB↔EPUB, sync по главе |
| Медиа | без картинок/сносок |
| Persist | bytes + re-parse; Web binary-safe (D6 base64) |

### D11 — Book-mode lightweight hit-test (locked)

- **Проблема:** `_SelectableCueText` строит `WidgetSpan` + `GestureDetector` + `Text` на каждый token — на Web EPUB-главе это тысячи виджетов → jank скролла/split.
- **Выбор:** в **book-режиме** paragraph row использует `Text.rich` из `TextSpan` (стили selection / flash / `isInVocab`) + **один** gesture layer на абзац; tap → map local offset → char index (`cueCharSlice` / TextPainter) → `tokenIndex` → существующий `_onOriginalTokenTap`.
- **Subtitle:** оставить текущий WidgetSpan CueRow path без изменений (короткие cues).
- **Тесты:** Keys `readerCueToken_*` для book заменить/дополнить Semantics или finder по `cueListIndex`+`tokenIndex`; RC-04/RC-07 обязаны зелёными.
- **Альтернативы:** (B) только chunk/cache без смены hit-test — отвергнуто (недостаточно для Web scroll); (C) dual path уже выбран (book vs subtitle).

### D12 — Progressive tokenize + chapter cache (locked)

- Показывать plain paragraphs сразу; `tokenize_cues` чанками (например первые N абзацев / idle callback на остаток); `mark_vocab_tokens` MAY после tokenize чанка или вторым проходом.
- Highlight MAY отставать от tap-ready (продуктово ок).
- In-memory `Map<chapterIndex, List<CueTokensDto>>` (и mark-ready flag) на время Reader session; invalidate generation при смене главы in-flight уже есть; при Add/edit/delete vocab — re-mark затронутых кешей / текущей главы как сейчас.
- Не персистить token cache в Drift.

### D13 — Book split divider without per-drag list rebuild (locked)

- Divider ratio через `ValueNotifier` / аналог: менять только высоты `SizedBox`, не вызывать полный `setState` на `ReaderScreen`, пересоздающий оба paragraph `ListView` на каждый pointer-move.
- Включение split по-прежнему строит две панели один раз; последующий drag — дешёвый.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| `rbook` + `zip` раздувают WASM / getrandom | `default-features = false`; early wasm check (D7); без JS EPUB fallback |
| TOC vs spine / fragment href | D3: fragment ignore v1; fixtures nested TOC + same-file fragments |
| Длинные главы → тяжёлый tokenize | D12 progressive + chapter cache; не весь book tokenize |
| Неравное N глав у «готовых» переводов | продукт предполагает равенство; clamp + ошибка на load |
| Map абзацев на cue pipeline | D4 + D11 lightweight hit-test; без time-sync |
| Web UTF-8 `Documents.content` ломает EPUB ZIP | D6: base64 (locked) или Blob companion |
| Hit-test miss на wrap / Unicode | переиспользовать char-span модель + widget tests RC-04/07 |
| Тесты зависят от `readerCueToken_*` Widget Keys | обновить book finders; subtitle Keys не ломать |

## Migration Plan

- Новый format `epub` в allowlist; schema bump **не** обязателен для якоря главы (reuse `lastReadStartMs` format-gated) и для Web persist (base64 в `content`).
- Schema bump только если выбран Blob-companion вместо base64.
- Rollback: feature за format gate; subtitle flows не ломать.
- Perf follow-up: только Flutter book path; откат = вернуть CueRow tokens path для book (хуже perf, поведение RC-04 сохраняется).

## Open Questions

- Нет открытых. Paragraph resume — out of scope v1 (D6). FRB DTO/error names — D2. Web binary persist — D6. Book session / adapter — D4. Hit-test / progressive tokenize / divider — D11–D13.
