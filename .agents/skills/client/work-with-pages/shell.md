# App shell (`lib/app/`)

Read with the [core pages skill](SKILL.md) when changing `MaterialApp`, theme,
primary navigation, or shared chrome that every product screen sits inside.

## Current composition

```text
main.dart
  → await RustLib.init()   # or fatal _InitFailureApp
  → runApp(TranslationReaderApp)          # lib/app/app.dart

lib/app/app.dart
  → open Drift → soft-fail stub→morph migrate → loading / error+retry / AppShell
  → MaterialApp(theme: AppTheme.light, darkTheme: AppTheme.dark, themeMode: system)

lib/app/shell.dart
  → fullscreen loading gate until offline-translate ensure/warm-up settles
  → NavigationBar: Library | Vocabulary (not built/interactive until gate dismissed)
  → inject Folder/Document/Dictionary repos + LemmaKeysForSurfacesFn
```

| File | Role |
|------|------|
| `lib/app/app.dart` | `MaterialApp` + Drift open + post-open migrate + theme host |
| `lib/app/theme.dart` | `AppTheme` + `ReaderColors` (`lemmaHighlight` light/dark) |
| `lib/app/shell.dart` | Primary destinations + body outlet; offline-translate fullscreen loading gate |
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

## Offline translate fullscreen loading gate

Engine warm-up gates the **product shell** behind a fullscreen loader —
not the Drift-open spinner in `TranslationReaderApp`, not inside
`RustLib.init` / `init_app` (design D1, revises the earlier quiet-only hook):

- After `AppShell` builds (Drift already open), run
  `await warmUpOfflineTranslate()` and keep a fullscreen `Scaffold` +
  `CircularProgressIndicator` on screen until it resolves. **Library/Vocab and
  the bottom `NavigationBar` are not built** while the gate is up (TO-09) —
  this is stronger than "quiet background": the shell is genuinely blocked
  until ready/fail. Native cold path is extract→path init (not full
  ONNX bytes→FRB); still yield so the gate spinner can paint — see
  [warmup.md](../../server/work-with-translate/warmup.md).
- Success **or** failure both dismiss the gate (`setState` flips a `_gateDone`
  flag) and show the product shell; failure MUST NOT show a SnackBar from the
  gate itself (TO-13) — Translate reports the kind later.
- Facade seam: `warmUpOfflineTranslate` / `ensureOfflineTranslateReady` (via
  `offline_translate_platform.dart`); share **one process-wide in-flight Future**
  with Translate (native + Web). Full rules:
  [work-with-translate/warmup.md](../../server/work-with-translate/warmup.md).
- **Tests:** `AppShell` MUST expose an escape hatch (`enableOfflineTranslateWarmUp:
  false` and/or skip the gate entirely when an injectable `offlineTranslate` is
  passed) so `shell_test` and other mounts are interactive immediately, with no
  gate and no model asset / FRB load (`test/features/shell_test.dart` TO-09 /
  TO-13 escape-hatch cases).

## Navigation wiring

Until AGENTS / OpenSpec adopts a router package:

- Shell holds an index (or equivalent) for Library / Vocab / Settings.
- Pass `DictionaryRepository` into both `VocabularyScreen` and `LibraryScreen`
  (Library forwards it to `ReaderScreen` for Add/edit sheets).
- Pass injectable `LemmaKeysForSurfacesFn` into Vocab (manual add morph keys);
  default = FRB `lemmaKeysForSurfaces`. Reader gets `MarkVocabTokensFn` via
  Library → Reader ctor (tests inject fakes).
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
