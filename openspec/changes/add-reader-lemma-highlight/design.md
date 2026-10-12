## Context

See `proposal.md` — Why / Scope. **Сейчас в sibling app:** Rust `tokenize/` + FRB `tokenize_cues` (`panko` 0.2.0); Reader — batch tokenize, contiguous tap selection, Add CTA/sheet → Drift `DictionaryRepository` с stub `normalizeLemma` (единственная unique-колонка `lemma`). Lemma morph / `is_in_vocab` / highlight paint — отсутствуют. Drift `schemaVersion` = 3.

Слои: **Rust** lemmatize + membership flags (тонкий FRB); **Flutter** Reader paint/gestures + Drift schema/migration + theme tokens. Generated `lib/src/rust/**` не править вручную. Чеклист реализации — `tasks.md`.

Explore-решения (закрыты): morph сразу; `wordnet-lemmatizer`; EN-only WordNet всегда; Rust `is_in_vocab`; single-token highlight/Add; dual-key + one-shot schema migration; merge коллизий по max `updated_at`; phrases не highlight/не Add; tap in_vocab → edit-sheet; theme green tint.

## Goals / Non-Goals

**Goals:**

- Добавить EN morph lemma_key на токены; membership `is_in_vocab` в Rust; paint + refresh в Reader.
- Выпилить stub-identity: Drift `lemma` = morph; `surface_key` = legacy/casefold surface.
- Сузить Reader Add до single-token; in_vocab → edit-sheet (upsert/delete).
- Одноразовая migration stub→morph с merge.

**Non-Goals (design-level):**

- Phrase spans / phrase save; RU morph backend; settings language gate.
- Persist токенов на диск; highlight на translation pane.
- Offline auto-translate; ручной UI разбора merge-коллизий.
- Новый Reader chrome сверх Material bottom sheet (edit = тот же паттерн sheet).

## Decisions

### D1 — Crate: `wordnet-lemmatizer` (EN morph)

- **Choice:** `wordnet-lemmatizer = { version = "0.1", default-features = false }` в `rust/Cargo.toml` рядом с уже существующим `panko = "0.2.0"`.
- **Why:** crates.io `0.1.0` — embedded WordNet, `Lemmatizer::embedded()`, `no_std`+alloc без runtime файлов — fit mobile/WASM; зафиксировано explore D2. API: `lemmatize(word, Pos)` → shortest lemma или вход unchanged; `Pos::{Noun,Verb,Adj,Adv}`.
- **API usage:** один процессный `Lemmatizer` через `std::sync::OnceLock` (не вызывать `embedded()` на каждый токен); POS cascade Noun → Verb → Adj по D3; Adv не в cascade.
- **Alternatives rejected:** stub-only highlight (D1=B отвергнут); отложить morph; `morph-rs` (внешние словари / не WASM-friendly для нашего канона).
- **Применить:** обёртка в новом `rust/src/lemmatize/`; тонкий export в `api/` (`tokenize.rs` и/или `lemmatize.rs`). Не класть бизнес-логику только в `api/`.

### D2 — FRB: расширить tokenize batch + mark membership + surface→lemma helper

- **Choice:** не плодить отдельный `process_cue_text` с нуля. Расширить существующий pipeline:

```text
tokenize_cues(texts)           # surface + char spans + lemma_key; is_in_vocab=false
  → mark_vocab_tokens(...)     # is_in_vocab from Drift lemma set
lemma_keys_for_surfaces(...)   # Vocab UI / Drift migration (без tokenize)
```

- **Locked DTO shape (имена для apply; codegen после правок api/):**

```rust
pub struct TokenDto {
    pub surface: String,
    pub char_start: u32,
    pub char_end: u32,
    pub token_index: u32,
    pub lemma_key: String,
    pub is_in_vocab: bool, // default false until mark pass
}

pub struct CueTokensDto {
    pub cue_index: u32,
    pub tokens: Vec<TokenDto>,
}

/// Batch tokenize + lemmatize. is_in_vocab initially false.
pub fn tokenize_cues(texts: Vec<String>) -> Vec<CueTokensDto>;

/// Set is_in_vocab from morph lemma set (casefold keys from Drift).
pub fn mark_vocab_tokens(
    cues: Vec<CueTokensDto>,
    vocab_lemmas: Vec<String>,
) -> Vec<CueTokensDto>;

/// Same D3 algorithm without tokenize — Vocab manual add + stub→morph migration.
/// One output per input, same order. Whitespace surfaces → casefold only (phrase path).
pub fn lemma_keys_for_surfaces(surfaces: Vec<String>) -> Vec<String>;
```

