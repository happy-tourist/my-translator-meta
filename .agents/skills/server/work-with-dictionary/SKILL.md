---
name: work-with-dictionary
description: >-
  Personal vocabulary CRUD in Rust for Translation Reader (my_translator_app):
  add / list / update / remove entries keyed by lemma, store translations and
  optional context phrase/cue, lookup-by-lemma for reader highlight. Use when
  implementing or changing dictionary domain, SQLite persistence, or thin FRB
  dictionary API under rust/src/api/. Not Flutter vocab UI chrome (work-with-pages
  vocab.md) and not cloud sync.
---

# Work With Dictionary

Use this skill for the **personal vocabulary (dictionary)** in
**my_translator_app** (product: **Translation Reader**) — Rust core + local
SQLite target. Flutter only lists/adds via FRB; **Rust owns truth**.

**Paths:** this skill lives in **my_translator_meta** at
`.agents/skills/server/work-with-dictionary/`. Runtime paths (`rust/src/…`)
are relative to the **my_translator_app** sibling (`../my_translator_app`
from meta; see `docs/projects-map.md`). Read `{app}/AGENTS.md` before changing
the crate.

Stack: **Rust** edition `2021`, crate `rust_lib_my_translator_app`, public FFI
in `rust/src/api/`, FRB `2.13.0` (in-process).

## Scaffold Honesty

**Today there is no dictionary module and no SQLite.** Scaffold only exposes
`process_text` / `add_numbers` in `rust/src/api/simple.rs`. Treat everything
below as the **target contract** to implement when an OpenSpec change asks for
it — do **not** pretend entries already persist, and do **not** grow the
demo `simple` API into a fake in-memory “dict” without a real domain module.

## When to Use

- Add / list / update / remove personal vocabulary entries
- Lemma as the stable key; translations + optional context (phrase / cue text)
- Lookup-by-lemma (membership / entry) so Reader can highlight saved words
- Wire thin FRB exports for dictionary after domain logic exists
- Introduce SQLite persistence for dictionary rows (when persistence lands)

**Not this skill:** Flutter vocab list chrome alone → `work-with-pages`
(`vocab.md`); tokenize/lemmatize algorithms → `work-with-tokenizematize`;
offline gloss generation → `work-with-translate`; FRB codegen mechanics →
`work-with-frb-api` / client `work-with-frb`; multi-device or cloud sync
(out of scope in `{app}/AGENTS.md`).

## Entry Model (target)

| Field | Role |
|-------|------|
| **Lemma** | Canonical key (from lemmatizer). Match key for highlight across wordforms (`run` / `running` / `ran`) |
| **Translation(s)** | One or more gloss strings the user keeps |
| **Context** | Optional source phrase and/or subtitle cue text at add time |
| **Ids / timestamps** | Stable entry id + created/updated when persistence exists |

Rules:

- Identity for highlight = **lemma**, not surface form.
- Surface form may be stored as display/context helper; do not use it as the
  membership key.
- Duplicate lemma policy must be explicit when implementing (reject vs upsert
  vs multi-sense rows) — pick one in OpenSpec/design; do not invent silently
  in Flutter.

## Operations (target CRUD + lookup)

| Op | Intent |
|----|--------|
| **Add** | Persist lemma + translation(s) + optional context (from Reader word card) |
| **List** | Return entries for Vocabulary screen (lemma, translation, context) |
| **Update** | Change translation(s) and/or context / notes when API supports edit |
| **Remove** | Delete entry; subsequent lemma lookup misses → highlight clears |
| **Lookup by lemma** | Membership / entry fetch for Reader highlight flags |

Add is primarily triggered from Reader word card; Vocabulary screen is
management/review (list / remove / optional update) — see client
`work-with-pages` / `vocab.md`.

## Layering

```text
Flutter (vocab screen / word card)
  → generated FRB (lib/src/rust/api/…)
      → rust/src/api/…          # thin: types + call domain
          → dictionary module   # rules + persistence
              → SQLite (target)
```

