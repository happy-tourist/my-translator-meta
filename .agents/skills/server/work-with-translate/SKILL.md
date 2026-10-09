---
name: work-with-translate
description: >-
  Offline word/phrase translation in Translation Reader Rust core (`rust/`):
  gloss lookup, language-pair settings, lemma-aware results for the reader
  card, save-into-dictionary handoff. Use when implementing or changing
  offline translate modules, translate FRB API, source/target language settings
  for content translation, or distinguishing local offline engines from
  optional future AI providers. Not for Flutter card chrome alone
  (work-with-reader), not for ARB UI locale (work-with-localization), not for
  AI/cloud translate implementation.
---

# Work With Translate (Rust offline)

Target **offline word/phrase translation** for **Translation Reader**
(`my_translator_app`). Logic lives in **Rust** under `{app}/rust/`; Flutter
shows the card and calls a thin FRB surface. Path package /
crate: `rust_lib_my_translator_app`. Resolve `{app}` via
`docs/projects-map.md` (key `app`). Read `{app}/AGENTS.md` before changing
the crate.

**Scaffold today:** product translate is **not implemented**. Demo
`process_text` / `add_numbers` are unrelated — do not grow them into a
translate engine. Prescribe the contracts below so modules land correctly.

## When to Use

- Offline translate of a **word** (token / lemma) or **phrase** (token span)
- Language-pair settings that drive translate direction (learning content
  source → target)
- Wiring translate results into dictionary “add” and reader card data
- Designing / changing the thin `rust/src/api/` translate exports
- Choosing where a concrete offline engine or dictionary pack plugs in

**Not this skill:**

| Concern | Use instead |
|---------|-------------|
| Word/phrase **card UI**, gestures, tap/select | `work-with-reader` (+ `tap.md`) |
| Flutter error chrome (SnackBar / card error slot) | `client-work-with-errors` |
| FRB codegen / `RustLib.init` mechanics | `work-with-frb` |
| UI chrome locale (ARB) | `work-with-localization` |
| Rust test harness layout | `server-work-with-test` |
| **Optional AI** explain/translate providers | Out of scope — boundary only (below) |

## Core Principles

1. **Offline by default.** No network required for word/phrase gloss. No
   required API keys at app start.
2. **Heavy logic in Rust.** Flutter must not invent translations, stemming,
   or engine selection rules in Dart.
3. **Thin FRB API.** Public surface in `rust/src/api/`; algorithms in
   internal modules (e.g. `translate/`). Never hand-edit
   `frb_generated.rs` or `lib/src/rust/**`.
4. **Engine not locked.** This skill defines the **module / trait
   boundary**, not a concrete vendor, crate, or dictionary pack.
5. **Scaffold is not product.** Until modules exist, add
   `translate` + thin `api` + tests together — do not fake glosses in Flutter.

## Boundary: Optional AI (out of scope)

Product vision allows **optional AI later** (phrase nuance, phrasal verbs,
context). That path is **out of scope for this skill**.

- Do **not** implement cloud providers, API-key settings, or prompt pipelines
  under this skill.
- Offline translate must remain usable **without** any AI provider configured.
- If a change mixes offline + AI, keep a clear seam (e.g. separate module /
  trait); offline path must not depend on network or keys.
- Mention AI only as “later / explicit / optional” — never as the default
  gloss source for the reader card.

## Developer Decision (flag, do not invent)

**Concrete offline engine and dictionary pack are a developer decision.**

Until decided, agents must:

- Implement against a **stable interface** (trait / module API) inside
  `rust/src/translate/` (name may vary; keep logic out of `api/`).
- **Not** hard-lock Cargo deps to a specific MT engine, binary model, or
  proprietary pack without an explicit product/dev decision.
- **Flag** in OpenSpec / task notes: *engine + pack TBD* when AC needs a
  real gloss and none is chosen yet.
- Stub / fixture-backed implementations are OK for tests and early UI wiring
  if labeled as temporary behind the same interface.

