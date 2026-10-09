---
name: work-with-dictionary
description: >-
  Personal vocabulary domain for Translation Reader (my_translator_app): lemma
  keys, translation/context model, lookup-by-lemma for reader highlight. User
  CRUD persistence is Drift in Flutter lib/ (not Rust SQLite / FRB storage).
  Use when clarifying Rust NLP handoff to vocab, lemma-key contracts, or thin
  FRB compute helpers — not for implementing Drift DAOs (client skills).
---

# Work With Dictionary

Use this skill for the **personal vocabulary (dictionary)** domain in
**my_translator_app** (product: **Translation Reader**).

**Persistence canon (current):** dictionary CRUD (`dictionary/entries`) lives
in **Drift** on Flutter `lib/`. Rust does **not** own user SQLite
(`rusqlite` / `diesel` / `rust/src/db` for user data — **not** required).
FRB storage API for dictionary is **not** part of the Drift-library change.

**Rust role:** NLP lemma keys / tokenize-lemmatize / offline gloss that feed
highlight and add-from-Reader **later**. Domain *rules* about lemma identity
may still be documented here; **persist rows = Drift**, not FRB.

**Paths:** this skill lives in **my_translator_meta** at
`.agents/skills/server/work-with-dictionary/`. Runtime paths (`rust/src/…`,
`lib/…`) are relative to the **my_translator_app** sibling
(`../my_translator_app` from meta; see `docs/projects-map.md`). Read
`{app}/AGENTS.md` before changing layers.

Stack: **Rust** edition `2021`, crate `rust_lib_my_translator_app`, public FFI
in `rust/src/api/`, FRB `2.13.0` (in-process) for **compute**; user store =
Drift.

## Scaffold Honesty

**Today there is no dictionary module in Rust and no rusqlite.** Scaffold only
exposes `process_text` / `add_numbers` in `rust/src/api/simple.rs`. Product
vocab persistence is implemented on the **Flutter/Drift** side per OpenSpec
(`add-drift-library-storage` and main specs after sync). Do **not** grow the
demo `simple` API into a fake in-memory “dict” or introduce `rust/src/db`
for user rows unless a future change explicitly reassigns persist to Rust.

## When to Use

- Clarify lemma as the stable key; translations + optional context
- Lookup-by-lemma contract so Reader can highlight saved words (membership
  may be checked in Flutter against Drift, or via a future compute helper)
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
| **Lemma** | Canonical key (from lemmatizer). Match key for highlight across wordforms (`run` / `running` / `ran`) |
| **Translation(s)** | One or more gloss strings the user keeps |
| **Context** | Optional source phrase and/or subtitle cue text at add time |
| **Ids / timestamps** | Stable entry id + created/updated in Drift |

Rules:

- Identity for highlight = **lemma**, not surface form.
- Surface form may be stored as display/context helper; do not use it as the
  membership key.
- Duplicate lemma policy = upsert + normalize (trim + case-fold) — see
  OpenSpec `dictionary/entries` / design D4.

## Operations (product CRUD + lookup)

| Op | Intent | Layer |
|----|--------|-------|
| **Add** | Persist lemma + translation(s) + optional context | **Drift** (Vocab UI now; Reader card later) |
| **List** | Return entries for Vocabulary screen | **Drift** |
| **Update** | Change translation(s) and/or context | **Drift** when UI supports edit |
| **Remove** | Delete entry; subsequent lookup misses → highlight clears | **Drift** |
| **Lookup by lemma** | Membership / entry fetch for Reader highlight | Drift on Flutter; NLP keys from Rust |

## Layering

```text
Flutter (vocab screen / word card)
  → Drift dictionary repository     # persist authority
  → (later) FRB tokenize/lemmatize / translate   # compute only
      → rust/src/api/…              # thin
          → tokenize_lemmatize / translate
```

| Layer | Path (target) | Role |
|-------|---------------|------|
| Persist | Flutter `lib/` Drift tables/DAOs | CRUD + reopen durability |
| FFI surface | `rust/src/api/` | Thin compute export only — **not** dict storage |
| NLP | `rust/src/tokenize_lemmatize/` (or peers) | Lemma keys for highlight / tap |
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
- [ ] Entry fields: lemma key, translation(s), optional context phrase/cue
- [ ] Lookup-by-lemma supports Reader highlight (same key as lemmatizer)
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
// → lemma key from Rust tokenize/lemmatize + Drift membership

// Fake HashMap “dict” left in api/simple forever
// → product vocab via Drift (+ NLP modules when compute lands)
```
