# App shell (`lib/app/`)

Read with the [core pages skill](SKILL.md) when changing `MaterialApp`, theme,
primary navigation, or shared chrome that every product screen sits inside.

## Current composition

```text
main.dart
  → await RustLib.init()   # or fatal _InitFailureApp
  → runApp(TranslationReaderApp)          # lib/app/app.dart

lib/app/app.dart
  → open Drift → loading / error+retry / AppShell(database)

lib/app/shell.dart
  → NavigationBar: Library | Vocabulary
  → inject Folder/Document/Dictionary repositories
```

| File | Role |
|------|------|
| `lib/app/app.dart` | `MaterialApp` + Drift open + theme host |
| `lib/app/shell.dart` | Primary destinations + body outlet |
| `lib/main.dart` | Init + `runApp` only — no feature markup |

## Primary destinations

| Destination | Opens | Notes |
|-------------|-------|-------|
| Library / Upload | [upload.md](upload.md) | Default home (index 0) |
| Vocabulary | [vocab.md](vocab.md) | Personal dictionary |
| Settings | [settings.md](settings.md) | Languages; later optional AI — not a tab yet |

**Reader** and **Split view** are session screens pushed from a selected
subtitle document — not top-level tabs. `Navigator.push` from Library covers
the shell (bottom nav hidden on the Reader route); back pops to the same
Library folder.

## Chrome rules

- One shared shell: brand/title, theme, and primary nav live here.
- Feature screens provide `Scaffold` body content (or fill a shell-provided
  slot). Do not nest a full competing nav shell inside upload/reader/vocab.
- Document title / `MaterialApp.title`: **Translation Reader** (replace demo
  «Flutter + Rust Demo»).
- Theme: define once in `lib/app/`; screens consume `Theme.of(context)`.
- No account/auth gates, staff roles, or leave-mode chrome (out of scope).
- No backend session; offline-first. Shell does not call remote APIs.

## Navigation wiring

Until AGENTS / OpenSpec adopts a router package:

- Shell holds an index (or equivalent) for Library / Vocab / Settings.
- Open Reader with `Navigator.push` after successful FRB parse; pass cues (+
  document title) into `ReaderScreen`. Bytes come from Drift / content store;
  do not keep a parallel Dart subtitle parser.

When a router is adopted later, keep the same destinations and the same
«Reader is pushed from library» model; register routes in `lib/app/`, not
scattered across features.

## Do not

- Dump feature markup into `main.dart`.
- Duplicate AppBar brand + bottom nav on every feature screen.
- Port web layout shells, breadcrumbs, or brand-logo leave-mode chrome.