| Layer | Path (target) | Role |
|-------|---------------|------|
| FFI surface | `rust/src/api/` (e.g. `dictionary.rs`) | Thin export only — map args/results |
| Domain | `rust/src/dictionary/` (or `dict/`) | CRUD rules, lemma key, context attach |
| Persistence | same domain + db helper | SQLite on device |
| Generated | `frb_generated.rs`, `lib/src/rust/**` | **Never** hand-edit |

Heavy logic stays **outside** `api/`. Flutter must not keep a second source of
truth in `SharedPreferences` when SQLite/Rust owns the dictionary.

Structural pattern: keep the FRB `api/` thin; put create/ensure/migrate and
domain rules in a dictionary/db module; document the on-device SQLite path;
never ship/commit the live DB file as source.

## Persistence (target — developer decision)

- **Target store:** SQLite on the device, accessed **from Rust**.
- **Crate not chosen yet:** `diesel` **or** `rusqlite` (or equivalent) —
  **developer decision** before implementation; do not silently pick one in a
  drive-by change. Record the choice in OpenSpec/design when adopting.
- Open / migrate / path resolution live in Rust (app data dir via Flutter→Rust
  path handoff or Rust-side platform path — decide in design; keep one path).
- Use temp / in-memory DB in tests — never the developer’s real vocab file.

## Offline / Sync Scope

- Dictionary is **local-only** and works **offline**.
- **No cloud sync**, no multi-device merge, no shared decks — out of scope
  (`{app}/AGENTS.md`). Do not add sync endpoints or accounts “for later”
  without an explicit change.

## Related Skills (by name)

| Skill | Concern |
|-------|---------|
| `work-with-tokenizematize` | Lemma/token production that feeds the dict key + highlight |
| `work-with-translate` | Offline gloss for word/phrase card before/when adding |
| `work-with-frb-api` | Thin public `api/` surface + FRB export shape |
| `server-work-with-errors` | Rust error kinds / Result mapping for dict ops |
| `server-work-with-test` | `cargo test` for CRUD / lookup / SQLite fixtures |

Client UI: `work-with-pages` (`vocab.md`), `work-with-reader` (add + highlight),
`work-with-frb` (codegen / Dart call sites).

## Do

- Put rules and persistence in a dictionary domain module; keep `api/` thin.
- Key membership and highlight on **lemma** from the lemmatizer pipeline.
- Persist optional context (phrase / cue) with the entry when provided at add.
- Cover add → list → lookup → remove (and update if shipped) with
  `cargo test` → `server-work-with-test`.
- After public API change: edit `rust/src/api/` → codegen → Dart call sites
  (`work-with-frb-api` / client `work-with-frb`).
- Map failures to stable Rust error kinds → `server-work-with-errors`.

## Don't

- Reimplement dictionary rules or lemma match in Dart.
- Hand-edit `rust/src/frb_generated.rs` or `lib/src/rust/**`.
- Stuff CRUD into `api/simple.rs` or grow scaffold demo into fake persistence.
- Choose `diesel` vs `rusqlite` without an explicit developer/OpenSpec decision.
- Add cloud sync, accounts, or shared decks.
- Commit device DB files or secrets.

## Checklist

When implementing or changing dictionary:

- [ ] Domain module owns CRUD + lemma lookup; `api/` only exports
- [ ] Entry fields: lemma key, translation(s), optional context phrase/cue
- [ ] Lookup-by-lemma supports Reader highlight (same key as lemmatizer)
- [ ] SQLite path/crate choice documented if persistence is in scope
- [ ] Offline local-only; no sync surface
- [ ] Errors via `server-work-with-errors`; tests via `server-work-with-test`
- [ ] FRB codegen after `api/` change; no hand-edits to generated files
- [ ] Scaffold honesty: no claims that dict/SQLite already exists unless code does

## Anti-Patterns

```text
// Dart SharedPreferences as second dictionary truth
// → Rust + SQLite is authority when persistence lands

// Highlight by raw string equality in Flutter
// → lemma key from Rust tokenize/lemmatize + dict lookup

// api/ function with SQL inline and no domain module
// → thin api/ → dictionary module → db

// Fake HashMap “dict” left in api/simple forever
// → real module + persistence when the change asks for product vocab

// “We’ll sync later” tables / user_id columns without a sync change
// → out of scope
```
