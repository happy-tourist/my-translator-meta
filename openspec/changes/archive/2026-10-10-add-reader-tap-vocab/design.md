## Context

See `proposal.md` — Why / Scope. **Implemented** in sibling app: Rust `tokenize/` + FRB `tokenize_cues` (`panko` 0.2.0); Reader original pane — batch tokenize on open, tap selection, bottom CTA + sheet → Drift `DictionaryRepository` via Shell → Library → Reader. Lemma morph / offline translate / lemma highlight remain out of scope (still absent).

Слои: **Rust** tokenize compute + thin FRB `tokenize_cues`; **Flutter** Reader original pane (paint/gestures/CTA/sheet) → Drift persist. Generated `lib/src/rust/**` не править вручную.

## Goals / Non-Goals

**Goals:**

- Ввести tokenize (`panko`) с FFI DTO на **char** spans и batch API на список текстов cues оригинала.
- Встроить tap selection + bottom CTA + bottom sheet в Reader original.
- Сохранять в Drift через существующий repository (normalize + upsert).

**Non-Goals (design-level):**

- Morph lemmatizer crate / `lemma_key` ≠ stub normalize.
- Offline translate engine; lemma highlight refresh после save.
- Persist токенов на диск; tokenize translation pane.
- Кастомный word-card chrome сверх Material bottom sheet.

## Decisions

### D1 — Tokenizer: `panko` (tokenize-only)

- **Choice:** зависимость `panko = "0.2.0"` (pin в `rust/Cargo.toml`) для Word/Symbol/Newline split; обёртка в `rust/src/tokenize/` маппит в owned DTO.
- **Why:** узкая задача split+Unicode/кириллица; lemmatize сознательно позже. Explore зафиксировал crate.
- **Alternatives:** свой regex/splitter (хуже Unicode edge cases); тяжёлый NLP сразу (out of scope).
- **FFI:** не отдавать `&str`/borrowed tokens; только owned `String` + `u32` spans.
- **WASM / mobile:** `panko` 0.2.0 — чистый Rust (`itertools`, `unicase`), без C/sys; ожидаемо собирается в native FFI и FRB WASM. При apply проверить `cargo check` (и web-путь при необходимости). API wrap за своим модулем — защита от drift молодого crate.

### D2 — Char spans canonical for Flutter

- **Choice:** DTO несёт `char_start` / `char_end` (Unicode scalar indices в тексте cue, end exclusive). Hit-test/paint используют только char; byte fields во FFI не экспортировать в этом change.
- **Why:** Dart `substring` / `TextSpan` работают не с UTF-8 byte offsets; explore зафиксировал byte-only как ловушку для кириллицы.
- **Алгоритм обёртки поверх `panko::tokenize`:**
  1. Вход `S` — **тот же** display-string реплики, что рисует Flutter: `cue.textLines.join('\n')` (один `\n` между линиями; не inventить другой join).
  2. Вызвать `panko::tokenize(&S)`. Экспортировать **только** `Token::Word`; `Symbol` и `Newline` — только для продвижения курсора.
  3. Для каждого `Word(w)`: `byte_start` = смещение `w` в `S` (через разницу указателей/`as_ptr` или последовательный consume); `char_start = S[..byte_start].chars().count() as u32`; `char_end = char_start + (w.chars().count() as u32)`; `surface = w.to_string()`.
  4. `Newline` соответствует одному разделителю строк в `S` (ожидаемо U+000A); сдвигать char/byte курсор согласованно с тем, как `str::lines()` вырезал разделитель. Вход с `\r\n` не целевой для Reader join, но обёртка не должна паниковать.
  5. Инвариант RT-12: срез `S` по char-диапазону == `surface`. Пустая / whitespace-only реплика → `tokens: []`.

### D3 — Module layout + locked FRB surface

- **Choice:** core `rust/src/tokenize/` (pipeline + mapping panko → tokens); thin FRB `rust/src/api/tokenize.rs`; `pub mod tokenize` в `api/mod.rs`; `mod tokenize` в `rust/src/lib.rs` (рядом с `parse`).
- **Locked FRB batch (имена для apply):**

