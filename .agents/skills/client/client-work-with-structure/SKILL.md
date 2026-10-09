---
name: client-work-with-structure
description: >-
  Use when placing or moving Flutter UI in my_translator_app (Translation
  Reader): screens, widgets, feature folders, presentation vs orchestration,
  FRB call sites, lib/ layout for upload / reader / vocab / settings / split
  view, or deciding whether new code belongs in a screen, widget, thin
  controller, or Rust via FRB. Flutter Material only; router and
  state-management package not mandated until AGENTS chooses one.
---

# Work With Structure

## Overview

Use this skill when creating or relocating UI under `lib/` in
**my_translator_app** (product: **Translation Reader**).

Stack (from `{app}/AGENTS.md`): Flutter / Dart (`lib/`), Rust core (`rust/`),
**flutter_rust_bridge** generated Dart (`lib/src/rust/` — do not edit by hand).
No backend. Heavy work (parse, tokenize, lemmatize, dictionary, offline
translate, SQLite) stays in Rust; Flutter is screens, gestures, and thin FRB
orchestration.

Path package: `package:my_translator_app/...`.

Skills for this client live under `.agents/skills/client/` in
`my_translator_meta`.

## Core Rules

1. Before adding folders, inspect a nearby peer in the **same layer** and match
   its file name and import style. While the repo is still scaffold
   (`lib/main.dart` demo), follow the **target** layout in this skill, not the
   demo dump.
2. Create only the files the feature needs. Do not add empty stubs “for later”.
3. Place UI by layer role (route/screen vs reusable widget vs orchestration).
4. Respect **allowed dependency direction** (see below). Never invert layers.
5. Import Dart by direct path. No per-feature `index.dart` barrels required.
6. Keep `main.dart` thin: `await RustLib.init()` then `runApp`. Shared chrome
   (app bar, nav shell) lives in one app/shell widget — not duplicated on every
   screen.
7. Keep FRB I/O in a thin orchestration layer (screen state, a dedicated
   controller/notifier, or a small service class). Prefer widgets that take
   props/callbacks over calling generated API deep in leaf widgets.
8. Do **not** reimplement parse / lemma / dictionary / translate rules in Dart.
9. Do **not** mandate Riverpod, Bloc, Provider, or go_router until
   `{app}/AGENTS.md` (or an OpenSpec change) adopts one. StatefulWidget /
   `ChangeNotifier` / plain async methods are fine; if a package is added later,
   keep the same layer roles.

## Folder Map (`lib`) — target

Scaffold today is flat (`main.dart` + generated `src/rust/`). Grow toward
feature folders under `lib/`:

| Layer | Path | Role |
|-------|------|------|
| Entry | `lib/main.dart` | `RustLib.init()` + `runApp`; no feature markup |
| App shell | `lib/app/` | `MaterialApp` / theme / optional router host |
| Features | `lib/features/<area>/` | Screen + local widgets + optional controller for one product area |
| Shared widgets | `lib/widgets/` or `lib/shared/` | Cross-feature UI (word chip chrome, empty states) once reused |
| Orchestration | beside screen or `…/controllers/` | Loading/error flags, FRB calls, UI-facing models |
| Generated FRB | `lib/src/rust/` | **GENERATED** — never hand-edit |
| Assets | `assets/` (pubspec) | Images / fonts when needed |

**Feature areas (target):**

| Area | Suggested folder | Screens / focus |
|------|------------------|-----------------|
| Upload | `features/upload/` | Pick / list subtitle files; hand off to Rust parse |
| Reader | `features/reader/` | Line list, tap/selection, lemma highlight paint |
| Split view | `features/split/` or under `reader/` | Original ↔ translation panes, sync scroll |
| Vocab | `features/vocab/` | Dictionary list / entry detail |
| Settings | `features/settings/` | Languages; later optional AI provider |

Outside `lib/`: `rust/` (authority), `rust_builder/` (glue only), `test/`,
`integration_test/`, platform runners. Do not put business logic in
`rust_builder/`.

**Not used:** generic `blocks/` / `dialogs/` registry layers unrelated to
features.

Scaffold leftovers (`MyHomePage` demo TextField / `processText`) — do not
extend for product features; replace with feature screens as work lands.

## Dependency Direction

Allowed:

```text
main / app shell
  → features/*/screens (or routes)
      → features/*/widgets  +  shared widgets
      → orchestration (local state / controller)
          → package:my_translator_app/src/rust/api/*   (generated)
              → rust/src/api/*  →  rust modules (parse / NLP / dict / db)
```

