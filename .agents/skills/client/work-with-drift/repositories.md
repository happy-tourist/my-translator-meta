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
  helpers in `document_format.dart` (`.srt`/`.vtt`/`.ass`/`.ssa`/`.epub` →
  stored `srt`/`vtt`/`ass`/`epub`); **load bytes for Reader open**
  (`loadBytes(Document)` — native via `storageKey`; Web via Drift `content`
  decoded with `decodeWebDocumentContent` — UTF-8 for subtitles, **base64 for
  EPUB** (LD-27 / D6)).
- **Reader translation link (MVP `reader/split` / `library/documents`):**
  nullable `translationDocumentId` on original; `listInFolder` / watch must
  hide satellite translation rows; `attachTranslation`,
  `detachAndDeleteTranslation`, cascade on original delete,
  `loadLinkedTranslation` — see OpenSpec LD-12…LD-16. One translation slot per
  original (attach while linked → validation error; delete first).
  **Family match (LD-26):** EPUB↔EPUB or subtitle↔subtitle only — reject mixed
  with `incompatibleTranslationFormatMessage`.
- **Reading resume (LD-17 / LD-28 / RS-12):** `ReadingAnchor` (`startMs`,
  optional `alignment`); `saveReadingAnchor` / `loadReadingAnchor` on the
  **original** document only. Clamp alignment to `0.0…1.0`. **Format-gated
  `startMs`:** subtitles = cue start ms; EPUB (`documentFormatEpub`) =
  **0-based chapter index** (same Drift column). Library open loads
  best-effort and passes `initialReadingAnchor` into `ReaderScreen`; persist
  failures must not block UI.
- **Video attachment (`add-reader-video-frames` / LD-18…LD-23):**
  `attachVideo` (`clearFrameCache` default true) / `detachVideo` /
  `getVideoAttachment` / `saveVideoLastFramePage` + frame JPEG
  `putFrameCache` / `getFrameCache` on the **original**; cascade on
  `deleteDocument`; Library open best-effort restore (fail must not block);
  durable link may die → keep cache + re-pick with `clearFrameCache: false`
  (LD-22); replace with different duration clears prior JPEG (LD-23).
  **`VideoAttachment.lastFramePage` / `saveVideoLastFramePage`:** API names keep
  the historical column (`videoLastFramePage`); **semantics = 0-based
  `last_second`**; page = `videoFramePageForSecond` on restore). Validate
  `≥ 0` (not `≥ 1`). UI: `work-with-reader` / `video-frames.md`.
- **Shared parse seams (Library + Reader):**
  `lib/data/library/parse_subtitles_fn.dart` (`ParseSubtitlesFn`),
  `lib/data/library/parse_epub_fn.dart` (`ParseEpubFn`),
  `lib/data/library/parse_error_messages.dart` (`messageForParseError`,
  `messageForEpubParseError`).
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

Document **bytes** are not always a Drift BLOB-only story:

| Impl | Path | When |
|------|------|------|
| Abstract + memory fake | `document_content_store.dart` | Production interface + tests; `encodeWebDocumentContent` / `decodeWebDocumentContent` |
| IO (native) | `document_content_store_io.dart` | Sandbox file + `storageKey` (raw bytes; format unused for encode) |
| Web | `document_content_store_web.dart` | Inline `content` in Drift — UTF-8 text for subtitles; **base64** for EPUB |
| Platform select | `document_content_store_platform.dart` | Conditional export |

`store({ fileName, bytes, format })` — pass stored format so Web encoding is
correct. `DocumentRepository` uses the store, then writes metadata rows. Tests
inject `MemoryDocumentContentStore` (content vs storageKey modes) and cover
EPUB base64 roundtrip (LD-27).

**Read path (Reader open):** `DocumentContentStore.load(storageKey)` (native
sandbox file; Web path usually reads Drift `content` via
`DocumentRepository.loadBytes` + format-gated decode — Web store may throw if
called). Do **not** change Drift schema for EPUB (reuse `content` column). Keep
`MemoryDocumentContentStore.load` so widget tests can open imported fixtures.

## Injection

Prefer constructing repos once in `AppShell` (or tests) and passing into
`LibraryScreen` / `VocabularyScreen`, and into `ReaderScreen` when split
translation load/persist/delete is in scope (`DocumentRepository` + optional
parse fn). Optional constructor overrides enable fakes without a global
service locator.
