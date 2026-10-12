---
name: server-work-with-structure
description: >-
  Use when placing or moving Rust code in my_translator_app (Translation
  Reader): rust/src/api thin FRB surface vs internal parse / tokenize /
  lemmatize / translate (compute-only), frb_generated.rs, lib.rs module
  wiring, or deciding whether new logic belongs in api/ vs a domain module.
  User persistence is Drift in Flutter — not rust/src/db / rusqlite.
---

# Work With Structure

## Overview

Use this skill when creating or relocating code under `rust/` in
**my_translator_app** (product: **Translation Reader**).

Rust is the in-process **compute** core (parse, NLP, offline translate).
User persistence (library folders/documents, dictionary) is **Drift** in
Flutter — do **not** require `rust/src/db` / `rusqlite` for user data.
Flutter talks to Rust through **flutter_rust_bridge** (FRB): generated Dart →
thin `rust/src/api/` → internal modules.

Stack (from `{app}/AGENTS.md`): Rust edition `2021`, crate
`rust_lib_my_translator_app` (`cdylib` + `staticlib`), FRB `2.13.0`,
`flutter_rust_bridge.yaml` with `rust_input: crate::api`. User store = Drift.

Path package root: `{app}/rust/` (resolve `{app}` via
`docs/projects-map.md` key `app`).

Skills for this Rust layer live under `.agents/skills/server/` in
`my_translator_meta`. Sibling UI placement:
`client/client-work-with-structure`.

## Core Rules

1. Before adding modules, inspect a nearby peer in the **same layer** and
   match its file name and `mod` / `use` style. While the crate is still
   scaffold (`api/simple` only), follow the **target** layout in this skill,
   not the demo dump.
2. Create only the files the feature needs. Do not add empty stubs “for
   later”.
3. Place code by layer role: **thin FRB surface** (`api/`) vs **domain
   module** (parse / NLP / translate) vs **generated** (`frb_generated.rs`).
4. Respect **allowed dependency direction** (see below). Never invert
   layers.
5. Keep `api/` thin: FRB-friendly types, argument validation at the edge,
   and calls into internal modules. No SRT grammar or lemma tables inside
   `api/` beyond trivial glue. No user-SQL / rusqlite in `api/`.
6. Never hand-edit `rust/src/frb_generated.rs` or
   `lib/src/rust/**`. Change `rust/src/api/` → run FRB codegen.
7. Do not put business logic in `rust_builder/` (glue / cargokit only).
8. Do not reimplement parse / lemma / dictionary / translate rules in
   Flutter Dart — grow Rust modules and expose them through `api/`.

## Specialized Topics

Read the matching file in this folder when the change involves that area
(do not load every file at once):

| Topic | File |
|-------|------|
| Target module map (parse / NLP / dict / translate / db; owns vs not) | [content.md](content.md) |

## Folder Map (`rust/src`) — scaffold vs target

### Scaffold today

```text
rust/src/
├── lib.rs              # pub mod api; mod frb_generated;
├── api/
│   ├── mod.rs          # pub mod simple;
│   └── simple.rs       # process_text / add_numbers demo
└── frb_generated.rs    # GENERATED — do not edit
```

`lib.rs` does not declare parse / NLP / dict / translate / db yet.
Scaffold leftovers (`process_text` → `to_uppercase`, `add_numbers`) — do
not extend for product features; replace or leave unused as real APIs land.

### Target layout

| Layer | Path | Role |
|-------|------|------|
| Crate root | `rust/src/lib.rs` | `pub mod api;` + internal `mod`s + `mod frb_generated;` |
| FRB public surface | `rust/src/api/` | Thin `#[frb]` functions / types only; one file (or submodule) per product area |
| Parse | `rust/src/parse/` (or `parse.rs`) | SRT / VTT / ASS(SSA) → cues / timings |
| Tokenize / lemmatize | `rust/src/tokenize/` + `lemmatize/` | Word spans; EN WordNet morph keys + mark helpers |
| Translate | `rust/src/translate/` | Offline word/phrase translate; later optional AI adapters |
| Dictionary (NLP helpers only) | optional module later | Lemma-key helpers — **not** user CRUD store |
| User persistence | **Flutter Drift** (`lib/`) | folders / documents / dictionary — **not** `rust/src/db` |
| Generated | `rust/src/frb_generated.rs` | **GENERATED** — never hand-edit |
| Glue (outside src) | `rust_builder/` | cargokit / FFI build only |

Prefer a **directory + `mod.rs`** once a domain has more than one file;
a single `foo.rs` is fine for the first landing. Details of owns/deps →
[content.md](content.md).

**Out of crate:** Flutter `lib/` (UI + orchestration), generated Dart
`lib/src/rust/`, platform runners. Tests: Rust unit tests beside modules
or under `rust/` via `cargo test`; do not invent network `loadtest/`
trees for this offline core.

## Dependency Direction

Allowed:

