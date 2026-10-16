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

Stack (from `{app}/AGENTS.md`): Flutter / Dart (`lib/`), Rust compute (`rust/`),
**flutter_rust_bridge** generated Dart (`lib/src/rust/` — do not edit by hand),
**Drift** for user persistence. No backend. Heavy compute (parse, tokenize,
lemmatize; offline translate **on native**) stays in Rust. **Web** offline
translate may use Transformers.js behind `lib/data/translate/` (same UX
contract — see `work-with-translate`). Flutter is screens, gestures, thin
FRB/facade orchestration, and Drift data layer (folders / documents / dictionary).

Path package: `package:my_translator_app/...`.

Skills for this client live under `.agents/skills/client/` in
`my_translator_meta`.

## Core Rules

1. Before adding folders, inspect a nearby peer in the **same layer** and match
   its file name and import style. Match the **current** `lib/app` +
   `lib/data` + `lib/features` layout — do not dump product UI into `main.dart`.
2. Create only the files the feature needs. Do not add empty stubs “for later”.
3. Place UI by layer role (route/screen vs reusable widget vs orchestration).
4. Respect **allowed dependency direction** (see below). Never invert layers.
5. Import Dart by direct path. No per-feature `index.dart` barrels required.
6. Keep `main.dart` thin: `await RustLib.init()` then `runApp`. Shared chrome
   (app bar, nav shell) lives in one app/shell widget — not duplicated on every
   screen.
7. Keep Drift / FRB I/O in a thin orchestration layer (screen state, a dedicated
   controller/notifier, or a small service class). Prefer widgets that take
   props/callbacks over calling repositories or generated API deep in leaves.
8. Do **not** reimplement parse / lemma / translate rules in Dart on **native**
   (call FRB). On **Web**, offline translate may use the documented JS engine
   only via `lib/data/translate/` facade — never invent glosses in widgets.
   User persist belongs in Drift (`work-with-drift`), not Rust SQLite.
9. Do **not** mandate Riverpod, Bloc, Provider, or go_router until
   `{app}/AGENTS.md` (or an OpenSpec change) adopts one. StatefulWidget /
   `ChangeNotifier` / plain async methods are fine; if a package is added later,
   keep the same layer roles.

## Specialized Topics

| Topic | File / skill |
|-------|----------------|
| Drift schema / repos / Web | [`work-with-drift`](../work-with-drift/SKILL.md) |

## Folder Map (`lib`) — current + growth

| Layer | Path | Role |
|-------|------|------|
| Entry | `lib/main.dart` | `RustLib.init()` + `runApp`; no feature markup |
| App shell | `lib/app/` | `MaterialApp`, Drift open, Library\|Vocab shell |
| Data (Drift) | `lib/data/` | Database, repositories, document content store |
| Data (video extract) | `lib/data/video/` | FE-only frame probe/extract facade (FFmpegKit / web canvas; no FRB) |
| Data (translate facade) | `lib/data/translate/` | Offline MT: native FRB ORT \| Web Transformers.js |
| Features | `lib/features/<area>/` | Screen + local widgets + optional controller for one product area |
| Shared widgets | `lib/widgets/` or `lib/shared/` | Cross-feature UI once reused |
| Orchestration | beside screen or `…/controllers/` | Loading/error flags, Drift/FRB calls, UI models |
| Generated FRB | `lib/src/rust/` | **GENERATED** — never hand-edit |
| Assets | `assets/` (pubspec); Drift Web under `web/` | Images / fonts; `sqlite3.wasm`, `drift_worker.js` |

**Feature areas:**

| Area | Folder | Screens / focus |
|------|--------|-----------------|
| Upload / Library | `features/upload/` | Folders + import (`library_screen.dart`) — **present** |
| Vocab | `features/vocab/` | Dictionary list / add / remove — **present** |
| Reader | `features/reader/` | Line list, contiguous tap Add, lemma/phrase highlight/edit (`reader/highlight`), optional paged video frames (`reader/video-frames`) |
| Split view | under `reader/` (or `features/split/`) | Original ↔ translation panes (`reader/split`) |
| Settings | `features/settings/` | Languages; later optional AI (later) |

Outside `lib/`: `rust/` (compute), `rust_builder/` (glue only), `test/`,
`integration_test/`, platform runners. Do not put business logic in
`rust_builder/`.

**Not used:** generic `blocks/` / `dialogs/` registry layers unrelated to
features.

## Dependency Direction

Allowed:

```text
main / app shell
  → features/*/screens (or routes)
      → features/*/widgets  +  shared widgets
      → orchestration (local state / controller)
          → lib/data/** repositories (Drift)
          → package:my_translator_app/src/rust/api/*   (generated, compute)
              → rust/src/api/*  →  rust modules (parse / NLP / translate)
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
4. **Parse / tokenize / lemma / offline translate (native)?** → Rust
   (`rust/src/…` + thin `rust/src/api/`), then codegen — not Dart.
   **Web offline translate exception:** Transformers.js behind
   `lib/data/translate/` facade (conditional `io`/`html`); UI never calls FRB
   translate on web — see `work-with-translate` + OpenSpec design D4.
4b. **User persist (folders / documents / dictionary / settings)?** → Drift
   in `lib/` (data layer / repositories) — not Rust SQLite / FRB storage.
4c. **Video frame probe / JPEG extract (`reader/video-frames`)?** → Flutter
   facade under `lib/data/video/` (platform conditional export; injectable
   fake in tests) — **not** Rust/FRB. Pages UI under `features/reader/`
   (`video_frame_pages.dart`); attach/cache via Drift — see
   `work-with-reader` / `video-frames.md`.
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

- Drift repository calls: folders / documents / dictionary CRUD, settings later.
- FRB calls (when compute APIs exist): parse document, tokenize line, translate
  word/phrase.
- UI-facing loading / error flags and mapped view models.

**Put in Rust (via FRB)**

- Subtitle parse, NLP (tokenize/lemma), offline translate **on native** —
  compute only. User persistence is Drift, not Rust SQLite.
- **Web:** offline translate compute lives in JS worker; Dart owns only the
  facade under `lib/data/translate/` (same UX contract).

### What NOT to put

| Avoid | Prefer |
|-------|--------|
| Parsing / lemmatizing in Dart | Rust API + generated Dart call |
| Editing generated `lib/src/rust/**` | Change `rust/src/api/` → codegen |
| Mandatory Riverpod/Bloc “because another stack had a store” | Match AGENTS; add a package only when chosen |
| Empty `features/*` folders for unused areas | Add when the first file is needed |
| Dumping product UI into `main.dart` | Feature screens under `lib/features/` |
| FRB / Drift calls inside every decorative child | One orchestration / screen call site |

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

**Upload / Library** — root + folders + import via Drift; file picker bytes →
Drift document store. Reader/parse later via FRB. No SRT grammar in Dart.

**Reader** — screen shows cue lines; tap/selection → orchestration asks Rust
for token/lemma → CTA/sheet; Translate via `OfflineTranslateFn` fills
Translation; “add to vocab” → Drift
dictionary (same store as Vocab UI). Highlight spans from Rust-prepared token
data; Flutter paints.

**Split view** — two synced scroll views (original / translation). Translation
text and alignment data from Rust; Flutter owns scroll controllers and layout.

**Vocab** — list/add/delete via Drift + Translate via `OfflineTranslateFn`.
Empty state supports in-UI add; Reader sheets use the same dictionary store.
No second source of truth in SharedPreferences.

**Settings** — languages (and later optional AI). Persist via Drift when
settings land; UI does not invent a parallel prefs schema for core data.

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
2. Call Drift repositories for user data; import generated FRB API only for
   compute (`package:my_translator_app/src/rust/api/...`).
3. After Rust API changes: run FRB codegen, then update callers/tests.

**New compute capability (parse / NLP / translate)**

1. Implement in `rust/`; export thin surface in `rust/src/api/`.
2. Codegen → update Flutter orchestration only.

**New user persist (folders / documents / dictionary / settings)**

1. Implement under `lib/data/` per `work-with-drift`.
2. Wire screens via constructor-injected repositories — not FRB storage.

## Domain Anchors

| Domain | Flutter (target) | Authority |
|--------|------------------|-----------|
| Library folders/documents | `features/upload/` + Drift data | Drift persist; Rust parse later |
| Reader + tap/select | `features/reader/` | Rust tokens / lemmas / translate |
| Split view | `features/split/` or `reader/` | Flutter one-shot align (Show / load); independent panes after |
| Vocab | `features/vocab/` | Drift dictionary persist |
| Settings | `features/settings/` | Drift (target) / local UI until wired |
| Word/phrase card | reader widgets | Rust gloss + Drift add |
| App chrome | `lib/app/` | Flutter only |
| FRB contract | `lib/src/rust/` (generated) | `rust/src/api/` (compute) |

User entities (Library folder/document, Dictionary entry, Settings) — **Drift**
truth. Compute entities (Token/Lemma, parsed cues) — Rust when those APIs land.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Dumping new UI into `main.dart` | Add `features/.../*_screen.dart` |
| Parsing subtitles or lemmatizing in Dart | Move to Rust + FRB |
| Hand-editing generated Dart/Rust FRB files | Edit `rust/src/api/` → codegen |
| Mandating Riverpod/Bloc without AGENTS decision | Use simple state; adopt package when documented |
| FRB calls in every small decorative widget | Centralize in orchestration / screen |
| Empty feature folders “for later” | Create on first real file |
| Importing a screen from a shared widget | Invert: screen imports widget |
| Putting user persist in Rust SQLite / FRB storage | Drift in `lib/` data layer |
| Reimplementing parse/lemma/translate in Dart | Rust compute + FRB |
| Treating `rust_builder/` as app logic | Leave as glue |

## Related Skills

- Drift user persist → `work-with-drift`
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
- [ ] Dependency direction respected (UI → orchestration → Drift and/or FRB → Rust compute)
- [ ] User persist under `lib/data/`; no hand-edits under `lib/src/rust/` or `*.g.dart`
- [ ] Heavy compute not duplicated in Dart
- [ ] No mandatory state-management/router package beyond AGENTS
- [ ] Library / vocab / reader / settings / split land under agreed feature areas