Acceptable interface shape (illustrative — adapt names to the crate):

```rust
// rust/src/translate/mod.rs (target — not in scaffold)

/// Offline gloss engine. Swap implementations without changing FRB shapes.
pub trait OfflineTranslator: Send + Sync {
    fn translate_word(
        &self,
        query: &WordQuery,
    ) -> Result<TranslationResult, TranslateError>;

    fn translate_phrase(
        &self,
        query: &PhraseQuery,
    ) -> Result<TranslationResult, TranslateError>;
}
```

Packaging notes (non-prescriptive): on-device dictionary file, embedded
resource, or platform asset loaded by Rust — all fine **after** a decision;
document load path next to the chosen impl, not in Flutter.

## Language Pair Settings

| Setting | Role | Owner |
|---------|------|--------|
| Source language | Subtitle / learning content language | Rust settings (+ persist when SQLite lands) |
| Target language | Gloss language for the card | Same |
| UI locale (ARB) | App chrome strings | Flutter — **not** this skill |

Rules:

- Translate APIs take an explicit pair **or** read the persisted default pair
  from settings — pick one contract and keep FRB stable; prefer passing pair
  (or pair id) when the reader session may override defaults.
- Do not conflate UI locale with learning pair (see
  `work-with-localization`).
- Unsupported pair → typed error (not empty success, not panic).

## Integration: Tokenize / Lemmatize

Reader tap resolves a **token span** with surface form + **lemma** (Rust NLP).
Translate must consume that, not re-tokenize in Dart.

| Input | Word path | Phrase path |
|-------|-----------|-------------|
| Surface | Token text under tap | Contiguous selected text |
| Lemma | Required when NLP has it; key for dict save / highlight | Optional per-token lemmas; do **not** force a single lemma for multi-word unless product defines it |
| Context | Cue / line id or snippet for later dict attach | Same |

Contract with NLP modules (target layout `lemma/` / tokenize):

1. Prefer `lemma_id` / lemma string from the already-tokenized span.
2. If only surface is available (tokenize pending), either refuse with a clear
   error or look up after tokenize — **do not** guess stems in the translate
   module differently from the lemma module.
3. Phrase: one `translate_phrase` call for the whole span (matches
   `work-with-reader` / `tap.md`). Do not translate each token and join.

## Integration: Dictionary (save translation)

Translate **produces** gloss data; dictionary **persists** user vocab.

| Step | Owner |
|------|--------|
| Offline gloss for card | `translate` |
| User taps “Add to vocab” | Flutter → dictionary FRB |
| Store lemma + translation(s) + context | `dict` (+ SQLite when present) |
| Refresh highlight marks | Dictionary / lemma match — not translate |

Rules:

- Translate must return fields the dict add API can accept (lemma when known,
  chosen gloss string, optional alternates, context handle).
- Do **not** auto-write dictionary on every translate; save is an explicit
  user action (unless a future AC says otherwise).
- Do not duplicate dict CRUD inside `translate/`.
- Phrase add: only if dictionary API supports phrase entries; otherwise return
  gloss for display and leave add disabled or defined by dict skill/AC.

## Target Module Layout

```text
my_translator_app/rust/src/
  api/                 # thin FRB: translate_word / translate_phrase (+ types)
  translate/           # OfflineTranslator impl(s), errors, pair checks
  lemma/               # tokenize + lemmatize (peer — not owned here)
  dict/                # vocab persist (peer — not owned here)
  frb_generated.rs     # GENERATED — never edit
```

Dependency direction:

```text
api (FRB) → translate → (reads) settings / lemma ids
                ↘ does not own dict writes
Flutter → generated api → orchestrates translate then optional dict add
```

## Thin FRB API (target shapes)

Export from `rust/src/api/` only. Names illustrative; keep args/results
FFI-friendly (owned `String`, plain structs, `Result` / FRB error mapping).