```text
Flutter lib/**  (UI / orchestration + Drift user persist)
  → Drift DAOs / repositories
  → lib/src/rust/api/*          (generated Dart, compute)
      → rust/src/api/*          (thin FRB surface)
          → parse / tokenize_lemmatize / translate
```

Also normal:

- `api/` calls one or more internal modules and maps errors/types for FRB.
- Internal modules may call each other when the domain needs it — prefer
  keeping cycles out; see [content.md](content.md).
- `lib.rs` only declares modules; no business logic there.
- User CRUD does **not** go through FRB → `db`.

**Forbidden inversions:**

| Wrong | Why |
|-------|-----|
| Internal module → `api/` | Domain must not depend on FRB surface |
| `api/` → Flutter / Dart types | Rust owns the contract; Dart is generated |
| Hand-edit `frb_generated.rs` or `lib/src/rust/**` | Regenerate after `api/` changes |
| Business logic in `rust_builder/` | Glue only |
| Fat domain code inside `api/simple.rs` (or peers) | Extract to internal module; keep `api/` thin |
| Parse / lemma duplicated in Dart | Belongs in Rust compute |
| User SQLite / `rust/src/db` for vocab/library | Drift in Flutter |

## Where New Code Belongs

Decide in this order:

1. **New FFI entry the UI must call?** → thin function in `rust/src/api/`
   (new `api/<area>.rs` + `api/mod.rs` export), then FRB codegen.
2. **Subtitle formats / cue model?** → `parse/` (not `api/`, not Dart).
3. **Token spans?** → `tokenize/` (shipped). **Lemmas / word-form match?** →
   later `lemmatize/` (do not invent a combined `tokenize_lemmatize/` unless a
   change consolidates them).
4. **Vocab add/list/remove / lookup by lemma / context (persist)?** →
   **Flutter Drift** — not Rust `db/` / FRB storage. See
   `work-with-dictionary` + client `vocab.md`.
5. **Offline (or later AI) translate?** → `translate/` (+ thin `api/`
   export).
6. **User SQLite schema in Rust?** → **Do not** add for folders/documents/
   dictionary; persist = Drift unless a future change says otherwise.
7. **Init / default user utils?** → keep FRB `#[frb(init)]` in `api/`;
   do not open a user SQLite from Rust — not grow demo `simple.rs` forever.
8. **Pure helper used by one domain?** → same domain module; promote to
   a small shared `util` only when a second domain needs it (do not
   invent a grab-bag early).

### `api/` vs internal — what belongs where

**Put in `api/`**

- Public functions/types annotated for FRB.
- Edge validation (empty path, unsupported lang code) that maps cleanly
  to Flutter errors.
- Orchestration of a single user-facing compute call: parse file → return
  cues; lemmatize line → return tokens; translate word/phrase → gloss.

**Put in internal modules**

- Parsers, tokenizers, lemma tables/rules, translate backends.

**Put in Flutter Drift (not Rust)**

- User folders / documents / dictionary_entries persistence.

**Put in generated / glue**

- Nothing hand-written: `frb_generated.rs`, `lib/src/rust/**`,
  `rust_builder/` internals.

### What NOT to put

| Avoid | Prefer |
|-------|--------|
| SRT/VTT/ASS grammar inside `api/` | `parse/` + thin `api` wrapper |
| Lemma rules inside Flutter | `tokenize_lemmatize` + FRB |
| SQL in `api/` or Flutter | `db/` via dictionary/settings |
| Empty `parse/` + `dictionary/` stubs “for later” | Add on first real file |
| Growing `simple.rs` into the product API | New `api/<area>.rs` modules |
| Editing generated Dart/Rust FRB files | `api/` change → codegen |

## Naming

| Kind | Convention | Examples |
|------|------------|----------|
| Modules / files | `snake_case` | `parse/mod.rs`, `dictionary/entry.rs` |
| FRB api files | `snake_case` area | `api/parse.rs`, `api/dictionary.rs`, `api/simple.rs` (scaffold) |
| Public Rust fns | `snake_case` (FRB maps to camelCase Dart) | `process_text` → `processText` |
| Types | `PascalCase` | `SubtitleCue`, `DictionaryEntry`, `TokenSpan` |
| Tests | `*_test` modules or `#[cfg(test)]` beside code | `parse::tests`, `cargo test` |
| Crate | existing name | `rust_lib_my_translator_app` |

Match peers once they exist; prefer AGENTS entity names (Subtitle document,
Token/Lemma, Dictionary entry, Reader session, Settings) for public types.

## Typical Shapes

### Crate root

```text
rust/src/lib.rs
  pub mod api;
  mod parse;
  mod tokenize_lemmatize;  # or tokenize + lemmatize
  mod dictionary;
  mod translate;
  mod db;
  mod frb_generated;
```

### Thin API + domain

```text
rust/src/api/
├── mod.rs           # pub mod parse; pub mod dictionary; …
├── parse.rs         # frb: parse_subtitles(bytes|path) -> cues
└── dictionary.rs    # frb: add_entry / list_entries / …

rust/src/parse/
├── mod.rs
├── srt.rs
└── vtt.rs
```

