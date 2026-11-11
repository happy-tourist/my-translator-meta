# Offline translate — session warm-up + responsive cold path

Read with the [core translate skill](SKILL.md). Shell start hook lives in
[work-with-pages/shell.md](../../client/work-with-pages/shell.md).

Canon: OpenSpec `translate/offline` (TO-09…TO-12). Planning detail while open:
`openspec/changes/warm-offline-translate-startup/` (design D1–D3, D6, D8).

## Start hook (product shell)

Quiet background warm-up runs after the **first post-frame** of `AppShell`
(Library/Vocab visible after Drift open) — **not** the DB bootstrap spinner in
`TranslationReaderApp`, not before first paint, not inside `RustLib.init` /
`init_app`.

| Rule | Detail |
|------|--------|
| Call once | `unawaited(warmUpOfflineTranslate())` from post-frame callback |
| Quiet fail | Warm-up swallows errors — **no** SnackBar; later Translate shows kinds |
| Escape hatch | `enableOfflineTranslateWarmUp: false` **and/or** injectable `offlineTranslate` skips production warm-up (required for widget tests) |

## Shared ensure (process-wide)

Public Dart seam (via `offline_translate_platform.dart` → native \| web):

| API | Role |
|-----|------|
| `ensureOfflineTranslateReady()` | Join / start one in-flight init; Translate awaits this |
| `warmUpOfflineTranslate()` | Same ensure; catches errors (quiet) |

**MUST** be **module-level / process-wide** (static slot), **not** only on one
`createOfflineTranslate()` instance — Reader/Vocab/tests may create a second
facade; shell warm-up and Translate MUST converge on one load.

| Platform | Shared slot | Ready / dedupe |
|----------|-------------|----------------|
| **Native** | `_sharedInitFuture` in `offline_translate_native.dart` | `isOfflineTranslateReady` → no-op re-init |
| **Web** | `_sharedEnsureFuture` in `offline_translate_web_ensure.dart` (no `dart:js_interop` — VM-testable) + bridge `warmupPromise` | Worker module `translatorPromise` shared by `warmup` + `translate` ops |

Web layout:

```text
lib/data/translate/
  offline_translate_web.dart          # js_interop bridge + _runWarmup
  offline_translate_web_ensure.dart   # process-wide Future; unit-tested on VM
web/
  translate_bridge.js                 # OfflineTranslate.warmup() + translate()
  translate_worker.js                 # op 'warmup' | 'translate'; one pipeline
```

## Responsive native cold path (TO-11)

Large AssetBundle → FRB init must not freeze the UI isolate:

1. Yield a frame before heavy work (`Future<void>.delayed(Duration.zero)`).
2. Load assets **sequentially** (async `bundle.load` per file).
3. Own large views with **chunked copy + yields** — do **not** `Isolate.run` on
   typical AssetBundle / Immutable buffer views (often not sendable; shipping
   ~100MB+ still stalls the UI isolate).
4. `initOfflineTranslate` stays normal async FRB (worker pool).

Lazy init on first Translate remains the fallback while warm-up is still in
flight; button loading covers the wait (TO-10).

## Tests (agent expectations)

- Facade unit: shared Future race / ready no-op —
  `test/data/offline_translate_native_ensure_test.dart`,
  `test/data/offline_translate_web_ensure_test.dart`.
- Shell: `enableOfflineTranslateWarmUp: false` or inject translate —
  `test/features/shell_test.dart`.
- Mid-init Translate loading → gloss/err — Vocab / Reader widget tests (TO-10).

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Warm-up from `main` / DB spinner | First `AppShell` post-frame |
| Per-instance `_initFuture` only | Process-wide shared ensure |
| SnackBar on warm-up fail | Quiet; SnackBar on Translate |
| `Isolate.run` on AssetBundle views | Chunked own-copy + yields on UI isolate |
| Second Web pipeline on warmup∥translate | One `translatorPromise` + bridge `warmup` |
