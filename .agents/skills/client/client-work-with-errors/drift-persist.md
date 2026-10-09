# Drift / repository persistence errors

Read with the [core errors skill](SKILL.md). Persist layer patterns:
`work-with-drift`.

## Sources

| Source | Typical type | User-facing rule |
|--------|--------------|------------------|
| DB open / readiness fail | any from `openAppDatabase()` | Blocking screen on `TranslationReaderApp` with **retry** (not fatal InitFailureApp) |
| Validation (empty name/lemma/…) | `LibraryValidationException` / `DictionaryValidationException` | Short message from `exception.message` (already user-safe) |
| Missing row / not found | same validation types or null | Inline / SnackBar: not found; keep list |
| Constraint / SQLite I/O | Drift / sqlite3 exceptions | Generic “Couldn’t save. Try again.” + `debugPrint` details |
| File pick cancel | — | **Not** an error |
| Import read failure | platform / I/O | “Couldn’t read the file” |

Do **not** treat Drift failures as FRB `AnyhowException` / panic mapping.

## Surfaces

| Failure | Surface |
|---------|---------|
| DB open after successful `RustLib.init` | Full-body error + retry in `TranslationReaderApp` (shell not shown) |
| Folder/document/dict mutate fail | SnackBar or inline on Library / Vocab |
| List/load fail | Inline error + retry; empty list ≠ error |
| Destructive delete confirm | `AlertDialog` confirm; failure after confirm → SnackBar/inline |

## Orchestration shape

Same as FRB actions: clear `error`, `try`/`catch`/`finally`, `mounted` after
`await`. Catch validation types **before** bare `catch`.

```dart
try {
  await dictionary.add(lemma: lemma, translation: translation);
} on DictionaryValidationException catch (e) {
  error = e.message;
} catch (e, st) {
  assert(() {
    debugPrint('dict persist: $e\n$st');
    return true;
  }());
  error = 'Couldn’t save. Try again.';
}
```

## Do not

- Open product shell when DB open failed (unless retry succeeds).
- Dump sqlite/Drift stack traces into UI.
- Add Rust/FRB storage API to “fix” persist errors — fix Drift / repositories.