Also normal:

- Screens compose widgets and call orchestration methods.
- Leaf widgets prefer constructor params + callbacks; avoid importing FRB
  unless the widget is the intentional call site for a tiny one-off.
- Shared chrome (nav, theme) in `app/` shell, not copied into each feature.

**Forbidden inversions:**

| Wrong | Why |
|-------|-----|
| Widget → screen / route type | Widgets stay screen-agnostic |
| Orchestration → screens/widgets for layout | Data/async layer must not import UI trees |
| Dart re-parse of SRT/VTT/ASS or lemma rules | Belongs in Rust |
| Hand-edit `lib/src/rust/**` or `rust/src/frb_generated.rs` | Regenerate via FRB after `rust/src/api/` changes |
| Business logic in `rust_builder/` | Glue / cargokit only |
| Scattering FRB calls across many leaf widgets | Keep I/O in orchestration (or one clear screen method) |

## Where New UI Belongs

Decide in this order:

1. **New screen / major flow?** → `lib/features/<area>/<name>_screen.dart` (+
   route/shell entry when navigation exists).
2. **Reusable control used in 2+ screens or clearly generic?** → feature-local
   `widgets/` first; promote to `lib/widgets/` / `lib/shared/` when a second
   feature needs it.
3. **Async FRB / session / list loading?** → thin orchestration next to the
   screen (or `controllers/`), not inline business rules in `build()`.
4. **Parse / tokenize / lemma / dictionary / SQLite / offline translate?** →
   Rust (`rust/src/…` + thin `rust/src/api/`), then codegen — not Dart.
5. **Page-local sheet / dialog?** → Keep in the screen or extract a widget —
   **do not** invent a global dialogs registry.

### Screens vs widgets vs orchestration

**Put in screens**

- Route/entry widget and screen orchestration (selection, scroll position,
  which cue is focused).
- Screen-only chrome (titles, FABs tied to that flow).
- Thin wiring: call orchestration, show loading / error / empty.

**Put in widgets**

- Small reusable pieces (token span painter, vocab list tile, split pane
  chrome) shared or clearly extractable.
- Prefer extracting when reuse or screen size justifies it — screens may own
  substantial markup early.

**Put in orchestration**

- FRB calls: parse document, tokenize line, lookup lemma, add/remove vocab,
  translate word/phrase, load settings.
- UI-facing loading / error flags and mapped view models (do not mirror full
  Rust domain graphs in Dart).

**Put in Rust (via FRB)**

- Subtitle parse, NLP, dictionary truth, offline translate, persistence.

### What NOT to put

| Avoid | Prefer |
|-------|--------|
| Parsing / lemmatizing in Dart | Rust API + generated Dart call |
| Editing generated `lib/src/rust/**` | Change `rust/src/api/` → codegen |
| Mandatory Riverpod/Bloc “because another stack had a store” | Match AGENTS; add a package only when chosen |
| Empty `features/*` folders for unused areas | Add when the first file is needed |
| Growing the scaffold `MyHomePage` into product | New feature screens |
| FRB calls inside every decorative child | One orchestration / screen call site |

## Naming

| Kind | Convention | Examples |
|------|------------|----------|
| Screen file | snake_case + `_screen` | `reader_screen.dart`, `upload_screen.dart`, `vocab_screen.dart` |
| Screen class | PascalCase + `Screen` | `ReaderScreen`, `SplitViewScreen` |
| Widget file | snake_case | `lemma_highlight_text.dart`, `vocab_entry_tile.dart` |
| Widget class | PascalCase | `LemmaHighlightText`, `VocabEntryTile` |
| Controller / notifier | descriptive | `reader_controller.dart`, `vocab_list_controller.dart` |
| Feature folder | lowercase area | `features/reader/`, `features/vocab/` |

Screens may live as flat files under `features/<area>/` (e.g.
`features/reader/reader_screen.dart`), matching peers once they exist.

## Typical Shapes

### Feature screen

```text
lib/features/reader/reader_screen.dart
lib/features/reader/widgets/cue_line.dart
lib/features/reader/reader_controller.dart   # optional; FRB + UI state
```

### Shared widget (when promoted)

```text
lib/widgets/empty_state.dart
```

### App entry

```text
lib/main.dart          # RustLib.init + runApp
lib/app/app.dart       # MaterialApp / theme / shell
```

### FRB (generated — do not edit)

