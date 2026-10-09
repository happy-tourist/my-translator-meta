---
name: client-work-with-errors
description: >-
  Use when adding, changing, reviewing, or debugging error handling in
  my_translator_app (Translation Reader) Flutter UI — FRB / Rust call
  failures, RustLib.init failures, file pick/parse errors, empty vs error
  states, SnackBar / dialog / inline patterns, async surfacing after await,
  or mapping exceptions so Rust internals are not shown to users. Flutter
  Material surfaces; in-process FRB exceptions.
---

# Work With Errors

Use this skill when working with UI and FRB failure paths under `lib/` in
**my_translator_app** (product: **Translation Reader**).

Stack (from `{app}/AGENTS.md`): Flutter / Dart Material, Rust core via
**flutter_rust_bridge** 2.13 (`lib/src/rust/` generated). Heavy work fails in
Rust and surfaces as Dart exceptions / results on the FRB call.

Failures are orchestration-local `error` + Material surfaces
(SnackBar / inline / dialog) — not a global toast plugin bus.

## Core Model

Error handling is **orchestration-local**, not a second global toast bus:

- FRB I/O (parse, tokenize, dictionary, translate, settings) lives in a thin
  orchestration layer next to the feature (screen state, controller/notifier,
  or small service) — see `client-work-with-structure`.
- Failed actions set a UI-facing `String? error` (or typed UI error enum) to a
  **user-safe** message. Never assign raw `e.toString()`, panic text, or
  anyhow backtraces to what the user reads.
- Clear `error` at the start of a new attempt.
- Loading flags clear in `finally` (or equivalent).
- Screens bind display to that orchestration error (inline banner / body text,
  SnackBar, or dialog — see Display Surfaces). Do not invent a parallel local
  `error` that duplicates the same failure.
- After every `await`, check `mounted` (or equivalent notifier lifecycle)
  before `setState`, `ScaffoldMessenger`, or navigation.

Do not invent global `FlutterError.onError` UI popups for product FRB flows,
or a parallel toast-plugin bus. Prefer orchestration `error` + Material
surfaces below.

## Exception Sources

| Source | Typical Dart type | User-facing rule |
|--------|-------------------|------------------|
| Rust `Result`/anyhow via FRB | `AnyhowException` (`message`) | Map known codes/prefixes → short copy; else generic “Couldn’t complete that” |
| Rust panic across FRB | `PanicException` | **Never** show panic body; generic + debug log |
| Other FRB bridge issues | `FrbException` subtypes | Generic; log details |
| `RustLib.init()` failure | any thrown from init | Blocking fatal UI — app cannot run product screens |
| File picker cancel | — | **Not** an error — return silently |
| Pick/read permission / I/O | `PlatformException` / `FileSystemException` / similar | Short permission or “Couldn’t read file” copy |
| Parse / format reject (Rust) | usually `AnyhowException` | “Unsupported or invalid subtitle file” (+ format hint if API provides one) |
| Empty list / no document | — | **Empty state**, not SnackBar error |

Import FRB exception types from `package:flutter_rust_bridge/flutter_rust_bridge_for_generated.dart`
(or the package export your generated API already uses). Prefer
`on AnyhowException` / `on PanicException` before a bare `catch`.

When Rust gains typed error enums on the FFI surface, map those first; do not
keep scraping free-form panic strings for UX.

## Init Failures (`RustLib.init`)

`main()` must `await RustLib.init()` before `runApp` of the product app
(`{app}/AGENTS.md`). Init failure is **fatal** for product UI:

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  try {
    await RustLib.init();
  } catch (e, st) {
    assert(() {
      debugPrint('RustLib.init failed: $e\n$st');
      return true;
    }());
    runApp(InitFailureApp(message: userFacingInitFailure(e)));
    return;
  }
  runApp(const MyApp());
}
```

- `InitFailureApp`: minimal Material screen — title + short message + no
  reader/upload chrome that would call Rust.
- Do **not** proceed to feature screens if init failed.
- Do **not** surface stack traces or WASM/FFI linker text in the UI.

## Orchestration Call Pattern

Every mutating / loading FRB action should follow the same shape:

```dart
loading = true;
error = null;
notifyListeners(); // or setState

