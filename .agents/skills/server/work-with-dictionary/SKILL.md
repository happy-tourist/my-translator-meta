---
name: work-with-dictionary
description: >-
  Personal vocabulary domain for Translation Reader (my_translator_app): morph
  and phrase lemma keys, translation/context model, lookup-by-lemma for reader
  highlight. User CRUD persistence is Drift in Flutter lib/ (not Rust SQLite /
  FRB storage). Use when clarifying Rust NLP handoff to vocab, lemma-key
  contracts, or thin FRB compute helpers — not for implementing Drift DAOs
  (client skills).
---

# Work With Dictionary

Use this skill for the **personal vocabulary (dictionary)** domain in
**my_translator_app** (product: **Translation Reader**).

**Persistence canon (current):** dictionary CRUD (`dictionary/entries`) lives
in **Drift** on Flutter `lib/`. Rust does **not** own user SQLite
(`rusqlite` / `diesel` / `rust/src/db` for user data — **not** required).
FRB storage API for dictionary is **not** part of the Drift-library change.

**Rust role:** morph `lemma_key` + membership flags (`mark_vocab_tokens`) and
`lemma_keys_for_surfaces` for Vocab/migration (shipped with
`add-reader-lemma-highlight`). Domain *rules* about lemma identity stay here;
**persist rows = Drift**, not FRB.

**Paths:** this skill lives in **my_translator_meta** at
`.agents/skills/server/work-with-dictionary/`. Runtime paths (`rust/src/…`,
`lib/…`) are relative to the **my_translator_app** sibling
(`../my_translator_app` from meta; see `docs/projects-map.md`). Read
`{app}/AGENTS.md` before changing layers.

Stack: **Rust** edition `2021`, crate `rust_lib_my_translator_app`, public FFI
in `rust/src/api/`, FRB `2.13.0` (in-process) for **compute**; user store =
Drift.

## Scaffold Honesty

**There is no dictionary CRUD module in Rust and no rusqlite for user vocab.**
Shipped FRB compute includes `api/parse` + `api/tokenize` (morph fields +
`mark_vocab_tokens` / `lemma_keys_for_surfaces`); demo `api/simple`
(`process_text` / `add_numbers`) must **not** grow into a fake in-memory dict.
Product vocab persistence stays on the **Flutter/Drift** side. Do **not**
introduce `rust/src/db` for user rows unless a future change explicitly
reassigns persist to Rust.

## When to Use

- Clarify morph lemma (singles) and casefold(join) phrase identity; translations
  + optional context + `surface_key`
- Lookup-by-lemma / membership for Reader highlight: Drift lemmas → FRB
  `mark_vocab_tokens` (morph **or** phrase windows → `is_in_vocab`); Flutter
  paints flags only
- Wire thin FRB exports for **NLP/gloss** that feed vocab — not for storing
  entries
- Review whether a change wrongly adds rusqlite / FRB CRUD for user dict

**Not this skill:** Flutter vocab list/add chrome + Drift DAOs →
`work-with-pages` (`vocab.md`) / `client-work-with-structure`;
tokenize/lemmatize algorithms → `work-with-tokenizematize`; offline gloss
generation → `work-with-translate`; FRB codegen mechanics →
`work-with-frb-api` / client `work-with-frb`; multi-device or cloud sync
(out of scope in `{app}/AGENTS.md`).

## Entry Model (target)

| Field | Role |
|-------|------|
| **Lemma** | Single: morph identity (`lemma_key`). Phrase: casefold(join surfaces). Match key for highlight across wordforms / phrase spans |
| **surface_key** | Casefold surface or phrase join (`normalizeSurface` collapses whitespace); dual-key; not unique |
| **Translation(s)** | One or more gloss strings the user keeps |
| **Context** | Optional source phrase and/or subtitle cue text at add time |
| **Ids / timestamps** | Stable entry id + created/updated in Drift |

Rules:

- Identity for highlight / upsert = **morph lemma** (singles) or **phrase
  join** (multi-word), not raw surface casing. Call sites supply keys (Reader
  single `token.lemmaKey` / phrase join; Vocab: `lemma_keys_for_surfaces` for
  singles, casefold identity for multi-word input) — repo does **not** embed
  WordNet.
- Store **`surface_key`** alongside `lemma` (schema 4; stub→morph
  `migrateStubKeysIfNeeded` post-open). `normalizeSurface` collapses internal
  whitespace so phrase keys match Rust join.
- Membership for paint = Rust `is_in_vocab` via `mark_vocab_tokens` — not
  Dart-only surface compare as sole truth.
