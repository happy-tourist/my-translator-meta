---
name: client-locate-change-points
description: >-
  Use when the user asks to analyze a task, locate implementation points, find
  affected files, or identify where Flutter `lib/` changes should be made without
  editing code in Translation Reader (`my_translator_app`). Covers screens,
  widgets, theme, i18n, FRB call sites, and Dart tests; flags when the real fix
  belongs in `rust/src/api/` (+ FRB codegen) rather than Dart.
---

# Locate Change Points

Use this skill when the user gives a task statement and wants only analysis of
where changes are needed in the Flutter UI layer of **Translation Reader**
(`my_translator_app`).

Stack: Flutter / Dart (`lib/`), Material (scaffold), `flutter_rust_bridge` 2.13
(generated imports under `lib/src/rust/`), Rust core in sibling `rust/` (same
repo). Resolve app path via [`docs/projects-map.md`](../../../../docs/projects-map.md)
→ `../my_translator_app`. Skills for this client live under
`.agents/skills/client/` in `my_translator_meta`.

**Not a Rust skill.** Point to `rust/src/api/` when the contract or heavy logic
must change; do not invent Rust module layouts or implement Rust here.

## Hard Rule

Do NOT edit, create, delete, format, or commit files.

Only inspect the codebase and report:

- existing files that likely need changes;
- exact screens, widgets, theme/i18n, FRB call sites, startup (`main`), tests,
  or related Rust/FRB touch points;
- places where new files should be added, if the task requires new code;
- open questions when the implementation location cannot be determined confidently.

## Client Layer Map

Composition roots under `lib/`. Search and assign ownership top-down. Today the
app is mostly **scaffold**: UI lives in `lib/main.dart`; product screens/widgets
are **target** layout — prefer them for new work, not growing the demo forever.

| Layer | Path | Role |
|-------|------|------|
| Entry / shell | `lib/main.dart` | `RustLib.init()` → `runApp`; `MaterialApp` / theme / home (scaffold: `MyApp` + `MyHomePage`) |
| Screens | `lib/` (future `screens/` / `pages/` / feature folders) | Reader, subtitle list/load, dictionary, settings, split view — create when the task needs a real screen |
| Widgets | `lib/` (future `widgets/` / feature folders) | Reusable UI: token highlight, word/phrase card, sync scroll panes; extract only when reuse is real |
| Theme | `ThemeData` in `MaterialApp` (today in `main.dart`); future dedicated theme file | Colors, typography, light/dark |
| Localization | future `*.arb` / `l10n` / `flutter_localizations` | User-visible copy (scaffold still hardcodes Russian/English strings in `main.dart`) |
| FRB imports (generated) | `lib/src/rust/**` | **Do not hand-edit.** Call sites import `package:my_translator_app/src/rust/api/…` + `frb_generated.dart` |
| FRB contract (Rust) | `rust/src/api/**/*.rs` | Public FFI surface; after change → `flutter_rust_bridge_codegen generate` |
| Glue | `rust_builder/` | cargokit/FFI only — not product logic |
| Unit / widget tests | `test/` | `flutter test` |
| Integration tests | `integration_test/` | e.g. `simple_test.dart` (may lag UI) |
| Config | `pubspec.yaml`, `flutter_rust_bridge.yaml`, `analysis_options.yaml` | Deps, FRB paths, lints |

Allowed dependency direction:

```text
lib/** (UI)  →  lib/src/rust/api/* (generated)  →  rust/src/api/*  →  rust core modules
```

Flutter orchestrates calls and paints; parse / lemmatize / dictionary / offline
translate / SQLite stay in Rust. Do not reimplement that logic in Dart.

### Current scaffold (confirmed)

| Item | Location |
|------|----------|
| Startup | `main()` → `await RustLib.init()` → `runApp(const MyApp())` |
| Demo UI | `MyHomePage`: `TextField` + «Вызвать Rust» + result text |
| Generated API used | `processText`, `addNumbers` from `api/simple.dart` |
| Router / state mgmt | None yet (no `go_router` / Provider / Riverpod / Bloc) |
| `test/` | Empty directory |
| `integration_test/simple_test.dart` | Out of sync with current `main.dart` UI |

### Target UX areas (product, mostly not in tree yet)

| Area | Prefer Flutter | Prefer Rust (+ thin `api/`) |
|------|----------------|------------------------------|
| Subtitle load / list | screen + file-pick UX | parse SRT/VTT/ASS → cues |
| Reader (lines, tap, highlight) | gestures, layout, paint tokens | tokenize / lemmatize / match |
| Word/phrase card | sheet/dialog UI, «add to dict» | translate + dictionary CRUD |
| Split view + sync scroll | two panes, scroll sync | translation lines / data |
| Dictionary screen | list UI | list/remove/lookup by lemma |
| Settings | languages UI | persist settings (target SQLite) |

## Decision Guidance

Use these rules to pick the layer before naming files.

### Where does UI live?

| If the change is… | Prefer |
|-------------------|--------|
| App startup / `RustLib.init` / root `MaterialApp` | `lib/main.dart` |
| A new product screen | new file under `lib/` (match any emerging `screens/` / feature folder peers; otherwise introduce a clear screen file next to `main.dart` and wire it from `MaterialApp` / navigator) |
| Screen-specific layout / gesture | owning screen widget |
| Reusable across screens | `lib/…/widgets/` (only when reuse is real; avoid premature extraction) |
| Theme / global `ThemeData` | `MaterialApp.theme` (and dark theme when added); extract theme file when shell grows |
| Copy / locale strings | l10n/`arb` when present; until then the owning widget — flag need for i18n if task is multi-locale |
| FRB call site (bind existing API) | screen/widget that owns the UX; import generated `api/*.dart` |
| New / changed FFI method or DTO | **`rust/src/api/`** then codegen; then update Dart call sites + tests — list Rust under Related, do not implement from this skill |
| Hand-edit `lib/src/rust/**` or `frb_generated.*` | Never — report as wrong target |
| `rust_builder/` | Only if build/glue breakage; not for features |