### Generated (do not edit)

```text
rust/src/frb_generated.rs
lib/src/rust/api/….dart
lib/src/rust/frb_generated.dart
```

## Real Composition Examples (target)

**Parse upload** — Flutter picks file bytes/path → `api::parse` →
`parse::` builds cues → returns structured lines/timings. No subtitle
grammar in Dart or in fat `api` bodies.

**Reader tap** — UI sends line text + language → `api` →
`tokenize_lemmatize` → optional `translate` / `dictionary` lookup →
token spans + lemma + gloss for the word card.

**Highlight** — Rust prepares lemma keys / spans; Flutter only paints.
Matching `run` / `running` / `ran` lives in NLP + dictionary, not Dart.

**Vocab CRUD** — Drift repositories in Flutter (`dictionary/entries`).
Rust may later supply lemma keys / gloss for Reader add; **persist is not
FRB**. Do not add `api::dictionary` + `rusqlite` for user rows without an
explicit OpenSpec change.

**Offline translate** — `api::translate` → `translate` module (local
data/rules). Optional AI later stays behind the same module boundary.

**Init** — `#[frb(init)]` in `api/` for FRB runtime only; Drift open lives
in Flutter `main` / app bootstrap.

## Creating New Pieces — Checklist

**New FRB endpoint**

1. Implement domain logic under the right internal module.
2. Add a thin wrapper in `rust/src/api/<area>.rs`; export from
   `api/mod.rs`.
3. Run `flutter_rust_bridge_codegen generate`.
4. Update Flutter orchestration + tests; never hand-edit generated
   Dart.

**New domain module**

1. Add `mod <name>;` in `lib.rs` (not `pub` unless another crate needs
   it — usually private; `api` is the public FFI surface).
2. Keep the first feature’s files only — no empty sibling folders.
3. Call from `api/`, not from Flutter.

**New user persistence**

1. Implement in Flutter Drift (`lib/` data layer) — see client structure /
   OpenSpec `library/*` / `dictionary/entries`.
2. Do **not** create `rust/src/db` or add `rusqlite` for user CRUD.

**Retiring scaffold demo**

1. Stop routing product work through `process_text` / `add_numbers`.
2. Remove or leave demo APIs only when codegen/UI/tests are updated in
   the same change.

## Domain Anchors

| Domain | Primary paths (target) | Authority |
|--------|------------------------|-----------|
| FRB surface | `rust/src/api/` | Thin FFI only; source of truth for compute contract |
| Parse | `rust/src/parse/` | Cue/timing model |
| Tokenize / lemma | `tokenize_lemmatize/` (or split) | Tap units + highlight keys |
| Dictionary persist | Flutter Drift | Vocab rows + context |
| Translate | `translate/` | Offline (later AI) glosses |
| Library persist | Flutter Drift | folders / documents |
| Generated bridge | `frb_generated.rs` + `lib/src/rust/` | Tooling only |
| Flutter UI | `lib/features/…` | Display / gestures — see client structure skill |

User entities (library, dictionary) — **Drift** truth. Compute entities
(tokens, cues, gloss) — Rust when APIs land.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Growing `api/simple.rs` into product logic | New `api/<area>.rs` + internal module |
| Parsing or lemmatizing in Dart | Rust module + thin `api/` + codegen |
| Hand-editing generated FRB files | Edit `api/` → codegen |
| Internal `use crate::api::…` | Depend on domain modules only |
| Empty domain folders “for later” | Create on first real file |
| Putting user SQL / rusqlite in Rust | Drift in Flutter `lib/` |
| Business logic in `rust_builder/` | Leave as glue |
| Fat `api` that reimplements parsers | Move body to `parse/` / NLP / etc. |

## Related Skills

- Module map detail → [content.md](content.md)
- Parse domain → `work-with-parse`
- Tokenize / lemmatize → `work-with-tokenizematize`
- Dictionary → `work-with-dictionary`
- Translate → `work-with-translate`
- FRB API surface / codegen contract → `work-with-frb-api`
- Config / paths / init helpers → `work-with-config`
- Flutter `lib/` placement → `client/client-work-with-structure`
- Flutter FRB call sites / `RustLib.init` → `client/work-with-frb`
- Rust/API implementability of meta claims → `align-implementability-back`
- App AGENTS canon → `{app}/AGENTS.md` via `docs/projects-map.md`

## Verification

For structure-only placement tasks, confirm:

- [ ] Correct layer (`api/` thin vs internal domain vs generated)
- [ ] Dependency direction: Flutter → generated Dart → `api/` → internal
- [ ] No hand-edits under `frb_generated.rs` or `lib/src/rust/`
- [ ] No business logic in `rust_builder/`
- [ ] Scaffold `simple` demo not extended as the product API
- [ ] New modules added only when first real code lands
- [ ] After `api/` changes: codegen + Flutter callers/tests updated
- [ ] Heavy logic not duplicated in Dart
