## Context

См. `proposal.md` — Why. Сейчас в sibling `my_translator_app`: Library + Drift documents уже импортируют `.srt`/`.vtt`/`.ass`/`.ssa`; tap показывает «Reader later». Rust — scaffold FRB (`process_text` / `add_numbers`); модуля `parse/` нет. Канон: compute в Rust, user bytes в Drift; skills `work-with-parse`, `work-with-reader`, `work-with-frb-api`, client `work-with-pages` / `work-with-frb`.

## Goals / Non-Goals

**Goals:**
- Внутренний Rust parse (SRT/VTT/ASS) + тонкий FRB export owned cues.
- Flutter: tap документа → load bytes → FRB parse → Reader (время + текст, scroll) или ошибка в Library.
- Согласованный FFI-контракт в том же change.

**Non-Goals:**
- Word tap / lemma / translate / split / paging (см. proposal Out of scope).
- Смена схемы Drift / storage layout байтов.
- Валидация при import.

## Decisions

### D1 — Парсер: `fasrt` 0.5
- **Выбор:** зависимость `fasrt = "0.5.0"` (pin exact) в `rust/Cargo.toml`; feature **`std` (default)** — достаточно для native + WASM probe; `alloc`-only не нужен для FRB crate.
- **Почему:** README + `src/{srt,vtt,ass}` покрывают SRT + WebVTT + ASS/SSA; zero-copy iterators → owned `String` / `Vec` на нашей границе; пользователь выбрал крейт в explore.
- **Альтернативы:** hand-rolled; другой subtitle-crate — отложены.
- **Prerequisite toolchain:** крейт `edition = "2024"`, `rust-version = "1.95"`. Host/CI для app должны иметь rustc ≥ 1.95 (проверено: 1.99 собирает `fasrt` на `x86_64` и `wasm32-unknown-unknown`).
- **Spike (зафиксировано, не гадать на apply):**

| Формат | Entrypoint | Lossy / recovery |
|--------|------------|------------------|
| SRT | `fasrt::srt::Parser::lossy(&str)` → `Item = Result<Entry, ParseSrtError>` | явный `Options::lossy` / `Parser::lossy` |
| WebVTT | `fasrt::vtt::Parser::new(&str)` → `Item = Result<Block, ParseVttError>` | отдельного `Options::lossy` нет; встроенный recovery (`pending_line` / skip malformed timing) |
| ASS/SSA | `fasrt::ass::Parser::lossy(&str)` → `Item = Result<Block, ParseAssError>` | явный `Parser::lossy`; брать `Block::Event` с `EventKind::Dialogue` (не Comment) |

- **Map → Cue:** `start`/`end` через `Timestamp::to_duration()` → `as_millis() as u64` (ASS: centiseconds ×10 уже в `to_duration`). Текст: SRT/VTT body → split visual lines; ASS → `Event::plain_text().normalize()` (strip `{…}` / `\N`).
- **Format detect:** наш `parse/` (hint расширения + sniff `WEBVTT` / `[Script Info]`/`[V4+ Styles]`/`Dialogue:` / SRT-like); fasrt единого auto-detect не даёт. Неизвестный → `UnsupportedFormat`.
- При невозможности собрать WASM/native с pinned `fasrt` — эскалировать (не молча менять крейт).

### D2 — Слои Rust: `parse/` + thin `api/parse`
```
rust/src/parse/          # detect / decode UTF-8 / map fasrt → Cue / ParseError
rust/src/api/parse.rs    # FRB: Vec<u8> (+ optional hint) → ParsedSubtitles / error kind
```
Не раздувать scaffold `api/simple.rs`. После смены API — `flutter_rust_bridge_codegen generate`.

**Целевой FRB surface (иллюстративно, имена можно уточнить при codegen):**
```rust
// rust/src/api/parse.rs
pub struct CueDto {
    pub index: u32,
    pub start_ms: u64,
    pub end_ms: u64,
    pub text_lines: Vec<String>,
}
pub struct ParsedSubtitles {
    pub format: String, // "srt" | "vtt" | "ass"
    pub cues: Vec<CueDto>,
}
pub enum ParseErrorKind { UnsupportedFormat, Encoding, Empty, Malformed }
pub fn parse_subtitles(bytes: Vec<u8>, format_hint: Option<String>)
    -> Result<ParsedSubtitles, ParseErrorKind>; // или Result + message — см. server-work-with-errors
```
Внутренний `parse/` владеет политикой empty/lossy; `api/` только map DTO.

### D3 — Модель cue (FFI-flat)
| Поле | Назначение |
|------|------------|
| `index` | 0-based порядок |
| `start_ms` / `end_ms` | u64 ms |
| `text_lines` | `Vec<String>` visual lines |

Опционально на границе: `format` (`srt` \| `vtt` \| `ass`). Identifier/header WebVTT — не обязательны для Reader v1.

### D4 — Политики ошибок
| Случай | Исход | Kind |
|--------|--------|------|
| Lossy дефект блока | пропуск / recovery, если ≥1 cue | (успех) |
| Неизвестный формат | hard fail | `UnsupportedFormat` |
| Non-UTF-8 | hard fail (encoding) | `Encoding` |
| 0 cues (в т.ч. пустой файл / только заголовок) | hard fail | `Empty` |
| Прочие hard failures fasrt после detect | hard fail | `Malformed` |

