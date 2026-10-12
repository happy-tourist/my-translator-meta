# Drift schema, open, codegen, Web

Read with [SKILL.md](SKILL.md).

## Database

| Piece | Path | Role |
|-------|------|------|
| Tables | `lib/data/database/tables.dart` | `Folders`, `Documents` (nullable `translationDocumentId`; `lastReadStartMs` / `lastReadAlignment` for resume), `DictionaryEntries` (`lemma` UNIQUE morph + nullable `surfaceKey`) |
| DB class | `lib/data/database/app_database.dart` | `@DriftDatabase`, `schemaVersion`, `driftDatabase(...)` |
| Generated | `lib/data/database/app_database.g.dart` | **GENERATED** — never hand-edit |
| Open helper | `lib/data/database/database_provider.dart` | `openAppDatabase()` + readiness `SELECT 1` |

Connection name today: `translation_reader`. Web options must point at:

```dart
web: DriftWebOptions(
  sqlite3Wasm: Uri.parse('sqlite3.wasm'),
  driftWorker: Uri.parse('drift_worker.js'),
),
```

## Startup order

1. `main()` → `await RustLib.init()` (fatal UI on failure — no Drift).
2. `TranslationReaderApp` → `openAppDatabase()` → loading → OK shell / error+retry.
3. After open (schema ≥4): best-effort `DictionaryRepository.migrateStubKeysIfNeeded`
   via injectable `LemmaKeysForSurfacesFn` (default FRB). Soft-fail: do **not**
   block shell / conflate with Drift open failure; retry next launch.
4. `AppShell(database: …)` builds repositories + optional `lemmaKeysForSurfaces`.

Do not open Drift before successful FRB init in product flow.

## Codegen

After changing tables / `@DriftDatabase` / DAOs:

```bash
# from app root
dart run build_runner build --delete-conflicting-outputs
```

Deps (canon): `drift`, `sqlite3`, `drift_flutter`, `sqlite3_flutter_libs`,
`path` / `path_provider`; dev: `drift_dev`, `build_runner`.

## Web assets

| Asset | Role |
|-------|------|
| `web/sqlite3.wasm` | sqlite3 WASM for Drift Web |
| `web/drift_worker.js` | Drift worker |

Commit both with Drift Web setup. Chrome run must use fixed `--web-port=5555`
plus COOP/COEP (same origin → same OPFS/IndexedDB). Random ports look like a
data wipe — see `work-with-env-deploy` and `{app}/AGENTS.md`.

## Migrations

Bump `schemaVersion` and add a Drift `MigrationStrategy` when altering tables.
Do not silently wipe user data in product builds without an OpenSpec decision.

**Present:** `schemaVersion` **4** —

- from `< 2`: nullable `Documents.translationDocumentId` (self-FK → `Documents.id`)
- from `< 3`: nullable `lastReadStartMs` + `lastReadAlignment` (reading resume)
- from `< 4`: nullable `DictionaryEntries.surfaceKey` (DDL only). Morph rewrite
  of legacy stub lemmas is **post-open** `migrateStubKeysIfNeeded` (FRB after
  `RustLib.init`), not inside Drift `onUpgrade` every cold start.

Regenerate `.g.dart` after table edits.
