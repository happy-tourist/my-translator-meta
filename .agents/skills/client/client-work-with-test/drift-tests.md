# Drift / repository tests

Read with the [core test skill](SKILL.md). Persist canon: `work-with-drift`.

## Harness

| Helper | Path | Use |
|--------|------|-----|
| `createTestDatabase()` | `test/support/test_database.dart` | In-memory `NativeDatabase.memory()` |
| `createFileTestDatabase(File)` | same | Close/reopen durability |
| `MemoryDocumentContentStore` | `lib/data/library/document_content_store.dart` | Import without IO/Web |

```dart
final db = createTestDatabase();
addTearDown(db.close);
final folders = FolderRepository(db);
```

## What to cover

| Layer | Prefer | Avoid |
|-------|--------|-------|
| Repository unit | CRUD, validation, cascade delete, lemma lookup, reopen | Pumping full `MaterialApp` for SQL rules |
| Widget | Screen + injected repos/fakes; empty/error/loading | Real disk/Web wasm in widget tests |
| Persist reopen | File-backed DB: write → close → reopen → assert rows | Relying only on memory DB for durability SC |

Existing anchors: `test/data/*_repository_test.dart`,
`test/data/persistence_reopen_test.dart`,
`test/data/schema_migration_test.dart` (schemaVersion + upgrade paths),
`test/features/library_screen_test.dart`,
`test/features/vocabulary_screen_test.dart`,
`test/features/shell_test.dart`.

## FRB vs Drift in tests

- Library / Vocab persist tests **do not** need `RustLib.init` if the widget
  under test takes repositories / `AppDatabase` directly.
- Keep FRB fakes for compute-only screens (see [frb-boundary.md](frb-boundary.md)).
- Do not hand-edit `lib/src/rust/**` or `*.g.dart` for test doubles.

## Commands

From app root:

```bash
flutter test test/data
flutter test test/features
```
