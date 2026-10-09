---
name: server-locate-change-points
description: >-
  Use when the user asks to analyze a task, locate implementation points, find
  affected files, or identify where Rust `rust/` changes should be made without
  editing code in Translation Reader (`my_translator_app`). Covers FFI surface
  (`rust/src/api/`), internal modules (parse, tokenize/lemmatize, dictionary,
  translate, db), `Cargo.toml`, and Rust tests; flags when the real fix belongs
  in Flutter `lib/` and points to `client-locate-change-points`.
---

# Locate Change Points

Use this skill when the user gives a task statement and wants only analysis of
where changes are needed in the Rust core layer of **Translation Reader**
(`my_translator_app`).

Stack: Rust edition `2021`, crate `rust_lib_my_translator_app` (`cdylib` +
`staticlib`), `flutter_rust_bridge` 2.13 (public API under `rust/src/api/`,
generated glue in `rust/src/frb_generated.rs` and Flutter `lib/src/rust/`).
Same repo as Flutter UI — in-process FRB. Resolve app path via
[`docs/projects-map.md`](../../../../docs/projects-map.md) →
`../my_translator_app`. Skills for this layer live under
`.agents/skills/server/` in `my_translator_meta`.

**Not a Flutter skill.** Point to `lib/` when the change is UI/chrome/gestures;
do not invent Flutter screen trees or implement Dart here. For Flutter-only
placement, hand off to
[`client-locate-change-points`](../../client/client-locate-change-points/SKILL.md).

## Hard Rule

Do NOT edit, create, delete, format, or commit files.

Only inspect the codebase and report:

- existing files that likely need changes;
- exact FFI modules, internal crates modules, `Cargo.toml` deps, tests, or
  related Flutter/FRB touch points;
- places where new files should be added, if the task requires new code;
- open questions when the implementation location cannot be determined confidently.

## Rust Layer Map

Composition under `rust/`. Search and assign ownership top-down. Today the crate
is mostly **scaffold**: demo API in `rust/src/api/simple.rs`; product modules
(parse / NLP / dict / translate / db) are **target** layout — prefer them for
new work, not growing `process_text` forever.

| Layer | Path | Role |
|-------|------|------|
| Crate root | `rust/src/lib.rs` | Declares `api` + `frb_generated`; add `mod` for new internal modules |
| FFI surface | `rust/src/api/**/*.rs` | Public FRB API (source of truth); thin exports only |
| API module list | `rust/src/api/mod.rs` | `pub mod …` for each API file |
| Demo API (scaffold) | `rust/src/api/simple.rs` | `process_text`, `add_numbers`, `#[frb(init)] init_app` |
| Internal modules (target) | `rust/src/{parse,tokenize,lemma,translate,…}/` or sibling `.rs` | Heavy **compute**; not FRB-exported unless re-exported via `api/` |
| Generated Rust FRB | `rust/src/frb_generated.rs` | **Do not hand-edit** |
| Crate manifest | `rust/Cargo.toml` | Deps (`flutter_rust_bridge`, later parsers/NLP — **not** user `rusqlite`) |
| FRB config (app root) | `flutter_rust_bridge.yaml` | `rust_input` / `rust_root` / `dart_output` — mention only; details → work-with-config / work-with-frb-api (and Flutter bridge → `client/work-with-frb`) |
| Glue | `rust_builder/` | cargokit/FFI only — not product logic |
| Rust tests | `rust/` (`cargo test`; unit tests next to modules or `tests/`) | Parser / lemma / dict / translate coverage |
| Generated Dart (read-only) | `lib/src/rust/**` | Regenerated after API change; never hand-edit |

Allowed dependency direction:

```text
lib/** (UI)  →  lib/src/rust/api/* (generated)  →  rust/src/api/*  →  rust core modules
```

Rust owns parse / tokenize / lemmatize / offline translate (**compute**).
User library/dictionary persistence is **Drift in Flutter `lib/`** — do not
point agents at `rusqlite` / `rust/src/db` for user rows. Do not recommend
reimplementing NLP/parse in Dart.

### Current scaffold (confirmed)

| Item | Location |
|------|----------|
| Crate | `rust_lib_my_translator_app` in `rust/Cargo.toml` |
| Modules | `lib.rs` → `api` + `frb_generated`; `api/mod.rs` → `simple` only |
| Demo API | `process_text` → `to_uppercase`; `add_numbers`; `init_app` → `setup_default_user_utils()` |
| Product modules | **Not in tree yet** (parse, tokenize/lemmatize, translate) |
| User SQLite in Rust | **Not canon** — Drift owns folders/documents/dictionary |
| Extra Cargo deps beyond FRB | None yet |

