---
name: work-with-drift
description: >-
  Use when adding or changing Flutter Drift user persistence in
  my_translator_app (Translation Reader): AppDatabase / tables, repositories
  (folders, documents, dictionary), DocumentContentStore IO vs Web, DB open in
  TranslationReaderApp, drift_dev codegen, or Web sqlite3.wasm / drift_worker.js.
  Not for Rust SQLite / FRB storage APIs.
---

# Work With Drift (user persistence)

Canon for **user data** in **my_translator_app**: **Drift** in Flutter `lib/data/`.
Product: **Translation Reader**. Paths relative to app root
(`../my_translator_app` from meta; see `docs/projects-map.md`).

**Compute** (parse / tokenize / lemmatize; offline translate on **native**) stays
in Rust via FRB. Web offline translate uses the JS exception behind
`lib/data/translate/` — still not Drift. Do **not** add `rusqlite` /
`rust/src/db` / FRB CRUD for folders, documents, or dictionary entries.

## When to Use

- Schema / tables / migrations (`AppDatabase`, `tables.dart`)
- Repositories (folders, documents, dictionary)
- Subtitle bytes store (`DocumentContentStore` + platform split)
- Open DB after `RustLib.init`, inject into shell / screens
- `build_runner` / `drift_dev` codegen (`.g.dart`)
- Web Drift assets and readiness

**Not this skill:** screen chrome → `work-with-pages`; folder roles →
`client-work-with-structure`; FRB compute → `work-with-frb`; Rust dict NLP
handoff → `server/work-with-dictionary`.

## Specialized Topics

| Topic | File |
|-------|------|
| Schema, open, codegen, Web assets | [schema.md](schema.md) |
| Repositories + content store | [repositories.md](repositories.md) |

## Layer Map

```text
lib/main.dart                 # RustLib.init → runApp(TranslationReaderApp)
lib/app/app.dart              # openAppDatabase → soft-fail stub→morph migrate → shell
lib/app/shell.dart            # repos + LemmaKeysForSurfacesFn into screens
lib/data/database/            # AppDatabase schema 5+, tables, database_provider, *.g.dart
lib/data/library/             # FolderRepository, DocumentRepository (+ video attach/JPEG), content store
lib/data/dictionary/          # DictionaryRepository + lemma_keys_for_surfaces_fn + exceptions
lib/data/video/               # FE extract facade (not Drift); Reader uses repos for JPEG cache
lib/features/upload/          # Library UI (calls repositories; best-effort video restore)
lib/features/vocab/           # Vocab UI (calls dictionary repository)
test/support/test_database.dart
test/data/*_repository_test.dart
web/sqlite3.wasm
web/drift_worker.js
```

## Hard Rules

1. Source of truth for library folders/documents and dictionary rows = **Drift**.
2. Screens talk to **repositories** (or injected fakes), not raw SQL in widgets.
3. Never hand-edit `*.g.dart` — change tables/DB → `dart run build_runner build`.
4. Inject `AppDatabase` / repositories for tests (`NativeDatabase.memory()`,
   `MemoryDocumentContentStore`).
5. Web: keep `web/sqlite3.wasm` + `web/drift_worker.js` in sync with Drift /
   sqlite3 setup; URIs match `DriftWebOptions` in `app_database.dart`.
6. Do not expose user CRUD over FRB “because Rust used to own SQLite”.

## Capability Anchors

| Capability | Persist path |
|------------|--------------|
| `library/folders` | `FolderRepository` + `Folders` table |
| `library/documents` | `DocumentRepository` + `Documents` + `DocumentContentStore` (+ video attach / JPEG frame cache when `reader/video-frames`) |
| `dictionary/entries` | `DictionaryRepository` + `DictionaryEntries` |
| Settings (later) | Drift when that change lands — same `lib/data/` pattern |

## Related Skills

- Structure / `lib/data` vs features → `client-work-with-structure`
- Library / Vocab UX → `work-with-pages` (`upload.md`, `vocab.md`, `shell.md`)
- Persist errors → `client-work-with-errors` + [drift-persist.md](../client-work-with-errors/drift-persist.md)
- Tests → `client-work-with-test` + [drift-tests.md](../client-work-with-test/drift-tests.md)
- Run/Web COOP-COEP → `work-with-env-deploy`
- App canon → `{app}/AGENTS.md`

## Verification

- [ ] New user tables/repos under `lib/data/`, not `rust/src/db`
- [ ] Codegen regenerated after schema edits; `.g.dart` not hand-patched
- [ ] Shell/screens receive DB/repos via constructor (testable)
- [ ] Native vs Web content store used correctly; Web assets present
- [ ] Repository/unit tests use `createTestDatabase()` / memory executor