```text
lib/src/rust/api/….dart
lib/src/rust/frb_generated.dart
```

## Real Composition Examples (target)

**Upload** — screen picks a file path/bytes → orchestration calls Rust parse →
navigates/opens a reader session with structured cues. No SRT grammar in Dart.

**Reader** — screen shows cue lines; tap/selection → orchestration asks Rust
for token/lemma + translate → shows word/phrase card; “add to vocab” → Rust
dictionary API. Highlight spans come from Rust-prepared token data; Flutter
only paints.

**Split view** — two synced scroll views (original / translation). Translation
text and alignment data from Rust; Flutter owns scroll controllers and layout.

**Vocab** — list/detail screens; CRUD via FRB. No local “source of truth” list
that diverges from Rust/SQLite.

**Settings** — languages (and later optional AI). Persist through Rust when
storage exists; UI does not invent a parallel prefs schema for core data.

**App shell** — hosts navigation between upload / reader / vocab / settings;
keeps shared theme. No per-screen duplicate global nav once a shell exists.

## Creating New Pieces — Checklist

**New screen**

1. `lib/features/<area>/<name>_screen.dart`.
2. Wire into app shell / router **when** navigation exists (do not invent a
   mandatory router package).
3. Call FRB only through orchestration or a clear screen method.
4. Leave shared chrome in `app/` shell.

**New widget**

1. Feature-local `widgets/<name>.dart`, or `lib/widgets/` if cross-feature.
2. Prefer props/callbacks; do not import screens.
3. Avoid direct FRB unless this widget is the deliberate call site.

**New orchestration**

1. Keep next to the feature; expose loading/error and UI models.
2. Import generated API from `package:my_translator_app/src/rust/api/...`.
3. After Rust API changes: run FRB codegen, then update callers/tests.

**New core capability (parse / NLP / dict)**

1. Implement in `rust/`; export thin surface in `rust/src/api/`.
2. Codegen → update Flutter orchestration only.

## Domain Anchors

| Domain | Flutter (target) | Authority |
|--------|------------------|-----------|
| Upload / file list | `features/upload/` | Rust parse + file handling |
| Reader + tap/select | `features/reader/` | Rust tokens / lemmas / translate |
| Split view | `features/split/` or `reader/` | Rust line pairing; Flutter sync scroll |
| Vocab | `features/vocab/` | Rust dictionary + SQLite |
| Settings | `features/settings/` | Rust (target) / local UI until API exists |
| Word/phrase card | reader widgets | Rust lookup + add |
| App chrome | `lib/app/` | Flutter only |
| FRB contract | `lib/src/rust/` (generated) | `rust/src/api/` |

Entity names from AGENTS (Subtitle document, Token/Lemma, Dictionary entry,
Reader session, Settings) live as **Rust truth**; Dart holds view-facing
snapshots only.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Growing scaffold `MyHomePage` into the product | Add `features/.../*_screen.dart` |
| Parsing subtitles or lemmatizing in Dart | Move to Rust + FRB |
| Hand-editing generated Dart/Rust FRB files | Edit `rust/src/api/` → codegen |
| Mandating Riverpod/Bloc without AGENTS decision | Use simple state; adopt package when documented |
| FRB calls in every small decorative widget | Centralize in orchestration / screen |
| Empty feature folders “for later” | Create on first real file |
| Importing a screen from a shared widget | Invert: screen imports widget |
| Putting SQLite or dict rules in Flutter | Rust persistence / domain |
| Treating `rust_builder/` as app logic | Leave as glue |

## Related Skills

- Mock measurement / Visual Spec → `prepare-mock`
- Pixel/check vs mock → `verify-mock`
- Flutter implementability of meta claims → `align-implementability-front`
- Rust/API implementability → `align-implementability-back`
- App AGENTS canon → `{app}/AGENTS.md` via `docs/projects-map.md`

## Verification

For structure-only placement tasks, confirm:

- [ ] Correct layer (feature screen / widget / orchestration / Rust+FRB)
- [ ] `*_screen.dart` / clear feature folder (not dumping into `main.dart`)
- [ ] No new generic `blocks/` or dialogs registry
- [ ] Dependency direction respected (UI → orchestration → generated FRB → Rust)
- [ ] No hand-edits under `lib/src/rust/`
- [ ] Heavy logic not duplicated in Dart
- [ ] No mandatory state-management/router package beyond AGENTS
- [ ] Upload / reader / vocab / settings / split land under agreed feature areas
