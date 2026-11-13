# Offline translate — session warm-up + responsive cold path

Read with the [core translate skill](SKILL.md). Shell start hook lives in
[work-with-pages/shell.md](../../client/work-with-pages/shell.md).

Canon: OpenSpec `translate/offline` (TO-09…TO-14). Planning detail while open:
`openspec/changes/warm-offline-translate-startup/` (design D1–D3, D6, **D8 revised**, **D13**).

## Start hook (product shell, fullscreen gate)

`AppShell` blocks the product shell (Library/Vocab, bottom nav) behind a
**fullscreen loading gate** until `warmUpOfflineTranslate()` resolves —
**not** the DB bootstrap spinner in `TranslationReaderApp`, not before first
paint, not inside `RustLib.init` / `init_app` (design D1). Gate **stays**
(explore: do not remove loader / quiet-only default).

| Rule | Detail |
|------|--------|
| Call once | `await _runWarmUpGate()` (awaits `warmUpOfflineTranslate()`, then flips gate state) started from `initState` |
| Gate UI | Fullscreen `Scaffold` + `CircularProgressIndicator`; Library/Vocab/`NavigationBar` not built while shown (TO-09) |
| Quiet dismiss on fail | Warm-up swallows errors — gate still dismisses, **no** SnackBar from the gate; later Translate shows kinds (TO-13) |
| Escape hatch | `enableOfflineTranslateWarmUp: false` **and/or** injectable `offlineTranslate` skips the gate **entirely** — shell is interactive immediately (required for widget tests) |

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

## Responsive native cold path (TO-11 / TO-14, design D13)

Product ensure MUST **not** pass full ONNX through Dart→FRB `Vec<u8>`:

1. Yield a frame before heavy work (`Future<void>.delayed(Duration.zero)`).
2. Resolve app-local model dir via `path_provider`
   `getApplicationSupportDirectory` + subdir `opus-mt-en-ru`
   (`offline_translate_native.dart`).
3. **Reuse** when all required **flat basenames** look present
   (`tokenizer.json`, `encoder_model_uint8.onnx`, `decoder_model_uint8.onnx` —
   no `onnx/` subdir on disk). Size heuristic: ONNX ≥ ~1MiB, tokenizer ≥ 1KiB.
4. Otherwise load assets **sequentially** (async `bundle.load` from asset paths
   that still include `onnx/`) and **chunked disk write + yields** (~512KiB
   chunks; no `Isolate.run` on typical AssetBundle / Immutable buffer views).
5. Call FRB `initOfflineTranslateFromDir(modelDir: …)` (async Normal / worker
   pool) — Rust `OpusMtEnRu::from_model_dir` / ORT `commit_from_file`. Bytes
   `initOfflineTranslate` MAY stay for tests / compat only.

Lazy init on first Translate remains the fallback while warm-up is still in
flight; button loading covers the wait (TO-10).

### Native test hooks (unit)

In `offline_translate_native.dart` (reset via
`debugResetOfflineTranslateNativeEnsure`):

| Hook / counter | Role |
|----------------|------|
| `debugModelDirOverride` | Avoid `path_provider` in unit tests |
| `debugInitFromDirOverride` | Fake path FRB init (no ORT) |
| `debugOfflineTranslatePathInitStarts` | Count product path inits |
| `debugOfflineTranslateExtractPasses` | Count extract passes that wrote ≥1 file |
| `debugExtractOfflineTranslateModelAssets` / `debugWriteOfflineTranslateAssetFileChunked` | Direct extract / chunked-write tests |

## Tests (agent expectations)

- Facade unit: shared Future race / ready no-op / extract→path /
  reuse / chunked write —
  `test/data/offline_translate_native_ensure_test.dart`,
  `test/data/offline_translate_web_ensure_test.dart`.
- Shell gate: `enableOfflineTranslateWarmUp: false` or inject translate skips
  the gate entirely; production gate blocks shell until ready (TO-09) and
  dismisses quietly on failure (TO-13) — `test/features/shell_test.dart`.
- Mid-init Translate loading → gloss/err — Vocab / Reader widget tests (TO-10).
- Rust TO-14: `init_offline_translate_from_dir` success + missing file →
  `EngineUnavailable` (`rust/src/translate/tests.rs`).

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Warm-up from `main` / DB spinner | `AppShell` fullscreen gate after Drift open |
| Quiet-only background warm-up (no gate) | Fullscreen gate blocks shell until ready/fail (D1 / TO-09) |
| Product path = full ONNX bytes→FRB | Extract → path/dir init (D8 revised / D13) |
| Per-instance `_initFuture` only | Process-wide shared ensure |
| SnackBar on warm-up fail | Quiet; SnackBar on Translate |
| `Isolate.run` on AssetBundle views | Chunked disk write + yields on UI isolate |
| Second Web pipeline on warmup∥translate | One `translatorPromise` + bridge `warmup` |
