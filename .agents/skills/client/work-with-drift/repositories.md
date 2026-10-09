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

- **Folders:** create (root/nested), list children, cascade delete.
- **Documents:** import bytes + metadata into a folder, list, delete; format
  helpers in `document_format.dart`.
- **Dictionary:** add (lemma + translation + optional context), list, remove,
  lookup by lemma (normalized).

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

## Injection

Prefer constructing repos once in `AppShell` (or tests) and passing into
`LibraryScreen` / `VocabularyScreen`. Optional constructor overrides enable
fakes without a global service locator.