- Duplicate lemma policy = upsert by lemma key (morph or phrase) — see OpenSpec
  `dictionary/entries`.

## Operations (product CRUD + lookup)

| Op | Intent | Layer |
|----|--------|-------|
| **Add** | Persist morph/phrase lemma + `surface_key` + translation(s) + optional context | **Drift** (Vocab UI **and** Reader Add sheet) |
| **List** | Return entries for Vocabulary screen | **Drift** |
| **Update** | Change translation(s) and/or context | **Drift** (Reader edit sheet; Vocab edit when UI supports) |
| **Remove** | Delete entry; refresh marks / clear highlight | **Drift** |
| **Lookup by lemma** | Entry fetch for edit sheet; lemma set for mark pass | Drift on Flutter; morph keys from Rust FRB |

## Layering

```text
Flutter (vocab screen / Reader add/edit sheet)
  → FRB lemma_keys_for_surfaces / tokenize_cues   # morph identity
  → Drift dictionary repository                   # persist by morph lemma
  → FRB mark_vocab_tokens                         # is_in_vocab for paint
      → rust/src/api/…                            # thin; no dict storage
```

| Layer | Path | Role |
|-------|------|------|
| Persist | Flutter `lib/` Drift tables/DAOs | CRUD + reopen durability |
| FFI surface | `rust/src/api/` | Thin compute export only — **not** dict storage |
| Tokenize | `rust/src/tokenize/` | Word spans for Reader tap |
| Lemmatize | `rust/src/lemmatize/` (+ FRB) | Morph keys / membership flags |
| Generated | `frb_generated.rs`, `lib/src/rust/**` | **Never** hand-edit |

Flutter must not keep a second source of truth in `SharedPreferences` beside
Drift. Do **not** require `rust/src/db` for dictionary persistence.

## Persistence (canon)

- **User store:** Drift in Flutter (Android / iOS / Web).
- **Do not** add `rusqlite` / `diesel` / `rust/src/db` for personal vocabulary
  unless a future OpenSpec change explicitly moves persist back to Rust.
- Dictionary CRUD of the Drift-library change = **Drift, not FRB**.

## Offline / Sync Scope

- Dictionary is **local-only** and works **offline**.
- **No cloud sync**, no multi-device merge, no shared decks — out of scope
  (`{app}/AGENTS.md`).

## Related Skills (by name)

| Skill | Concern |
|-------|---------|
| `work-with-tokenizematize` | Lemma/token production that feeds the dict key + highlight |
| `work-with-translate` | Offline gloss for word/phrase card before/when adding |
| `work-with-frb-api` | Thin public `api/` surface + FRB export shape (compute) |
| `server-work-with-errors` | Rust error kinds / Result mapping for compute ops |
| `server-work-with-test` | `cargo test` for NLP / lemma helpers |

Client UI + Drift: `work-with-pages` (`vocab.md`), `work-with-reader`,
`client-work-with-structure`.

## Do

- Keep lemma identity aligned with the lemmatizer pipeline.
- Persist optional context with the entry when provided at add (Drift).
- Cover Drift CRUD with Flutter tests (`client-work-with-test`); Rust tests
  cover NLP helpers when they exist.
- After public **compute** API change: edit `rust/src/api/` → codegen → Dart
  call sites.

## Don't

- Reimplement NLP / lemma production in Dart.
- Hand-edit `rust/src/frb_generated.rs` or `lib/src/rust/**`.
- Stuff user CRUD into `api/simple.rs` or invent FRB storage for vocab.
- Introduce `rusqlite` / `rust/src/db` for user dictionary without an
  explicit OpenSpec change.
- Add cloud sync, accounts, or shared decks.
- Commit device DB files or secrets.

## Checklist

When implementing or changing dictionary-related work:

- [ ] Persist path is Drift in Flutter (or an explicit future change says otherwise)
- [ ] Entry fields: morph lemma, `surface_key`, translation(s), optional context
- [ ] Lookup / mark path supports Reader highlight (same morph key as lemmatizer)
- [ ] No rusqlite / `rust/src/db` required for user vocab persist
- [ ] Offline local-only; no sync surface
- [ ] Scaffold honesty: no claims that Rust owns dict SQLite unless code does

## Anti-Patterns

```text
// FRB dictionary CRUD + rusqlite as user store
// → Drift in Flutter is persistence authority

// Dart SharedPreferences as second dictionary truth
// → Drift is the single store

// Highlight by raw string equality in Flutter
// → morph lemma_key + mark_vocab_tokens; Flutter paints is_in_vocab only

// Fake HashMap “dict” left in api/simple forever
// → product vocab via Drift (+ lemmatize/mark FRB compute)
```