try {
  final doc = await parseSubtitles(/* … */);
  // apply success to UI models
} on AnyhowException catch (e) {
  error = mapFrbUserMessage(e.message);
} on PanicException catch (e) {
  assert(() {
    debugPrint('Rust panic: ${e.message}');
    return true;
  }());
  error = genericFailureMessage;
} catch (e, st) {
  assert(() {
    debugPrint('Unexpected: $e\n$st');
    return true;
  }());
  error = genericFailureMessage;
} finally {
  loading = false;
  notifyListeners();
}
```

Applies to (target): subtitle parse, open document, tokenize/lemmatize for a
line, dictionary add/list/remove, offline translate, settings load/save.

- Re-throw only when the **caller** must skip a success path (e.g. navigate
  into reader only on successful parse). Otherwise keep the error in
  orchestration and let the screen bind it.
- Do not leave loading stuck if the widget disposes mid-await — clear in
  `finally`, and gate UI updates with `mounted` / `!disposed`.

### Mapping helper (keep thin)

```dart
String mapFrbUserMessage(String raw) {
  // Prefer stable codes from Rust API when available.
  if (raw.contains('unsupported_format') || raw.contains('parse')) {
    return 'Couldn’t open that subtitle file. Try SRT, VTT, or ASS/SSA.';
  }
  if (raw.contains('empty')) {
    return 'That file has no subtitle lines.';
  }
  return 'Something went wrong. Try again.';
}
```

Tune copy to product language; never paste Rust crate paths, `at `, or
backtrace frames into the string.

## File Pick And Parse

| Step | Failure | UX |
|------|---------|-----|
| User cancels picker | — | Silent return; clear any prior *in-flight* loading only |
| Permission denied | platform exception | SnackBar or inline: need file access |
| Read bytes/path fails | I/O | “Couldn’t read the file” |
| Rust parse rejects | FRB exception | Inline error on upload screen + keep prior document if any |
| Parse OK but zero cues | empty result | Empty state on reader/upload (“No lines found”), not a red panic |

Hand path/bytes to Rust; do not parse SRT/VTT/ASS in Dart. Flutter owns picker
UX and error chrome only.

## Display Surfaces

Prefer **one** primary surface per failure. Match severity:

| Severity | Surface | When |
|----------|---------|------|
| Fatal | Full-screen / dedicated `InitFailureApp` | `RustLib.init` failed |
| Blocking choice | `AlertDialog` / modal | Destructive confirm; rare hard blocks that need an explicit dismiss |
| Persistent until next try | Inline error on the screen (Material `Banner`, colored `Text`, or error slot above list) | Upload parse fail, vocab load fail — user stays on page |
| Transient | `ScaffoldMessenger` → `SnackBar` | Short action fail (add-to-vocab, translate tap) where the main view still works |
| No data | Empty-state widget (illustration/title/CTA) | No files, empty dictionary, no open document |

### SnackBar (transient)

```dart
if (!mounted) return;
ScaffoldMessenger.of(context).showSnackBar(
  SnackBar(content: Text(errorMessage)),
);
```

Use for one-shot actions. Do not also keep a duplicate permanent banner for the
same event unless product design asks for both.

### Inline / banner (persistent)

Bind to orchestration `error` on the owning screen (upload, reader, vocab) —
persistent inline feedback via Material widgets (`Banner`, body text, or
similar), not a global toast bus.

Clear when starting a new attempt or on explicit dismiss if the UI has an OK
action.

### Dialog

Reserve for confirms and true blockers. Do not wrap every FRB failure in
`showDialog`. Never use dialogs to dump `PanicException.message`.

### Empty vs error

| Situation | Treat as |
|-----------|----------|
| Vocab list length 0 after successful load | Empty state + “Add from reader” CTA |
| Vocab load threw | Error + retry |
| No subtitle opened yet | Empty / prompt to upload |
| Open failed | Error on upload/reader |

## Async Surfacing Checklist

1. `error = null` before the attempt.
2. `try` / typed `on` / `catch` / `finally` around the FRB (or pick) call.
3. Map to user-safe copy; log internals with `debugPrint` / crash reporter only.
4. After `await`: if `!mounted` (State) or disposed notifier → **return** (still
   run `finally` for flags).
5. Show via the chosen surface once; do not `print` alone and leave the UI idle.
6. Success path must clear stale error if not already cleared at start.

## When Orchestration Catch Vs Screen Catch

| Situation | Pattern |
|-----------|---------|
| Parse / open subtitle | Orchestration sets `error`, optionally re-throws; screen skips navigate-to-reader on failure |
| Reader tap → translate / lemma | Orchestration or screen method sets error; SnackBar or card error slot |
| Add/remove dictionary entry | Orchestration sets error; SnackBar or inline on vocab/reader |
| List vocab / reopen last doc | Orchestration catch sets `error`, keeps list empty or prior data; **no** silent swallow |
| File picker cancel | No error, no SnackBar |
| Leave / dispose mid-request | Swallow UI update; do not set a new user error for “cancelled by dispose” |
| `RustLib.init` | Only in `main` → fatal app; not a feature SnackBar |
| Best-effort side effect (e.g. analytics later) | May swallow only if product says so; never hide user-visible FRB failures this way |

## Patterns And Examples

### 1. Upload parse + navigate on success

```dart
// orchestration
Future<SubtitleDoc> openSubtitles(String path) async {
  loading = true;
  error = null;
  try {
    return await parseSubtitles(path: path); // generated FRB
  } on AnyhowException catch (e) {
    error = mapFrbUserMessage(e.message);
    rethrow;
  } on PanicException catch (e) {
    debugPrint('panic: ${e.message}');
    error = genericFailureMessage;
    rethrow;
  } finally {
    loading = false;
  }
}
```

```dart
// UploadScreen
try {
  final doc = await controller.openSubtitles(path);
  if (!mounted) return;
  // navigate / push reader with doc
} catch (_) {
  // message already in controller.error → inline banner
}
```

### 2. Tap-to-translate (transient SnackBar)

```dart
Future<void> translateTap(String lemma) async {
  try {
    final t = await translateWord(lemma: lemma);
    if (!mounted) return;
    showWordCard(t);
  } on AnyhowException catch (e) {
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(mapFrbUserMessage(e.message))),
    );
  } on PanicException catch (e) {
    debugPrint('panic: ${e.message}');
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('Translation failed. Try again.')),
    );
  }
}
```

### 3. Empty dictionary (not an error)

```dart
if (loading) return const CircularProgressIndicator();
if (error != null) return ErrorRetry(message: error!, onRetry: reload);
if (entries.isEmpty) {
  return const EmptyState(
    title: 'No saved words yet',
    subtitle: 'Tap a word while reading to add it.',
  );
}
// list…
```

### 4. Intentional non-error swallow

Only for: picker cancel, leave/dispose after unmounted await, or “already
closed” teardown. Document with a one-line comment. Do **not** use empty
`catch` on parse/translate/dict without setting `error` or navigating away.

## Common Mistakes

- Showing `e.toString()`, `AnyhowException(...)`, or panic bodies in the UI.
- `catch` with only `debugPrint` / `log` — user sees no feedback.
- Forgetting `mounted` after `await` → setState / SnackBar on defunct context.
- Skipping `finally` for `loading` (spinner stuck forever).
- Treating picker cancel as an error SnackBar.
- Using SnackBar for empty vocab / “no file open” (use empty state).
- Duplicating the same failure in controller `error` **and** a screen-local
  `error` ref with different copy.
- Calling FRB from many leaf widgets each with their own ad-hoc catch — keep
  one orchestration policy per feature.
- Proceeding to reader after failed parse because the screen ignored re-throw /
  `error`.
- Hand-editing generated `lib/src/rust/**` to “fix” error types — change
  `rust/src/api/` + codegen instead.

## Key Files (target)

| Path | Role |
|------|------|
| `lib/main.dart` | `RustLib.init` try/catch → product app or `InitFailureApp` |
| `lib/app/` | `MaterialApp`, theme; optional root messenger |
| `lib/features/upload/` | Pick/read errors + parse `error` inline |
| `lib/features/reader/` | Tap/translate/highlight failure surfaces |
| `lib/features/vocab/` | Load/add/remove errors + empty state |
| `lib/features/settings/` | Save/load preference failures |
| Feature `*_controller.dart` / screen state | Owns `error` / `loading` + FRB try/catch |
| `lib/src/rust/api/*.dart` | Generated call surface (do not hand-edit) |
| `{app}/AGENTS.md` | Init order, layering |

Scaffold today (`MyHomePage` + `processText`) has **no** error UI — add the
patterns above when replacing demo calls with product FRB flows; do not grow
the demo page into an error framework.

## Related Skills

- Layer placement / where orchestration lives → `client-work-with-structure`
- Flutter implementability of error UX claims → `align-implementability-front`
- Rust/API error contracts → `align-implementability-back`
- App canon → `{app}/AGENTS.md` via `docs/projects-map.md`

## Verification

When touching error handling, confirm:

- [ ] Orchestration clears `error` before retry; `loading` cleared in `finally`
- [ ] FRB failures mapped; no panic/anyhow dump in UI copy
- [ ] `mounted` / disposed checks after awaits before UI side effects
- [ ] Init failure is fatal full-screen, not a SnackBar on a half-started app
- [ ] Picker cancel and empty lists are not shown as errors
- [ ] One primary Material surface per failure (inline / SnackBar / dialog / fatal)
- [ ] Generated FRB files untouched; typed Rust errors preferred when API adds them