```rust
/// Word token for one cue display string (FFI-owned).
pub struct TokenDto {
    pub surface: String,
    pub char_start: u32, // inclusive Unicode scalar index in cue text
    pub char_end: u32,   // exclusive
    pub token_index: u32, // 0-based among word tokens in this cue
}

pub struct CueTokensDto {
    /// Index into the input `texts` vec (same order as batch args).
    pub cue_index: u32,
    pub tokens: Vec<TokenDto>,
}

/// Batch-tokenize cue display strings. One `CueTokensDto` per input, same order.
/// Infallible for valid Rust `String` (UTF-8): empty input → empty `tokens`.
pub fn tokenize_cues(texts: Vec<String>) -> Vec<CueTokensDto>;
```

- Per-cue internal helper (`tokenize` one `&str` → `Vec<TokenDto>`) OK для unit-тестов; Reader MUST звать **batch** `tokenize_cues` при open.
- Non-word gaps **не** экспортируются; Flutter восстанавливает interstitial text из `cueText` между word spans.
- **Ошибки:** домен tokenize не возвращает `Result` (нет hard-fail на обычном тексте). User-visible degrade (plain text + SnackBar) — на сбой FRB/init/вызова с Flutter-стороны, не на typed domain error.
- **Stub key:** при сохранении `lemma`/ключ = существующий Flutter `normalizeLemma` (trim + case-fold) от surface/фразы; morph поле / `lemma_key` в Rust DTO **не** вводить в этом change. Persist словаря — **Drift в Flutter**, не Rust SQLite.

### D4 — Batch on Reader open (loading / error locked)

- **Choice:** в `ReaderScreen` после mount (когда `originalCues` уже в ctor) — один раз собрать  
  `texts = originalCues.map((c) => c.textLines.join('\n'))`, вызвать injectable `tokenizeCues`  
  (default = generated FRB `tokenizeCues`), кэш `List<CueTokensDto>` / map by `cueIndex` в state.  
  Rebuild строк **только читает кэш** — без повторного FRB.
- **Why:** per-line FRB на build убивает perf (explore).
- **Loading (locked):** пока batch in-flight — original cues **уже видны** как plain text  
  (тот же join `\n`, как сейчас `CueRow`); **taps/selection выключены** (нет token hit-test).  
  Не блокировать весь Reader full-screen spinner’ом (split / scroll / back остаются доступны).  
  Опциональный лёгкий индикатор (AppBar/`LinearProgressIndicator`) — MAY, не MUST.
- **Error (locked):** сбой вызова FRB / init / injectable (`AnyhowException`, `PanicException`, иной) →  
  `SnackBar` с `Key('readerTokenizeErrorSnack')`, сообщение в духе «Could not prepare word selection»;  
  остаться на plain text без token selection; **без** auto-retry (повтор — закрыть/открыть Reader).  
  Library / split / reading-anchor не ломаются. Domain tokenize infallible — typed domain error не ожидается.

### D5 — Selection state machine (Flutter, locked)

- **State:** `activeCueIndex` + inclusive token range `[startToken, endToken]`  
  (`token_index` из DTO; инвариант `start ≤ end`, contiguous). Пусто = нет selection / нет CTA.
- **Cue display string:** всегда `cueText = cue.textLines.join('\n')` (тот же, что ушёл в batch).
- **Tap rules (same cue, tokens ready):**
  1. Selection пуст → select `[T,T]`, показать CTA.
  2. Есть `[S,E]`, tap `T`:
     - `T == S - 1` или `T == E + 1` → **expand** range до включения `T`.
     - `S ≤ T ≤ E` и `(E - S) ≥ 1` → **shrink** до одиночного якоря `[T,T]`  
       (повторный tap по единственному выбранному слову — no-op, CTA остаётся).
     - иначе (gap) → **reset** к `[T,T]` + brief visual feedback на новом якоре.
  3. Другая реплика оригинала → очистить предыдущий selection, начать `[T,T]` там (без gap-flash).
