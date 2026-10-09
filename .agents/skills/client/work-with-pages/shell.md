# App shell (`lib/app/`)

Read with the [core pages skill](SKILL.md) when changing `MaterialApp`, theme,
primary navigation, or shared chrome that every product screen sits inside.

## Target composition

Scaffold today mounts UI from `lib/main.dart` (`MyApp` → `MyHomePage`). Target:

```text
main.dart
  → await RustLib.init()
  → runApp(TranslationReaderApp)          # lib/app/

lib/app/
  → MaterialApp (title: Translation Reader, theme, home / router)
  → shell scaffold: AppBar / NavigationBar|Drawer|NavigationRail
  → Navigator / Router outlet for feature screens
```

Suggested files (create only what the change needs):

| File | Role |
|------|------|
| `lib/app/app.dart` | `MaterialApp` + theme |
| `lib/app/shell.dart` (or similar) | Primary destinations + body outlet |
| `lib/main.dart` | Init + `runApp` only — no feature markup |

## Primary destinations (target)

| Destination | Opens | Notes |
|-------------|-------|-------|
| Library / Upload | [upload.md](upload.md) | Default home after scaffold removal |
| Vocabulary | [vocab.md](vocab.md) | Personal dictionary |
| Settings | [settings.md](settings.md) | Languages; later optional AI |

**Reader** and **Split view** are session screens pushed from a selected
subtitle document — not usually top-level tabs. Back from Reader returns to
the library list.

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
- Open Reader with `Navigator.push` (or equivalent) carrying a document id /
  handle from Rust — not a full parsed tree owned only in Dart.

When a router is adopted later, keep the same destinations and the same
«Reader is pushed from library» model; register routes in `lib/app/`, not
scattered across features.

## Do not

- Extend `MyHomePage` demo as the long-term home.
- Duplicate AppBar brand + bottom nav on every feature screen.
- Port web layout shells, breadcrumbs, or brand-logo leave-mode chrome.
