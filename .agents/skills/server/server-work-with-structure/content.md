# Rust module map (`rust/src` target)

Read with the [core structure skill](SKILL.md). Domain depth lives in sibling
skills (`work-with-parse`, `work-with-tokenizematize`,
`work-with-dictionary`, `work-with-translate`, `work-with-frb-api`,
`work-with-config`). This file is **placement only**: what each module owns,
how to grow it, and allowed internal edges.

**User persistence** (folders / documents / dictionary) is **Drift in Flutter**
— not `rust/src/db` / `rusqlite`. See `{app}/AGENTS.md` and
`work-with-dictionary`.

## Scaffold vs target (reminder)

| Today | Target |
|-------|--------|
| `lib.rs` → `api` + `frb_generated` | Same + private `mod` for each compute domain below |
| `api/simple.rs` demo | Thin `api/<area>.rs` per compute surface |
| No parse / NLP / translate | Add modules on first real feature file |
| No user SQLite in Rust | Keep it that way — Drift owns user rows |

Do not pre-create empty module trees. Do not invent `db/` for user CRUD.

## Module table

| Module | Suggested path | Owns | Does not own |
|--------|----------------|------|--------------|
| **api** | `rust/src/api/` | FRB-exported fns/types; edge checks; one-shot orchestration into internals | Parsers, lemma tables, Flutter widgets, user SQL |
| **parse** | `rust/src/parse/` | SRT / VTT / ASS(SSA) → cues, timings, plain text lines | Tokenization, dictionary persist, UI layout |
| **tokenize** (shipped) | `tokenize/` + `api/tokenize.rs` | Word tokens + char spans for Reader tap | File format parsing, Drift/user store |
| **lemmatize** (later) | `lemmatize/` (or extend tokenize) | Morph keys for highlight / vocab | Do not invent FRB lemma fields until an OpenSpec change |
| **translate** | `rust/src/translate/` | Offline gloss for word/phrase; later provider adapters behind one façade | Cue parsing, vocab row persistence |
| **dictionary** (optional later) | helpers only if needed | Lemma-key / membership helpers for highlight | User CRUD store (→ Drift), raw SQL |
| **frb_generated** | `rust/src/frb_generated.rs` | Tooling output | Any hand edits |
| **config** (when needed) | helper module under init | Feature flags for compute crates | Flutter theme / ARB strings / user DB paths |

## Growing a module

1. **First file** — single `name.rs` + `mod name;` in `lib.rs`, or
   `name/mod.rs` if you already know you need subfiles.
2. **Second concern in same domain** — switch to `name/` directory; split
   by format or concern (`parse/srt.rs`, `parse/vtt.rs`).
3. **Public to Flutter?** — never `pub use` domain types straight into
   Dart. Add or extend `api/<area>.rs`, then codegen.
4. **Tests** — `#[cfg(test)]` in the domain module; run `cargo test` in
   `rust/`. Prefer pure fixtures (byte slices) over Flutter.

### Split or combine tokenize + lemmatize?

- **Current:** split — `tokenize/` shipped (panko); lemmatize not landed.
- **Combine** later only if the pipeline is always “text → tokens with lemmas”
  and one module is clearer than two.
- **Keep split** when tokenize and lemmatize gain separate backends, languages,
  or large files — keep a small façade so `api/` still calls one entry.

Stick to `tokenize/` (+ later `lemmatize/`) unless a change consolidates them.

## Internal dependency edges

Allowed (prefer acyclic):

```text
api
  → parse | tokenize | (later lemmatize / translate)
tokenize
  → (pure; no api/; wraps panko)
parse
  → (no translate required)
translate / lemmatize
  → (later; optional caches — not user vocab Drift)
```

Avoid:

- `api/` → Flutter / Drift types.
- `parse` → Flutter Drift (loading a file for Reader cues is compute, not
  library import — library import is Drift).
- Any internal module → `api` or `frb_generated`.
- Introducing `db` + `rusqlite` for user dictionary/library without an
  OpenSpec change that explicitly reassigns persist.

## FRB surface map (target areas)

| Area | Typical `api/` file | Calls into |
|------|---------------------|------------|
| parse | `api/parse.rs` | `parse/` |
| tokenize | `api/tokenize.rs` | `tokenize/` |
| translate | `api/translate.rs` | `translate/` |
| simple (scaffold) | `api/simple.rs` | demo only — retire for product |

Dictionary / library **storage** endpoints are **not** required on FRB;
persist via Drift. See client `work-with-pages` (`upload.md`, `vocab.md`).

Contract changes for compute: edit `api/` →
`flutter_rust_bridge_codegen generate` → update Dart orchestration. See
`work-with-frb-api` and `client/work-with-frb`.

## Owns / tree comments (quick)

**parse** — format detection; cue list with start/end/text; error types for
malformed files; no UI string tables.

**tokenize_lemmatize** — language-aware split; lemma for each token;
stable ids for highlight paint; morphology match for vocab keys.

**translate** — offline lookup first; optional AI later as adapter behind
same API so Flutter does not branch on providers.

**api** — mirror the compute table above; keep functions short; map domain
errors to FRB-friendly messages/codes (error-handling skill when present).

**User store** — Drift tables/DAOs in Flutter `lib/` (not this crate).

## Related skills

| Concern | Skill |
|---------|-------|
| Placement rules | [SKILL.md](SKILL.md) |
| Parsers | `work-with-parse` |
| Tokens / lemmas | `work-with-tokenizematize` |
| Vocab domain / Drift boundary | `work-with-dictionary` |
| Gloss / offline / AI façade | `work-with-translate` |
| FRB exports / codegen | `work-with-frb-api` |
| Paths / init / flags | `work-with-config` |