### Target domain modules (product, mostly not in tree yet)

| Concern | Prefer internal module | Thin FFI in `api/` |
|---------|------------------------|--------------------|
| Parse SRT / VTT / ASS | `parse` (or similar) | cues / document DTO export |
| Tokenize / lemmatize | `tokenize` / `lemma` | tokens + lemma keys for UI |
| Dictionary lemma keys / NLP handoff (later) | thin compute helpers if needed | **persist CRUD → Flutter Drift**, not Rust db |
| Offline translate word/phrase | `translate` | result DTO for word/phrase card |
| Settings / local persistence | — | **Flutter Drift** (settings later) |
| Optional AI (later) | Explicit later; do not invent providers from this skill | — |
| Init / bridge utils | keep `#[frb(init)]` in `api/` | `init_app` only |

## Decision Guidance

Use these rules to pick the layer before naming files.

### FFI surface vs internal module

| If the change is… | Prefer |
|-------------------|--------|
| New / changed method or DTO visible to Flutter | `rust/src/api/**/*.rs` (+ `api/mod.rs`); then **codegen required** |
| Heavy parse / NLP / translate logic | new or existing module **outside** `api/`; `api/` only re-exports / wraps |
| User folders / documents / dictionary persist | **Flutter Drift** — redirect to [`client-locate-change-points`](../../client/client-locate-change-points/SKILL.md) |
| Grow scaffold demo (`to_uppercase` / `add_numbers`) for a product feature | **No** — add target modules + thin `api/`; leave demo alone unless the task is explicitly demo-only |
| `#[frb(init)]` / default user utils | `rust/src/api/simple.rs` (`init_app`) or a dedicated init API module when the surface grows |
| Hand-edit `rust/src/frb_generated.rs` or `lib/src/rust/**` | Never — report as wrong target; regenerate after `api/` change |
| New crate dependency | `rust/Cargo.toml` (+ use from internal module) |
| FRB yaml paths / codegen knobs | Note `flutter_rust_bridge.yaml` under Related; defer how-to to **work-with-config** / **work-with-frb-api** (Flutter call sites → `client/work-with-frb`) |
| `rust_builder/` | Only if build/glue breakage; not for features |
| Crate module wiring | `rust/src/lib.rs` (`mod` / `pub mod`) when adding internal modules |

### When the fix belongs in Flutter instead

| If the change is… | Action |
|-------------------|--------|
| Screens, gestures, theme, i18n, layout, scroll sync UI | Flag: **Flutter `lib/`** — point to [`client-locate-change-points`](../../client/client-locate-change-points/SKILL.md); list under Related only |
| Bind existing generated API in a widget | Flutter call site; this skill only confirms the Rust API already exists (or must be added first) |
| Map Rust errors to SnackBar / banners | Flutter; Related: error shape in `rust/src/api/` if missing |
| Pure chrome / Visual Spec sizes/colors | Flutter + OpenSpec Visual Spec; not Rust |

### After API change → codegen

Any edit to the **public** surface in `rust/src/api/` implies:

```text
1. Edit rust/src/api/… (+ api/mod.rs)
2. Keep heavy logic in rust modules outside api/
3. flutter_rust_bridge_codegen generate  (from app root; CLI ≈ 2.13.0)
4. Update Flutter call sites / tests against generated Dart
5. cargo check|test in rust/; flutter analyze / flutter test as needed
```

Always list codegen + affected `lib/` call sites under **Related places**. Do not
implement them from this skill.

### Tests and config

- Unit tests for parse / lemma / dict / translate → next to modules or `rust/tests/`; `cargo test` in `rust/`.
- FFI contract smoke may also need Flutter `integration_test/` / widget tests — Related only; Flutter placement → client skill.
- `Cargo.toml` — new crates (subtitle parsers, NLP, etc.) when the task needs them; do **not** add user-store `rusqlite`/`diesel`; do not invent versions here beyond noting the need.
- `flutter_rust_bridge.yaml` — rarely for features; only if input/output paths or crate API root change.

## Domain Hotspots