Типизированный `ParseError` внутри `parse/`; на FRB — kind (+ опционально message) для UI. Product policy этого change: **empty = error** (строже, чем «optional Empty» в skill `work-with-parse`).

### D5 — Flutter open flow
Точка врезки: `lib/features/upload/library_screen.dart` (`_onDocumentTap`).
1. Loading на время load+parse: нестирающий list overlay / modal progress (не полный wipe папки как при `_reload`); не открывать Reader, пока parse не успешен.
2. Load document bytes через **`DocumentRepository` helper** (предпочтительно), внутри — `DocumentContentStore.load` / Web-ветка: native — sandbox по `storageKey`; Web — `utf8.encode(doc.content!)`. Сейчас у store нет `load` — **добавить** thin read (+ `MemoryDocumentContentStore`) в том же change; schema Drift не менять. `LibraryScreen` уже имеет `documentRepository` — отдельный inject store в экран не обязателен.
3. Generated FRB `parse_subtitles(bytes, format_hint)` с `format_hint: doc.format` (`srt` \| `vtt` \| `ass`).
4. Success → `Navigator.of(context).push` Reader (`lib/features/reader/`) с уже разобранными cues (+ title). Push идёт поверх `AppShell` (bottom nav скрыт на маршруте Reader) — так и задумано; back → `pop` возвращает в ту же папку Library.
5. Fail (load или parse) → существующий `_showError` / `SnackBar(key: libraryErrorSnack)` со стабильным user-facing текстом по kind (см. таблицу ниже); **не** push Reader.

**Map `ParseErrorKind` → SnackBar (EN, в стиле Library):**

| Kind / причина | Copy |
|----------------|------|
| `UnsupportedFormat` | `Unsupported or unrecognized subtitle format` |
| `Encoding` | `File encoding is not valid UTF-8` |
| `Empty` | `No subtitle cues found in this file` |
| `Malformed` | `Could not parse this subtitle file` |
| Load / IO / missing bytes | `Could not load document` |

**Reader v1 UI (без tokenize/highlight — out of scope):**
- `ListView` (или эквивалент): каждая cue — время **start→end** + текст (`text_lines` через `\n` / column).
- Формат времени из `start_ms` / `end_ms`: **`HH:MM:SS,mmm`** (zero-pad; как привычный SRT-стиль; `,` как разделитель мс). Одна функция форматирования на оба края.
- AppBar title = `document.title`; leading back → `Navigator.pop`.
- Не строить token spans / word card в этом change: plain text из FRB `text_lines`.

### D6 — Encoding / input
Flutter передаёт **`Vec<u8>`** (предпочтительно); **strict UTF-8 decode + parse** в Rust (`str::from_utf8`, не `from_utf8_lossy`). UTF-8 + BOM strip; CRLF normalize в parse pipeline (или полагаться на Lines fasrt — оба LF/CRLF).

**Caveat Web import:** текущий `DocumentContentStore` Web/`Memory…storeAsContent` пишет `utf8.decode(..., allowMalformed: true)` в Drift `content`. Для PS-05 на Web open: либо (a) при open считать уже «декодированный» текст и не эмулировать raw invalid bytes, либо (b) при import отклонять non-UTF-8 / хранить lossless — **не расширять scope на validate-on-import**; в тестах PS-05 гонять Rust unit на сырых байтах. Reader open на Web передаёт `utf8.encode(content)`.

### D7 — Тесты
- Rust: fixtures SRT/VTT/ASS (+ bad/empty/encoding) рядом с `parse/`; `cargo test` ↔ SC PS-*.
- Flutter: widget tests open success / fail / timestamps / back ↔ SC RC-*, LD-10/11; `flutter analyze`.

Чеклист реализации — `tasks.md`.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Молодой `fasrt`, API drift | Pin `0.5.0`; зафиксированные entrypoints выше; unit fixtures |
| MSRV ≥ 1.95 / edition 2024 dep | Проверить rustc в CI/агентах до apply; не понижать без эскалации |
| WASM | `cargo check --target wasm32-unknown-unknown` с fasrt уже проходит на probe; повторить в app crate после добавления dep |
| VTT без `Parser::lossy` | Использовать встроенный recovery; PS-07 fixture на VTT — skip broken timing + соседняя валидная cue |
| Большие файлы → полный `Vec` на bridge | Accept для v1; paging out of scope |
| ASS override codes в тексте | `plain_text().normalize()` в map-слое |
| Web `allowMalformed` при store | PS-05 доказывать в Rust unit; open передаёт bytes/encode без повторного lossy decode в Dart |
| Импортированные битые файлы остаются в Library | UX: ошибка на open; validate-on-import later |
| Нет `contentStore.load` сегодня | Добавить read path в Flutter tasks (3.x); schema Drift не трогать |

## Migration Plan

Greenfield parse+Reader поверх существующей Library. Нет миграции Drift schema. Rollback = revert change; документы в Library сохраняются.

## Technical prerequisites (из explore)

- Принят `fasrt` 0.5; SRT+VTT+ASS в одном change; Reader с таймкодами; lossy; fail/empty → не открывать Reader.
- Entrypoints зафиксированы в D1 (повторный «слепой» spike не нужен — только sanity `cargo check` после pin в app `Cargo.toml`).