- **Why split mark:** Drift владеет словарём; Rust не читает SQLite. Flutter передаёт `List` morph lemmas → один batch mark после tokenize и после dict mutation. Альтернатива «один вызов texts+lemmas» допустима как оптимизация, но контракт выше — default из explore Q7.
- **Why `lemma_keys_for_surfaces`:** D4 migration и Vocab UI не имеют token batch; без helper Flutter не может получить WordNet ключ (Dart-only WordNet запрещён). Singular `lemma_key_for_surface` не нужен — batch покрывает оба call site.
- **Membership match:** casefold обоих сторон перед equality (`lemma_key` токена vs каждый `vocab_lemmas[i]`); фразовые ключи с whitespace не совпадут с single-token `lemma_key` → RH-08 by construction.
- **Breaking для Flutter callers:** добавить поля в `TokenDto` + новые FRB fn — обновить Reader cache / tests / fakes `TokenizeCuesFn` (+ mark/lemma seams) в том же change (+ codegen).

### D3 — Lemma key algorithm (locked)

Единая функция `lemma_key_for_surface(surface: &str) -> String` (внутренняя; batch FRB = map):

1. `trimmed = surface.trim()`; если пусто → `""`.
2. `lower = casefold(trimmed)` (Rust: Unicode lowercase; согласовать с Flutter `toLowerCase` для BMP; тесты EN ASCII + кириллица RH-11 / DE-17).
3. Если `lower` содержит whitespace (фраза / multi-word stub) → return `lower` **без** WordNet (DE-16 / D4.4).
4. Иначе POS cascade на **`lower`** (не raw surface): для каждого `Pos` в порядке **Noun → Verb → Adj** вызвать `lemmatizer.lemmatize(&lower, pos)`.
5. **Первый** кандидат `c`, где `c != lower`, становится `lemma_key` (пример RH-10: `dogs`→Noun→`dog`; `running`→Noun может вернуть `running` → Verb→`run`).
6. Если все POS вернули `lower` (или эквивалент unchanged) → `lemma_key = lower` (morph-fail / не-EN / junk).
7. Identity в Drift = `lemma_key`; `surface_key = lower` при save из Reader (для фразы `surface_key = lemma = lower`).

### D4 — Drift dual-key + migration (schema 3 → 4)

- **Choice:** таблица `dictionary_entries`:
  - `lemma` TEXT UNIQUE — morph identity (как сейчас unique, семантика меняется).
  - `surface_key` TEXT — casefold surface / legacy stub (не обязательно unique).
  - прочие поля без изменений (`translation`, `context`, timestamps).
- **Migration (один раз в `onUpgrade` from→to):**
  1. Добавить колонку `surface_key` (nullable → backfill).
  2. Для каждой строки: `old = lemma`; вычислить morph key через FRB **`lemma_keys_for_surfaces`** (тот же D3; Dart-only WordNet запрещён) — batch из migration bootstrap / app open path, один раз (флаг migrated / schemaVersion bump).
  3. Коллизии одинакового morph: оставить ряд с max `updated_at`; `surface_key` победителя = его old; проигравшие удалить (translation/context проигравших отбросить — later manual merge UI).
  4. Multi-word `old` (содержит whitespace): **не** схлопывать через WordNet wordform; `lemma = casefold(old)` (фраза остаётся ключом), `surface_key = lemma`; не участвует в single-token highlight.
- **Practical apply note (locked FE bootstrap):** `main` уже делает `RustLib.init` до `runApp`; `TranslationReaderApp` открывает Drift после init — FRB доступен. **Предпочтительно:** в `onUpgrade` 3→4 только DDL (`surface_key` column); morph rewrite — **post-open** one-shot `DictionaryRepository.migrateStubKeysIfNeeded(lemmaKeysForSurfaces: …)` сразу после успешного `openAppDatabase` (в `TranslationReaderApp` / shell bootstrap), идемпотентно. Не вызывать FRB из sync-only пути и не на каждый cold start после успеха.
- **Repository:** выпилить stub-only `normalizeLemma` как sole identity — `normalizeSurface` (casefold) для `surface_key`; morph `lemma` приходит с call site (D5.1). Upsert/lookup по morph `lemma`.
- **Upsert:** по `lemma` (morph).

### D5 — Reader UX: paint + gesture fork

- **Files (touch):** `lib/features/reader/reader_screen.dart`, `cue_row.dart`, sheet widgets (существующий Add sheet → reuse/extend для edit), `tokenize_cues_fn.dart` (+ mark / lemma-keys seams), theme (`lib/app/` ThemeData + `ThemeExtension`).
- **Paint:** per-token background из `Theme.of(context).extension<ReaderColors>()!.lemmaHighlight` (light+dark). Не hardcode green в `cue_row`.
- **Selection machine (locked change vs previous contiguous MVP):**
  - Только single-token для Add path (`selectionStart == selectionEnd`); contiguous expand / multi-token range / gap-flash **удалить**.
  - Tap token `is_in_vocab` → clear Add selection → open edit sheet (lookup Drift by `lemma_key`), **no** Add CTA.
  - Tap non-vocab → select that single token + Add CTA; tap другого non-vocab → replace selection (no expand).
  - Повторный тап по тому же non-vocab single → no-op (CTA остаётся).
