---
name: client-locate-change-points
description: >-
  Use when the user asks to analyze a task, locate implementation points, find
  affected files, or identify where Flutter `lib/` changes should be made without
  editing code in Translation Reader (`my_translator_app`). Covers screens,
  widgets, theme, i18n, Drift data layer, FRB call sites, and Dart tests; flags
  when the real fix belongs in `rust/src/api/` (+ FRB codegen) rather than Dart.
---

# Locate Change Points

Use this skill when the user gives a task statement and wants only analysis of
where changes are needed in the Flutter UI layer of **Translation Reader**
(`my_translator_app`).

Stack: Flutter / Dart (`lib/`), Material, **Drift** (`lib/data/`),
`flutter_rust_bridge` 2.13 (generated under `lib/src/rust/`), Rust compute in
sibling `rust/` (same repo). Resolve app path via
[`docs/projects-map.md`](../../../../docs/projects-map.md)
→ `../my_translator_app`. Skills for this client live under
`.agents/skills/client/` in `my_translator_meta`.

**Not a Rust skill.** Point to `rust/src/api/` when the contract or heavy
**compute** must change; do not invent Rust module layouts or implement Rust
here. User persist → Drift (`work-with-drift`), not Rust SQLite.

## Hard Rule

Do NOT edit, create, delete, format, or commit files.

Only inspect the codebase and report:

- existing files that likely need changes;
- exact screens, widgets, theme/i18n, Drift repos, FRB call sites, startup
  (`main`), tests, or related Rust/FRB touch points;
- places where new files should be added, if the task requires new code;
- open questions when the implementation location cannot be determined confidently.

## Client Layer Map

Composition roots under `lib/`:

| Layer | Path | Role |
|-------|------|------|
| Entry | `lib/main.dart` | `RustLib.init()` → `runApp(TranslationReaderApp)` or fatal init UI |
| App shell | `lib/app/app.dart`, `lib/app/shell.dart` | Drift open; Library \| Vocab shell |
| Data (Drift) | `lib/data/database/`, `lib/data/library/`, `lib/data/dictionary/` | Schema, repos, content store |
| Features | `lib/features/<area>/` | Product screens (upload, vocab, …) |
| Widgets | feature-local or future `lib/widgets/` | Reusable UI when reuse is real |
| Theme | `lib/app/` (`MaterialApp` theme) | Colors, typography, light/dark |
| Localization | future `*.arb` / `l10n` | User-visible copy (many strings still hardcoded) |
| FRB imports (generated) | `lib/src/rust/**` | **Do not hand-edit.** Compute call sites only |
| FRB contract (Rust) | `rust/src/api/**/*.rs` | Public FFI surface; after change → codegen |
| Glue | `rust_builder/` | cargokit/FFI only — not product logic |
| Unit / widget tests | `test/` (`test/data/`, `test/features/`, `test/support/`) | `flutter test` |
| Integration tests | `integration_test/` | e.g. `simple_test.dart` |
| Config | `pubspec.yaml`, `flutter_rust_bridge.yaml`, `analysis_options.yaml` | Deps, FRB paths, lints |
| Drift Web assets | `web/sqlite3.wasm`, `web/drift_worker.js` | Drift Web |

Allowed dependency direction:

```text
lib/app + lib/features  →  lib/data (Drift | translate facade)
  and/or  lib/src/rust/api/* (generated, native compute)
  →  rust/src/api/*  →  rust compute modules
```

Flutter owns UI + Drift user persist; parse / lemmatize / offline translate
(**native**) stay in Rust. Web offline translate: JS behind `lib/data/translate/`
(see `work-with-translate`). Do not reimplement NLP/parse or invent glosses in
widgets.

### Current product shell (confirmed)

| Item | Location |
|------|----------|
| Startup | `main()` → `RustLib.init()` → `TranslationReaderApp` |
| Drift open | `lib/app/app.dart` via `openAppDatabase()` |
| Shell | `AppShell` — Library (default) + Vocabulary |
| Library UI | `lib/features/upload/library_screen.dart` |
| Vocab UI | `lib/features/vocab/vocabulary_screen.dart` |
| Drift data | `lib/data/**` repositories + `AppDatabase` |
| Router / state mgmt | Index in `AppShell` (no go_router / Riverpod yet) |
| Tests | `test/data/*`, `test/features/*`, `test/support/test_database.dart` |

### UX areas

| Area | Prefer Flutter | Prefer Rust (+ thin `api/`) |
|------|----------------|------------------------------|
| Subtitle library | folders + file-pick UX; Drift persist | parse SRT/VTT/ASS → cues (later) |
| Reader (lines, tap, highlight) | gestures, layout, paint tokens | tokenize / lemmatize / match |
| Vocab / Reader Translate → Translation field | sheet UI + `OfflineTranslateFn` | native FRB / Web JS behind `lib/data/translate/`; add → Drift |
| Split view + one-shot align | two panes; Show / load align only | translation cues / chapters from parse |
| Dictionary screen | list/add/delete UI + Drift | — (persist not in Rust) |
| Settings | languages UI | persist settings (Drift later) |