### FRB / Rust boundary

Shared session and heavy data belong in Rust via FRB, not ad-hoc Dart
reimplementation.

| Concern | Typical surface |
|---------|-----------------|
| Init bridge | `RustLib.init()` in `main()`; Rust `#[frb(init)]` |
| Demo / sample API | `rust/src/api/simple.rs` ↔ `lib/src/rust/api/simple.dart` (generated) |
| Parse subtitles | Rust modules + thin export in `rust/src/api/`; Flutter shows cues |
| Tokens / lemmas / highlight keys | Rust prepares; Flutter paints / handles taps |
| Dictionary CRUD + lemma lookup | Rust + SQLite (target); Flutter list/card actions call API |
| Offline translate word/phrase | Rust; Flutter shows result card |
| Optional AI (later) | Explicit later; do not invent providers from this skill |
| Errors / loading | Flutter UI state around async FRB calls; map Rust errors to banners/snackbars when present |

Local widget state is fine for ephemeral UI (controllers, scroll offset, selection)
that never needs to be authoritative product data.

### Forms, i18n, tests, config

- Demo text + button → `lib/main.dart` (`MyHomePage`) until product screens exist.
- Reader / dictionary / load / settings UX → new screens under `lib/` + FRB call sites; Related: matching `rust/src/api/` if contract missing.
- Theme tokens / Visual Spec-driven chrome → theme + owning widgets; mock-driven sizes/colors may also cite OpenSpec `design.md` Visual Spec (implement elsewhere).
- Locale messages → l10n when introduced; otherwise owning widgets + note i18n gap.
- Widget/unit tests → `test/` + `flutter test`; colocate by feature when the tree grows.
- Integration tests → `integration_test/`; update when FFI or primary UX changes (`simple_test.dart` currently expects old copy).
- Pub / FRB config → `pubspec.yaml`, `flutter_rust_bridge.yaml` (rarely for feature UI).

## Domain Hotspots

| Domain | Start here |
|--------|------------|
| Scaffold demo / FRB hello | `lib/main.dart` + generated `api/simple.dart`; Related: `rust/src/api/simple.rs` |
| App shell / title / theme | `MyApp` in `lib/main.dart` |
| Startup / bridge init | `main()` + `RustLib.init()` |
| Subtitle load / list (target) | new load/list screen under `lib/`; Related: parse API in `rust/src/api/` |
| Reader + tap + lemma highlight (target) | new reader screen/widgets; Related: tokenize/lemmatize/dictionary APIs |
| Word/phrase card + add to dict (target) | card widget + FRB add/lookup; Related: dictionary/translate API |
| Split view sync scroll (target) | split screen widgets; Related: translation line data from Rust |
| Dictionary screen (target) | dictionary screen; Related: dictionary list/remove API |
| Settings (target) | settings screen; Related: settings persistence in Rust when added |
| Generated Dart bindings | `lib/src/rust/**` (read-only; regen after API change) |
| Dart tests | `test/`, `integration_test/` |
| Platform runners | `android/` `ios/` `web/` only if the task is platform config — not default for UI features |

## Workflow

1. Read the task statement and extract:
   - target feature or behavior;
   - entities (subtitle document, token/lemma, dictionary entry, reader session, settings);
   - whether the task changes existing scaffold UI or adds a product flow;
   - whether the **FRB/Rust contract** (new method, DTO, error shape) is involved vs pure Flutter chrome.

2. Search by domain terms in `../my_translator_app`:
   - UI symbols (`MyApp`, `MyHomePage`, `MaterialApp`, `processText`, `addNumbers`, `RustLib`);
   - generated API paths (`src/rust/api`, `frb_generated`);
   - product terms (`subtitle`, `lemma`, `dictionary`, `reader`, `split`, `translate`) — expect many hits only in docs/`AGENTS.md` until implemented;
   - user-visible strings in `lib/` and future l10n;
   - tests under `test/` and `integration_test/`.

3. Walk ownership:
   - screen/widget → FRB import → (if missing or wrong) `rust/src/api/`;
   - theme/i18n from the shell;
   - flag any proposed hand-edit of `lib/src/rust/**` as invalid;
   - flag Dart reimplementation of parse/NLP/dict as a smell → Relocate to Rust.

4. Apply decision guidance above to classify each hit as edit vs add, and note related FRB/Rust touch points under Related (do not implement them from this skill).

5. Stop after identifying change points. Do not implement.

## Output Format

Respond in this structure:

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

Omit empty sections. Keep each bullet specific and actionable. Prefer paths under
`lib/`, plus `test/` / `integration_test/` when relevant. For FFI/contract work,
cite `rust/src/api/` (and codegen) under Related — do not edit Rust or generated
Dart from this skill.

## Confidence

Prefer saying "likely" when the location is inferred from naming, `AGENTS.md`
target UX, or nearby patterns. Say "confirmed" only when the inspected code
directly proves the relationship. When the tree is still scaffold-only, say so
and prefer **Where to add** for product screens rather than overloading
`MyHomePage` without an explicit demo-only task.
