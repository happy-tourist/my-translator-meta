# Rust module map (`rust/src` target)

Read with the [core structure skill](SKILL.md). Domain depth lives in sibling
skills (`work-with-parse`, `work-with-tokenizematize`,
`work-with-dictionary`, `work-with-translate`, `work-with-frb-api`,
`work-with-config`). This file is **placement only**: what each module owns,
how to grow it, and allowed internal edges.

## Scaffold vs target (reminder)

| Today | Target |
|-------|--------|
| `lib.rs` → `api` + `frb_generated` | Same + private `mod` for each domain below |
| `api/simple.rs` demo | Thin `api/<area>.rs` per product surface |
| No parse / NLP / dict / translate / db | Add modules on first real feature file |

Do not pre-create empty module trees.

## Module table

| Module | Suggested path | Owns | Does not own |
|--------|----------------|------|--------------|
| **api** | `rust/src/api/` | FRB-exported fns/types; edge checks; one-shot orchestration into internals | Parsers, lemma tables, SQL bodies, Flutter widgets |
| **parse** | `rust/src/parse/` | SRT / VTT / ASS(SSA) → cues, timings, plain text lines | Tokenization, dictionary, UI layout |
| **tokenize / lemmatize** | `tokenize_lemmatize/` **or** `tokenize/` + `lemmatize/` | Split text into tap units; lemma ids for highlight / vocab keys | File format parsing, SQLite, FRB attributes |
| **dictionary** | `rust/src/dictionary/` | Entry model, add/list/remove, lookup by lemma, context phrase rules | Raw SQL dialects (call `db`), subtitle file IO |
| **translate** | `rust/src/translate/` | Offline gloss for word/phrase; later provider adapters behind one façade | Cue parsing, vocab persistence invariants |
| **db** | `rust/src/db/` | Open DB, migrations, queries, paths on device | Product “should this lemma be saved?” rules (dictionary owns those) |
| **frb_generated** | `rust/src/frb_generated.rs` | Tooling output | Any hand edits |
| **config** (when needed) | helper module or under `db`/init | Data dir paths, feature flags for core | Flutter theme / ARB strings |

## Growing a module

1. **First file** — single `name.rs` + `mod name;` in `lib.rs`, or
   `name/mod.rs` if you already know you need subfiles.
2. **Second concern in same domain** — switch to `name/` directory; split
   by format or concern (`parse/srt.rs`, `parse/vtt.rs`).
3. **Public to Flutter?** — never `pub use` domain types straight into
   Dart. Add or extend `api/<area>.rs`, then codegen.
4. **Tests** — `#[cfg(test)]` in the domain module; run `cargo test` in
   `rust/`. Prefer pure fixtures (byte slices, in-memory DB) over Flutter.

### Split or combine tokenize + lemmatize?

- **Combine** (`tokenize_lemmatize`) when the pipeline is always “text →
  tokens with lemmas” and one skill/`work-with-tokenizematize` covers both.
- **Split** when tokenize and lemmatize gain separate backends, languages,
  or large files — keep a small façade so `api/` still calls one entry.

Either shape is fine; pick one when the first NLP code lands and stick to
it until a real split reason appears.

## Internal dependency edges

Allowed (prefer acyclic):

```text
api
  → parse | tokenize_lemmatize | dictionary | translate | db | (config)
dictionary
  → tokenize_lemmatize   # lemma keys / normalize surface forms
  → db                   # persist entries
  → translate            # optional: suggest gloss on add
translate
  → db                   # optional caches / installed packs
parse
  → (no dict/translate required)
tokenize_lemmatize
  → (pure; no db/api)
db
  → (sqlite crate only; no api/)
```

Avoid:

- `db` → `dictionary` for business rules (keep rules in dictionary).
- `parse` → `dictionary` (loading a file is not vocab mutation).
- Any internal module → `api` or `frb_generated`.

## FRB surface map (target areas)

Align new `api/*.rs` files with product calls (names illustrative):

| Flutter need | `api` module | Calls into |
|--------------|--------------|------------|
| Open / parse subtitle file | `api/parse` | `parse` |
| Tokens + lemmas for a line | `api/tokenize` or under parse/reader api | `tokenize_lemmatize` |
| Word/phrase card gloss | `api/translate` | `translate` (+ lemma) |
| Vocab CRUD / list | `api/dictionary` | `dictionary` → `db` |
| Settings / languages | `api/settings` or config export | `db` / config |
| Demo only | `api/simple` | none — do not grow |

Contract changes: edit `api/` → `flutter_rust_bridge_codegen generate` →
update Dart orchestration. See `work-with-frb-api` and
`client/work-with-frb`.

## Owns / tree comments (quick)

**parse** — format detection; cue list with start/end/text; error types for
malformed files; no UI string tables.

**tokenize_lemmatize** — language-aware split; lemma for each token;
stable ids for highlight paint; morphology match for vocab.

**dictionary** — entry identity by lemma (+ language pair as product
defines); context snippet from cue/phrase; list/filter; delete.

**translate** — offline lookup first; optional AI later as adapter behind
same API so Flutter does not branch on providers.

**db** — single place for schema version + migrations; dictionary and
settings are clients.

**api** — mirror the table above; keep functions short; map domain errors
to FRB-friendly messages/codes (error-handling skill when present).

## Related skills

| Concern | Skill |
|---------|-------|
| Placement rules | [SKILL.md](SKILL.md) |
| Parsers | `work-with-parse` |
| Tokens / lemmas | `work-with-tokenizematize` |
| Vocab domain | `work-with-dictionary` |
| Gloss / offline / AI façade | `work-with-translate` |
| FRB exports / codegen | `work-with-frb-api` |
| Paths / init / flags | `work-with-config` |