## Decision Guidance

### Where does UI / data live?

| If the change is… | Prefer |
|-------------------|--------|
| App startup / `RustLib.init` | `lib/main.dart` |
| Drift open / retry chrome | `lib/app/app.dart` |
| Primary nav / shell tabs | `lib/app/shell.dart` |
| Folders / documents / dictionary schema or repos | `lib/data/**` (`work-with-drift`) |
| Library / import UX | `lib/features/upload/` |
| Vocab UX | `lib/features/vocab/` |
| A new product screen | `lib/features/<area>/` + wire from shell |
| Theme / global `ThemeData` | `lib/app/` |
| FRB call site (compute) | owning screen/orchestration; generated `api/*.dart` |
| New / changed FFI method or DTO | **`rust/src/api/`** then codegen; then Dart call sites + tests |
| Hand-edit `lib/src/rust/**` or `*.g.dart` | Never — report as wrong target |
| `rust_builder/` | Only if build/glue breakage; not for features |

### FRB / Drift boundary

| Concern | Typical surface |
|---------|-----------------|
| Init bridge | `RustLib.init()` in `main()` |
| User folders / documents / dictionary | Drift repositories in `lib/data/` |
| Parse subtitles (later) | Rust + thin `api/`; Flutter shows cues |
| Tokens / lemmas / highlight keys | Rust prepares; Flutter paints |
| Offline translate word/phrase | `lib/data/translate/` facade; Vocab/Reader Translate fills Translation (not instant word card) |
| Errors / loading | Flutter UI state around async Drift/FRB calls |

### Forms, i18n, tests, config

- Library / Vocab UX → feature screens + Drift repos.
- Theme tokens → `lib/app/` + owning widgets.
- Locale messages → l10n when introduced; otherwise owning widgets.
- Widget/unit tests → `test/data`, `test/features`, `test/support`.
- Integration tests → `integration_test/` when primary UX or FFI boot changes.
- Pub / FRB / Drift deps → `pubspec.yaml` (+ codegen tools).

## Domain Hotspots

| Domain | Start here |
|--------|------------|
| Startup / bridge init | `lib/main.dart` |
| Drift open / shell | `lib/app/app.dart`, `lib/app/shell.dart` |
| Library folders/documents | `lib/data/library/*`, `lib/features/upload/library_screen.dart` |
| Dictionary CRUD | `lib/data/dictionary/*`, `lib/features/vocab/vocabulary_screen.dart` |
| Reader + tap + highlight | `lib/features/reader/`; Related: tokenize/lemmatize/mark FRB (`add-reader-lemma-highlight`) |
| Offline Translate (Vocab / Reader sheets) | `lib/data/translate/`; Vocab + Reader sheets; native FRB / Web JS |
| Split view | `lib/features/reader/` (mode); Related: translation cues from parse |
| Settings (later) | `features/settings/`; Related: Drift settings when added |
| Generated Dart bindings | `lib/src/rust/**` (read-only) |
| Drift generated | `lib/data/database/*.g.dart` (read-only; `build_runner`) |
| Dart tests | `test/`, `integration_test/` |
| Web Drift / FRB | `web/`, COOP/COEP run flags — `work-with-env-deploy` |

## Workflow

1. Read the task and extract feature, entities (folder, document, dictionary
   entry, reader session, settings), and whether it touches **Drift**, **UI
   chrome**, and/or **FRB compute**.
2. Search in `../my_translator_app`: `lib/app`, `lib/data`, `lib/features`,
   `test/`, generated `src/rust/api`, product terms.
3. Walk ownership: screen → repository/FRB → (if missing compute contract)
   `rust/src/api/`. Flag hand-edits of generated files as invalid.
4. Stop after identifying change points. Do not implement.

## Output Format

```markdown
## Where to edit
- `path/to/file.dart` — why this file is relevant and what kind of change is expected.

## Where to add
- `path/to/new_file.dart` — why a new file belongs here.

## Related places
- `path/to/related` — why it should be checked during implementation (incl. `rust/src/api/` + codegen when the contract changes).

## Questions
- Clarifying question, only if needed.
```

Omit empty sections. Prefer paths under `lib/`, `test/`, `integration_test/`.
For FFI/contract work, cite `rust/src/api/` under Related — do not edit Rust or
generated Dart from this skill.

## Confidence

Prefer "likely" when inferred from naming/`AGENTS.md`. Say "confirmed" only when
inspected code proves the relationship.