| Domain | Start here |
|--------|------------|
| Scaffold demo / FRB hello | `rust/src/api/simple.rs` (+ generated Dart under Related) |
| Bridge init | `init_app` in `api/simple.rs`; Flutter `RustLib.init()` under Related |
| Subtitle parse (target) | new `rust/src/…` parse module + thin `api/` export |
| Tokenize / lemmatize / highlight keys (target) | NLP modules + `api/` DTOs for tokens/lemmas |
| Dictionary persist CRUD | **Flutter Drift** (`lib/data/`) — not Rust |
| Offline translate (target) | translate module + `api/` |
| Settings persistence (target) | **Flutter Drift** (later) |
| Crate deps | `rust/Cargo.toml` |
| Generated Rust bindings | `rust/src/frb_generated.rs` (read-only; regen after API change) |
| Rust tests | colocated / `cargo test` in `rust/` |
| Flutter UI for the same feature | Related → `client-locate-change-points` |

## Workflow

1. Read the task statement and extract:
   - target feature or behavior;
   - entities (subtitle document, token/lemma, dictionary entry, reader session, settings);
   - whether the task changes existing scaffold API or adds a product flow;
   - whether the **FRB contract** (new method, DTO, error shape) is involved vs pure internal logic;
   - whether UI-only work should be redirected to the client skill.

2. Search by domain terms in `../my_translator_app/rust` (and app root for yaml/Cargo):
   - API symbols (`process_text`, `add_numbers`, `init_app`, `api::`);
   - module paths (`api/simple`, `frb_generated`, future `parse` / `lemma` / `dictionary` / `translate` / `db`);
   - product terms (`subtitle`, `srt`, `vtt`, `lemma`, `dictionary`, `tokenize`, `translate`) — expect many hits only in `AGENTS.md` until implemented;
   - `Cargo.toml` deps;
   - tests under `rust/`.

3. Walk ownership:
   - internal module → thin `api/` export → (Related) codegen + Flutter call sites;
   - flag any proposed hand-edit of `frb_generated.rs` or `lib/src/rust/**` as invalid;
   - flag Dart reimplementation of parse/NLP/dict as a smell → keep in Rust;
   - if the task is screens/gestures/theme only → stop and point to `client-locate-change-points`.

4. Apply decision guidance above to classify each hit as edit vs add, and note Related Flutter/codegen touch points (do not implement them from this skill).

5. Stop after identifying change points. Do not implement.

## Search Tips

| Look for… | Where / how |
|-----------|-------------|
| Public FFI functions | `rust/src/api/**/*.rs` |
| API module registration | `rust/src/api/mod.rs` |
| Crate modules | `rust/src/lib.rs` |
| Demo scaffold | `process_text`, `add_numbers`, `init_app` in `api/simple.rs` |
| Generated glue (read-only) | `rust/src/frb_generated.rs`, `lib/src/rust/**` |
| Deps | `rust/Cargo.toml` |
| FRB paths | `flutter_rust_bridge.yaml` at app root |
| Flutter call sites (Related) | `lib/main.dart`, `package:my_translator_app/src/rust/api/…` |
| Product terms not in tree | `{app}/AGENTS.md` target sections — recommend **Where to add** |

## Output Format

Respond in this structure:

```markdown
## Where to edit
- `path/to/file.rs` — why this file is relevant and what kind of change is expected.

## Where to add
- `path/to/new_file.rs` — why a new file belongs here.

## Related places
- `path/to/related` — why it should be checked during implementation (incl. codegen after `rust/src/api/` change, and `lib/` call sites / `client-locate-change-points` when UI is involved).

## Questions
- Clarifying question, only if needed.
```

Omit empty sections. Keep each bullet specific and actionable. Prefer paths under
`rust/`, plus `Cargo.toml` / `flutter_rust_bridge.yaml` when relevant. For
UI-only or Flutter call-site work, cite `lib/` under Related and point to
`client-locate-change-points` — do not edit Flutter or generated files from
this skill.

When a public API change is required, always include under Related:

- `flutter_rust_bridge_codegen generate` (required after `rust/src/api/` change);
- affected generated Dart paths / Flutter call sites (read-only from this skill).

## Confidence

Prefer saying "likely" when the location is inferred from naming, `AGENTS.md`
target UX, or nearby patterns. Say "confirmed" only when the inspected code
directly proves the relationship. When the tree is still scaffold-only, say so
and prefer **Where to add** for product modules rather than overloading
`api/simple.rs` / `process_text` without an explicit demo-only task.