- **After save/delete from Reader sheets:** перечитать morph lemma list из Drift → `markVocabTokens` на кэше → setState. Не флипать `is_in_vocab` только в Dart.
- **Vocab-screen delete while Reader not mounted:** достаточно mark на следующем открытии Reader (маршрут pushed; не строить cross-tab stream ради RH-04). Edit-sheet delete при открытом Reader → сразу `markVocabTokens`. Если Reader снова открыт в той же сессии после Vocab delete — open path уже делает tokenize→mark.
- **DI seams (widget tests):** рядом с `TokenizeCuesFn` — injectable `MarkVocabTokensFn` и `LemmaKeysForSurfacesFn` (default = generated FRB). Reader ctor optional overrides; Vocab / migration bootstrap используют тот же lemma-keys seam (через repo helper или screen).
- **Keys (locked):**
  - Add (сохранить): `readerAddVocabCta`, `readerAddVocabSheet`, `readerAddVocabSurface`, `readerAddVocabTranslation`, `readerAddVocabContext`, `readerAddVocabSave`, `readerAddVocabValidationError`
  - Edit: `readerEditVocabSheet`, `readerEditVocabLemma` (read-only morph), `readerEditVocabSurface` (read-only surface), `readerEditVocabTranslation`, `readerEditVocabContext`, `readerEditVocabSave`, `readerEditVocabDelete`, `readerEditVocabValidationError`
  - Tokens: `readerCueToken_${cueIndex}_${tokenIndex}` без изменений.

### D5.1 — Morph key ownership (Flutter)

- **`DictionaryRepository` не вызывает WordNet / FRB сам по себе** в unit-тестах без inject: принимает уже готовый morph `lemma` + `surfaceKey` (casefold).
- **Reader Add/edit save:** `lemma = token.lemmaKey`, `surfaceKey = casefold(token.surface)` (или helper `normalizeSurface`).
- **Vocab manual add:** UI (или тонкий injected helper) → `lemmaKeysForSurfaces([input])` → `repo.add(lemma: morph, surfaceKey: casefold(input), …)`.
- **Migration rewrite:** post-open `migrateStubKeysIfNeeded()` вызывает тот же `LemmaKeysForSurfacesFn` batch; идемпотентно (флаг / «все rows уже имеют surface_key и schema 4»).

### D6 — Theme tokens

- **Choice:** `ThemeExtension` **`ReaderColors`** (канон `work-with-styles` / theme.md) с `lemmaHighlight` / `lemmaHighlightOn` (+ существующий/новый selection token при необходимости) для light и dark; разумный green tint без Visual Spec. Wire на `MaterialApp` theme/darkTheme (сейчас в app — только `primarySwatch`; заменить/расширить на явную пару ThemeData).
- **Why:** единое имя с skills; Q8.

### D7 — Layers / ownership

| Concern | Owner |
|---------|--------|
| Tokenize spans | Rust `tokenize/` (unchanged algorithm) |
| Lemma key | Rust `lemmatize/` + WordNet (`lemma_keys_for_surfaces` + внутри `tokenize_cues`) |
| `is_in_vocab` | Rust `mark_vocab_tokens` |
| Dict CRUD / migration orchestration | Flutter Drift (morph rewrite через FRB helper) |
| Paint / gestures / sheets | Flutter Reader |
| Generated bindings | FRB codegen only |

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| WASM/binary size от embedded WordNet | `default-features = false`; verify `flutter build web` / apk size smoke в apply |
| WordNet POS cascade даёт «не ту» лемму | Зафиксировать fixtures RH-10; later POS tagger out of scope |
| Migration до `RustLib.init` | Идемпотентный post-init migrate; schemaVersion всё равно bump |
| Merge отбрасывает translation проигравшего ряда | Зафиксировано explore; manual merge later |
| Breaking TokenDto | Один change: codegen + все fakes/tests |
| Phrase rows vs single-token highlight | Explicit non-highlight; Add phrases disabled |
| Flutter `toLowerCase` vs Rust casefold drift | Shared fixtures EN+RU; document algorithm |

## Migration Plan

1. Bump Drift schema; add `surface_key`.
2. Ship Rust lemmatize + FRB + codegen.
3. On first app open after upgrade: rewrite rows → morph `lemma` + `surface_key`; merge collisions.
4. Reader: tokenize → mark → paint; single-token Add; edit on in_vocab.
5. Rollback: app store version pin; DB downgrade не поддерживаем (как обычно для Drift) — forward-only.

## Open Questions

Нет блокирующих. Мелочи при apply: точные Material hex для green tint (`ReaderColors` light/dark) — разумный green без Visual Spec OK.