- **Phrase text (RT-10):**  
  `cueText.substring(tokens[S].charStart, tokens[E].charEnd)`  
  (Dart `substring` по Unicode scalar indices = те же char offsets, что в DTO).
- **Gap feedback (locked):** ~300ms усиленная подсветка нового якоря  
  (`Duration(milliseconds: 300)`), затем обычный selected style; точные цвета — из `Theme` /  
  `work-with-styles` (не Visual Spec / не mock).
- **Gestures:** per-token `TapGestureRecognizer` / span hit-test на original pane only;  
  **не** вешать конкурирующий cue-level `InkWell`/`GestureDetector` на тот же tap.  
  Translation pane — без token recognizers (RT-08).

### D6 — Chrome: bottom CTA + Modal bottom sheet

- **CTA:** когда selection non-empty — `Scaffold.bottomNavigationBar` (или эквивалентный  
  persistent bottom `Material` bar внутри того же `Scaffold` с `Key('readerScreen')`),  
  кнопка/bar `Key('readerAddVocabCta')`, label **Add to vocabulary**.  
  Пустой selection → bar отсутствует (`bottomNavigationBar: null`).
- **Sheet:** по CTA — `showModalBottomSheet` (isScrollControlled по необходимости для клавиатуры).  
  Поля:
  - surface/phrase — read-only, prefill = phrase из D5; `Key('readerAddVocabSurface')`
  - translation — editable, initially empty, required; `Key('readerAddVocabTranslation')`
  - context — editable, prefill = полный `cueText` текущей реплики; `Key('readerAddVocabContext')`
  - Save — `Key('readerAddVocabSave')`; sheet root `Key('readerAddVocabSheet')`
- **Validation:** пустой translation после trim → **не** вызывать persist; показать ошибку в sheet  
  (`Key('readerAddVocabValidationError')`), sheet остаётся открытым (RT-07).  
  Пустой surface не ожидается при non-empty selection.
- **Persist:** `await dictionaryRepository.add(lemma: phraseSurface, translation: …, context: …)`  
  — тот же upsert/`normalizeLemma`, что Vocab UI.  
  `DictionaryValidationException` → текст в `readerAddVocabValidationError`.  
  Иной persist failure → `SnackBar`, sheet не закрывать.
- **After success:** закрыть sheet; очистить selection (CTA скрывается); **без** lemma highlight refresh.

### D7 — Original pane only

- **Choice:** tokenize + selection только для original cue list. Translation pane остаётся plain text  
  (`reader/split` без изменений контракта pairing).

### D8 — DI: DictionaryRepository + Library → Reader (locked)

**Pre-change:** `AppShell` держал `DictionaryRepository` только для `VocabularyScreen`;  
`LibraryScreen` / `ReaderScreen` repo словаря не принимали. **Delivered:**

1. **`AppShell`:** передаёт `_dictionary` в `LibraryScreen(dictionaryRepository: _dictionary, …)`  
   (рядом с folder/document repos).
2. **`LibraryScreen`:** required ctor param `DictionaryRepository dictionaryRepository`;  
   в `_openDocument` / `Navigator.push` →  
   `ReaderScreen(…, documentRepository: …, dictionaryRepository: widget.dictionaryRepository, …)`.
3. **`ReaderScreen`:** required ctor param `DictionaryRepository dictionaryRepository`  
   (тот же стиль injection, что `documentRepository`). Sheet/save использует только его —  
   не создавать `DictionaryRepository(db)` внутри Reader и не ходить в global DI.
4. **Tests:** `reader_screen_test` / library pumps — real/test `DictionaryRepository`;  
   Reader pumps обновлены под ctor.

### D9 — Test seams / Keys (Flutter)

- **Injectable tokenize (как `ParseSubtitlesFn`):**  
  `typedef TokenizeCuesFn = Future<List<CueTokensDto>> Function(List<String> texts);`  
  (или sync `List<…> Function(…)` если generated sync — подогнать к фактической FRB-сигнатуре после codegen).  
  Ctor `ReaderScreen.tokenizeCues` optional; default = generated `tokenizeCues`.  
  Widget-тесты сценариев selection/sheet/persist (см. Traceability в specs) — **fake** tokenize + fake/real Drift repo;  
  не требовать живой Rust в widget tests.
