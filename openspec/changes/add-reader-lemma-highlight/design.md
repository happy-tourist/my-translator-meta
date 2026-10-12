## Context

See `proposal.md`. **Foundation уже в sibling app (apply ##1–##4):** `wordnet-lemmatizer` + `lemmatize/`; `TokenDto.lemma_key` / `is_in_vocab`; FRB `tokenize_cues` / `mark_vocab_tokens` / `lemma_keys_for_surfaces`; Drift schema **4** + `surface_key` + post-open migrate; `ReaderColors`; single-token Add + edit sheet; morph-only mark (phrase keys **не** красят — RH-08 old).

**Этот delta:** вернуть contiguous phrase Add (tap-vocab machine) + phrase membership/paint/edit priority. Generated `lib/src/rust/**` не править вручную. Чеклист — `tasks.md` (новые ##5+; ##1–##4 остаются `[x]`).

Explore (phrase restore, locked): phrase key = casefold(join token surfaces); match consecutive tokens (not cue substring); morph only singles; paint all span tokens; overlaps paint both; tap priority longest phrase > shorter > single; selection expand/shrink/gap-flash ~300ms; legacy rows user deletes.

## Goals / Non-Goals

**Goals:**

- Contiguous multi-token Add + phrase persist key = casefold(join).
- Phrase + morph membership в одном mark pass; paint всех `is_in_vocab` токенов.
- Edit по тапу с overlap resolution; singles morph без регрессии.

**Non-Goals:**

- Morph-join внутри фразы; cue-substring identity; RU morph; translation-pane highlight; drag/long-press; legacy cleanup UI.

## Decisions

### D1 — Crate: `wordnet-lemmatizer` (EN morph) — **shipped**

Без изменений. `default-features = false`; `OnceLock`; POS Noun→Verb→Adj. Whitespace surfaces → casefold only (фразы не через WordNet).

### D2 — FRB mark: morph **и** phrase windows

- **Shipped:** `mark_vocab_tokens(cues, vocab_lemmas)` — casefold equality `token.lemma_key` ∈ vocab set (singles).
- **Delta:** тот же (или расширенный) mark MUST также:
  1. Разделить `vocab_lemmas` на **single** (без whitespace) и **phrase** (с whitespace после casefold/trim).
  2. Single: как сейчас — `lemma_key` ∈ single_set → `is_in_vocab`.
  3. Phrase: для каждой реплики скользящие consecutive windows длины = числу слов ключа; `casefold(join(surfaces, " "))` == phrase key → пометить **все** токены window `is_in_vocab = true` (OR с morph hit).
  4. Overlaps: токен может входить в несколько windows — остаётся `true` (paint both).
- **Alternatives rejected:** Flutter-only phrase scan (нарушает RH-09); cue substring match (user: token-join only).
- `lemma_keys_for_surfaces`: без изменений для singles; multi-word input → casefold only (уже D3 step 3).
- Codegen только если меняется сигнатура; предпочтительно **без** breaking DTO — логика внутри `mark_vocab_tokens`.

### D3 — Lemma key algorithm — **shipped** (singles)

Без изменений. Phrase dict keys никогда не проходят POS cascade (whitespace → casefold).

### D4 — Drift dual-key schema 4 — **shipped**

Без schema bump. Phrase rows: `lemma` = casefold(join); `surface_key` = тот же. Upsert по `lemma`. Repo по-прежнему не вызывает FRB.

### D5 — Reader UX: contiguous selection + paint + edit

- **Files:** `reader_screen.dart`, `cue_row.dart`, sheets, `tokenize_cues_fn.dart`, theme (уже `ReaderColors`).
- **Selection machine (restore tap-vocab):**
  - State: cue + `selectionStart`/`selectionEnd` (inclusive token indices).
  - Non-vocab: first tap → anchor; adjacent outside → expand; inside range len≥2 → shrink-to-single; gap → reset + ~300ms flash; other cue → move.
  - Retap sole selected non-vocab → no-op.
  - `is_in_vocab` tap → clear Add selection → edit (D8); no CTA.
- **Paint:** `is_in_vocab` → `ReaderColors.lemmaHighlight` (уже).
- **Add save (D5.1):**
  - Single (`start==end`): `lemma = token.lemmaKey`, `surfaceKey = normalizeSurface(surface)`.
  - Phrase: `lemma = surfaceKey = casefold(join selected surfaces with " ")`.
- **After save/delete:** Drift `list()` lemmas → `markVocabTokens` → setState.
- **Keys:** без изменений vs shipped Add/Edit keys; Edit lemma/surface read-only показывают выбранный dict key / display surface (фраза = join).

### D5.1 — Key ownership (Flutter) — updated

- Repo принимает готовый `lemma` + `surfaceKey`.
- Reader: branch single vs phrase как D5.
- Vocab: singles → `lemmaKeysForSurfaces`; multi-word typed input → casefold identity (no WordNet).

### D6 — Theme `ReaderColors` — **shipped**

Без изменений.

### D7 — Layers / ownership — updated

| Concern | Owner |
|---------|--------|
| Tokenize / lemma_key | Rust (shipped) |
| Morph + phrase membership | Rust `mark_vocab_tokens` |
| Dict CRUD | Flutter Drift |
| Selection / paint / sheets | Flutter Reader |
| Generated bindings | FRB codegen only |

### D8 — Overlap tap resolution

При тапе на токен с несколькими hits: выбрать dict entry **longest phrase span (token count) > shorter phrase > single morph**. Lookup Drift by that key. Paint остаётся OR всех hits (D2).

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Phrase window mark O(n·m) на длинных cues | Типичные cue короткие; при необходимости cap max phrase length later |
| Join ≠ cue substring (punct) | Зафиксировано user; fixtures RH-14 |
| Regression single-only tests (RT-03/14 old) | Переписать под contiguous; RH-08 inverted |
| Stale web/pkg after mark change | `build-web` в verify tasks |

## Migration Plan

1. Foundation schema 4 / morph — уже live.
2. Extend mark + restore selection + phrase save/edit priority.
3. Update widget/Rust tests; `flutter analyze` / `cargo test` / targeted `flutter test`.
4. User deletes unwanted legacy stub rows manually.

## Open Questions

Нет блокирующих.