| Export | Responsibility |
|--------|----------------|
| `translate_word` | Surface + lemma + language pair → gloss result |
| `translate_phrase` | Phrase text (+ optional token ids) + pair → gloss result |
| (settings) | Get/set learning language pair — may live in settings API |

Result should be enough for the reader card (see `work-with-reader`):

- Headword / phrase text (echo or normalized)
- Lemma (word path)
- One primary translation; optional alternates
- Stable miss/unsupported signaling (empty gloss ≠ success if lookup failed)

Keep `api/` free of engine I/O details (file paths to packs, model load) —
hide those behind the offline trait/impl.

## Errors

Use a typed `TranslateError` (or equivalent) with stable kinds agents and UI
can map:

| Kind | When |
|------|------|
| `EmptyQuery` | Empty / whitespace-only word or phrase |
| `UnsupportedLanguagePair` | Pair not available for offline engine/pack |
| `NotFound` | No gloss in offline data (card may show “no translation”) |
| `EngineUnavailable` | Pack missing / not loaded / init failed |
| `Internal` | Unexpected failure — log details; safe message outward |

Rules:

- Prefer `Result` / mapped FRB errors over panics for user input.
- Flutter surfaces: transient SnackBar or card error slot
  (`client-work-with-errors`) — Rust supplies kind + safe message, not Material
  widgets.
- Do not require network errors on the offline path.

## Tests

Follow **server-work-with-test**. Place unit tests next to `translate/`
(`#[cfg(test)]`). Cover at least:

- Word: known fixture lemma → expected primary gloss
- Phrase: multi-word fixture → single phrase result (not token join)
- Empty query → `EmptyQuery`
- Unsupported pair → `UnsupportedLanguagePair`
- Missing gloss → `NotFound` (or agreed empty-miss contract)
- Stub/fake `OfflineTranslator` behind the trait for api/orchestration tests
  without a real pack

Run `cargo test` from `{app}/rust`. Do not prove offline translate only via
`flutter test`.

OpenSpec scenarios for offline translate **must** include `cargo test` tasks
mapped to `SC-…` IDs — not `cargo check` alone.

## Flutter UX Expectations (do not implement UI here)

From `work-with-reader` / `tap.md` — Rust must enable:

1. Tap token → word card: lemma, translation(s), context, Add to vocab.
2. Phrase select → phrase card: full phrase + one phrase translation.
3. Loading / error flags owned by Flutter orchestration; data from FRB.
4. No Dart-local invented glosses; no per-token translate+concat for phrases.

## Anti-Patterns

| Avoid | Prefer |
|-------|--------|
| Cloud/AI as default gloss | Offline engine behind trait; AI later, separate |
| API keys required to open reader | Offline path with zero keys |
| Gloss logic in `api/` or Dart | Internal `translate/` + thin export |
| Hard-coding an engine crate “because popular” | Flag developer decision; keep interface |
| Auto-save every lookup to dict | Explicit add via dictionary API |
| Re-stem in translate ≠ lemma module | Consume lemma from tokenize/lemmatize |
| Phrase = join of word glosses | `translate_phrase` once |
| Extending `process_text` demo into MT | New modules + real FRB names |
| Hand-edit generated FRB/Dart | Codegen after `api/` change |

## Checklist

- [ ] Offline-only path; no required API keys
- [ ] Optional AI mentioned only as out-of-scope boundary
- [ ] `OfflineTranslator` (or equivalent) boundary; concrete engine/pack flagged TBD if undecided
- [ ] Language pair from settings / explicit args; not UI locale
- [ ] Word uses lemma from NLP; phrase is one call
- [ ] Dict save is separate explicit step; result fields align with dict add
- [ ] Thin `api/` exports; core in `translate/`
- [ ] Typed errors for empty / unsupported / miss / unavailable
- [ ] `cargo test` coverage in Rust; scaffold demo not abused
- [ ] Flutter card remains in `work-with-reader` scope