- **Keys (locked для finders):**  
  `readerTokenizeErrorSnack`, `readerAddVocabCta`, `readerAddVocabSheet`,  
  `readerAddVocabSurface`, `readerAddVocabTranslation`, `readerAddVocabContext`,  
  `readerAddVocabSave`, `readerAddVocabValidationError`;  
  per-token hit targets: `readerCueToken_${cueIndex}_${tokenIndex}` на original pane  
  (translation pane — без таких Keys / без selection).  
  Существующие `readerCue_*` / list Keys сохранить.
- Selection flash gap-reset **не** обязан иметь отдельный Key; сценарий RT-04 проверяет  
  итоговое выделение + видимость CTA после `pump` / `pump(Duration(milliseconds: 300))`.

### Prerequisites / resolved explore blockers

| ID | Resolution |
|----|------------|
| Tokenizer crate | `panko` **0.2.0** (pin); pure Rust → WASM/mobile OK |
| Lemmatize | out of scope; stub normalize (Flutter) |
| Gesture | tap-only + bottom CTA |
| Offline translate | out of scope; manual translation |
| Gaps / multi-cue / highlight / chrome | см. proposal Out of scope + Decisions D5–D6 |
| Dict DI path | D8: Shell → Library → Reader |
| Loading / FRB error UX | D4 locked |
| Widget-test seams | D9: `TokenizeCuesFn` + Keys |

## Integration points (files)

| Area | Point |
|------|--------|
| Rust deps | `rust/Cargo.toml` — `panko = "0.2.0"` |
| Rust core | new `rust/src/tokenize/` (+ `mod tokenize` in `lib.rs`; tests RT-11/RT-12) |
| Rust FRB | `rust/src/api/tokenize.rs` (`tokenize_cues` + DTOs), `api/mod.rs` → codegen |
| Shell DI | `lib/app/shell.dart` — pass `dictionaryRepository` into `LibraryScreen` |
| Library open | `lib/features/upload/library_screen.dart` — accept dict repo; pass into `ReaderScreen` on push |
| Flutter Reader | `lib/features/reader/reader_screen.dart` — ctor dict repo + optional `tokenizeCues`; batch; CTA; sheet |
| Flutter cue UI | `lib/features/reader/cue_row.dart` (+ selection/selectable helpers under `lib/features/reader/`) — RichText/spans from cache; token Keys |
| Persist | `lib/data/dictionary/dictionary_repository.dart` — reuse `add` / `normalizeLemma` (no API change) |
| Tests | `test/features/reader/reader_screen_test.dart` (+ helpers); update Library pumps if ctor breaks |
| Vocab | без обязательных UI-правок; записи видны через list |

Чеклист реализации — `tasks.md` (не дублировать здесь).

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| `panko` молодой / API drift | pin `0.2.0`; wrap behind `tokenize/`; tests on fixtures |
| Char vs panko byte slices / Newline | D2 algorithm; invariant RT-12; same `\n` join as UI |
| Stub keys ≠ future lemmas | accept MVP drift; later lemma-change may migrate keys |
| Large documents: one big batch on UI isolate | measure; if jank — move FRB to isolate / chunk batch later without changing product rules |
| Gesture conflicts with scroll | use Text.rich recognizers carefully; prefer clear selection on scroll only if needed (optional later) |
| FRB/init failure mid-open | D4: plain text + `readerTokenizeErrorSnack`; do not brick Library |
| Ctor churn in existing Reader/Library tests | D8/D9: required dict repo + optional tokenize fake; update pumps in same change |

## Migration Plan

- Нет миграции Drift schema (поля dictionary уже есть).
- Нет данных токенов на диске.
- Rollback = revert change; Vocab CRUD остаётся.

## Open Questions

Нет блокеров. Visual Spec / mock — **n/a** (этот change не mock-driven).
