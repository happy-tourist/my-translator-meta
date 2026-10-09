---
name: work-with-pages
description: >-
  Use when creating or changing Flutter screens under lib/features/* in
  my_translator_app (Translation Reader): upload/list, reader (+ word card /
  highlight), split view, vocabulary, settings, app shell / navigation wiring,
  or replacing scaffold MyHomePage. Core in SKILL.md; topics in shell.md,
  upload.md, reader.md, vocab.md, settings.md. Structure / folder roles →
  client-work-with-structure. Flutter Material screens only — product surfaces
  (upload, reader, vocab, settings).
---

# Work With Pages

Use this skill when adding or changing **screens** (route-level Flutter widgets)
in **my_translator_app** (product: **Translation Reader**).

Stack: **Flutter / Dart** (`lib/`), **flutter_rust_bridge** generated API
(`lib/src/rust/` — do not hand-edit), Rust core (`rust/`).

Skills for this client live under `.agents/skills/client/` in
`my_translator_meta`. App canon: `{app}/AGENTS.md`. Folder roles:
`client-work-with-structure`.

## Specialized Topics

Read the matching file in this folder when the change involves that area
(do not load every file at once):

| Topic | File |
|-------|------|
| App shell — MaterialApp, nav, chrome | [shell.md](shell.md) |
| Upload + subtitle library list | [upload.md](upload.md) |
| Reader, word card, highlight, split view | [reader.md](reader.md) |
| Vocabulary (личный словарь) | [vocab.md](vocab.md) |
| Settings | [settings.md](settings.md) |

## Hard Restrictions

When the task is only about creating or wiring a screen:

- Do not invent or expand i18n catalogs unless the user explicitly asks.
- Do not write tests unless the user explicitly asks for screen/widget tests.
- Do not install packages (no go_router / Riverpod / Bloc unless AGENTS or an
  OpenSpec change already adopts them).
- Do not start long-lived `flutter run` unless needed for verification; when
  verifying, prefer `flutter analyze` from the app package root.
- Do not fix IDE diagnostics unrelated to the change.
- Do not hand-edit `lib/src/rust/**` or `rust/src/frb_generated.rs`.
- Do not reimplement parse / tokenize / lemma / dictionary / translate in Dart.
- Do **not** invent screens outside product surfaces (upload, reader, vocab, settings).

## Standard Screen Wiring

1. Add the screen under `lib/features/<area>/` (e.g.
   `lib/features/reader/reader_screen.dart`). Optional local widgets beside it.
2. Register navigation from the app shell ([shell.md](shell.md)) — named route,
   `Navigator`, or adopted router package. Do not leave a new screen unreachable.
3. Keep shared chrome (app bar brand, primary destinations) in `lib/app/` —
   details in [shell.md](shell.md). Do not duplicate a second shell inside a
   feature screen.
4. Prefer: **screen → orchestration → generated FRB API → Rust**. Widgets take
   props/callbacks; leaf widgets should not spray FRB calls.
5. Wrap the body in `Scaffold` (or a shell-provided scaffold slot). Match nearby
   Material patterns once product screens exist.

If the route name or screen placement is unclear, ask the user before editing.

## Current vs Target Screens

### Scaffold today (`lib/main.dart`)

| Widget | Role |
|--------|------|
| `MyApp` | `MaterialApp` + demo theme |
| `MyHomePage` | Demo TextField / «Вызвать Rust» / `processText` + `addNumbers` |

No router, no product screens. **Do not extend `MyHomePage` for product UX** —
replace it with the target map below as features land.

### Target screen map

| Area | Suggested path | Notes |
|------|----------------|-------|
| Shell | `lib/app/` | `MaterialApp`, theme, nav host — [shell.md](shell.md) |
| Upload / list | `lib/features/upload/` | File pick + saved subtitle list — [upload.md](upload.md) |
| Reader | `lib/features/reader/` | Cues, tap/selection, highlight, word card — [reader.md](reader.md) |
| Split view | `lib/features/reader/` or `lib/features/split/` | Original + translation, sync scroll — [reader.md](reader.md) |
| Vocabulary | `lib/features/vocab/` | Dictionary list / entry — [vocab.md](vocab.md) |
| Settings | `lib/features/settings/` | Languages; later optional AI — [settings.md](settings.md) |

Entry stays thin:

```text
lib/main.dart          → RustLib.init() + runApp(TranslationReaderApp)
lib/app/…              → MaterialApp + shell / routes
lib/features/<area>/   → *Screen + local widgets + optional controller
lib/src/rust/          → GENERATED FRB (never hand-edit)
```

## Screen Composition

**Screen owns the view; Rust owns parse / NLP / dictionary / offline translate.**

- Screens compose widgets and call a thin orchestration layer (screen state,
  `ChangeNotifier`, or small service — package choice not mandated).
- Import generated API as
  `package:my_translator_app/src/rust/api/….dart` only after the Rust surface
  exists; until then, keep UI wired to clear TODO call sites, not fake Dart
  parsers.
- Word card / highlight chrome may live as local reader widgets or move to
  `lib/widgets/` when reused.

Examples (target):

- Upload / list — pick file → Rust parse → open Reader ([upload.md](upload.md)).
- Reader — render cues/tokens from Rust; tap → word card; lemma highlight from
  dictionary lookup data ([reader.md](reader.md)).
- Split — two panes, sync scroll; translation lines from Rust, not Dart
  string hacks ([reader.md](reader.md)).
- Vocab — list/remove entries via Rust dictionary API ([vocab.md](vocab.md)).
- Settings — languages / prefs; persist via Rust/SQLite when available
  ([settings.md](settings.md)).

Do not put a second app shell inside a feature screen.

## Navigation Patterns

- Prefer a single nav host in `lib/app/` (bottom nav / drawer / rail — product
  choice; keep destinations: Upload/List, Vocab, Settings; Reader/Split are
  usually pushed from a document).
- Named routes or typed route objects once a router exists; until then,
  `Navigator.of(context).push` with a clear screen widget is fine.
- Deep links / web URL strategy — only when an OpenSpec change asks for them.
- Keep catch-all / unknown routes redirecting to the library (upload list).

## Domain Map

| Domain | Where |
|--------|--------|
| App shell / nav | [shell.md](shell.md) |
| Upload + subtitle list | [upload.md](upload.md) |
| Reader / word card / highlight / split | [reader.md](reader.md) |
| Vocabulary | [vocab.md](vocab.md) |
| Settings | [settings.md](settings.md) |
| Folder / dependency rules | `client-work-with-structure` |
| Parse / lemma / dict / translate | Rust `rust/src/` + thin `rust/src/api/` |

## Verification and Final Response

For screen-only tasks, verify by reviewing the changed screen(s), shell/nav
wiring, and that FRB imports (if any) match generated API. Prefer
`flutter analyze` when the change compiles UI. In the final response, state
which files changed and explicitly mention that tests, package installs, and
IDE diagnostics were not run or used when this skill forbids them.
