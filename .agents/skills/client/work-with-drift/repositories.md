# Repositories and document content store

Read with [SKILL.md](SKILL.md).

## Repositories

| Domain | Class | Path |
|--------|-------|------|
| Folders | `FolderRepository` | `lib/data/library/folder_repository.dart` |
| Documents | `DocumentRepository` | `lib/data/library/document_repository.dart` |
| Dictionary | `DictionaryRepository` | `lib/data/dictionary/dictionary_repository.dart` |

Typed validation errors (not FRB):

| Exception | Path |
|-----------|------|
| `LibraryValidationException` | `lib/data/library/library_exceptions.dart` |
| `DictionaryValidationException` | `lib/data/dictionary/dictionary_exceptions.dart` |

UI maps these to short copy (SnackBar / inline) — see
`client-work-with-errors` / `drift-persist.md`.

### Typical ops

- **Folders:** create (root/nested), list children, cascade delete. Before bulk
  document delete in a folder subtree, clear `translationDocumentId` self-FK
  links so SQLite FK checks do not fail.
- **Documents:** import bytes + metadata into a folder, list, delete; format
  helpers in `document_format.dart`; **load bytes for Reader open**
  (`loadBytes(Document)` — native via `storageKey`, Web via Drift `content` →
  `utf8.encode`).
- **Reader translation link (MVP `reader/split` / `library/documents`):**
  nullable `translationDocumentId` on original; `listInFolder` / watch must
  hide satellite translation rows; `attachTranslation`,
  `detachAndDeleteTranslation`, cascade on original delete,
  `loadLinkedTranslation` — see OpenSpec LD-12…LD-16. One translation slot per
  original (attach while linked → validation error; delete first).
- **Reading resume (LD-17 / RS-12):** `ReadingAnchor` (`startMs`, optional
  `alignment`); `saveReadingAnchor` / `loadReadingAnchor` on the **original**
  document only. Clamp alignment to `0.0…1.0`. Library open loads best-effort
  and passes `initialReadingAnchor` into `ReaderScreen`; persist failures must
  not block UI.
- **Video attachment (`add-reader-video-frames` / LD-18…LD-23):**
  `attachVideo` (`clearFrameCache` default true) / `detachVideo` /
  `getVideoAttachment` / `saveVideoLastFramePage` + frame JPEG
  `putFrameCache` / `getFrameCache` on the **original**; cascade on
  `deleteDocument`; Library open best-effort restore (fail must not block);
  durable link may die → keep cache + re-pick with `clearFrameCache: false`
  (LD-22); replace with different duration clears prior JPEG (LD-23).
  **`VideoAttachment.lastFramePage` / `saveVideoLastFramePage`:** API names keep
  the historical column (`videoLastFramePage`); **semantics = 0-based
  `last_second`** (timeline row index), not page-of-3. Validate `≥ 0` (not
  `≥ 1`). UI: `work-with-reader` / `video-frames.md`.
- **Shared parse seams (Library + Reader):**
  `lib/data/library/parse_subtitles_fn.dart` (`ParseSubtitlesFn`),
  `lib/data/library/parse_error_messages.dart` (`messageForParseError`).
- **Dictionary:** add (`lemma` + `surfaceKey` + translation + optional context),
  list, remove, lookup by lemma. Call site supplies key — single morph
  (`token.lemmaKey` / FRB `lemmaKeysForSurfaces`) or phrase
  `normalizeSurface(join)` — repository does **not** embed WordNet.
  `normalizeSurface` collapses internal whitespace (phrase parity with Rust).
  Lookup: exact `lemma` first; multi-word also accepts legacy rows that only
  differ by irregular whitespace. Schema 5+: `surface_key` column; post-open
  idempotent stub→morph `migrateStubKeysIfNeeded` (OpenSpec
  `add-reader-lemma-highlight` D4 / D5.1).

Keep business validation in repositories (trim, empty checks, parent exists).
Keep list/refresh orchestration in screens or thin controllers.

## DocumentContentStore

Subtitle **bytes** are not always a Drift BLOB-only story:

| Impl | Path | When |
|------|------|------|
| Abstract + memory fake | `document_content_store.dart` | Production interface + tests |
| IO (native) | `document_content_store_io.dart` | Sandbox file + `storageKey` |
| Web | `document_content_store_web.dart` | Inline UTF-8 `content` in Drift |
| Platform select | `document_content_store_platform.dart` | Conditional export |

`DocumentRepository` uses the store, then writes metadata rows. Tests inject
`MemoryDocumentContentStore` (content vs storageKey modes).

**Read path (Reader open):** `DocumentContentStore.load(storageKey)` (native
sandbox file; Web path usually reads Drift `content` via
`DocumentRepository.loadBytes` instead — Web store may throw if called). Do
**not** change Drift schema for load. Keep `MemoryDocumentContentStore.load`
so widget tests can open imported fixtures.

## Injection

Prefer constructing repos once in `AppShell` (or tests) and passing into
`LibraryScreen` / `VocabularyScreen`, and into `ReaderScreen` when split
translation load/persist/delete is in scope (`DocumentRepository` + optional
parse fn). Optional constructor overrides enable fakes without a global
service locator.
