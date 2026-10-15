# Upload / library (subtitles + EPUB)

Read with the [core pages skill](SKILL.md) for shell/nav. Folder roles:
`client-work-with-structure` → `lib/features/upload/`. Persist:
`work-with-drift` → `lib/data/library/`.

## Target screens

| Screen | Suggested path | Role |
|--------|----------------|------|
| Library / list | `lib/features/upload/library_screen.dart` (name flexible) | Folder tree + documents in selected folder |
| Import / pick | same feature folder (dialog, sheet, or dedicated screen) | Pick SRT / VTT / ASS(SSA) / EPUB into a folder |

Default post-scaffold **home** destination in [shell.md](shell.md).

## UX canon (target)

- Navigate **folders** (`library/folders`): create root/nested, open folder, cascade delete with confirm.
- Inside a selected folder: list child folders + **documents** (`library/documents`); import via platform file picker.
- Import / delete go through **Drift** in Flutter — do **not** wait for a Rust persistence API.
- Supported extensions: `.srt` / `.vtt` / `.ass` / `.ssa` / `.epub`
  (`unsupportedLibraryFileTypeMessage`).
- Selecting a document row **opens Reader** after `loadBytes` + FRB parse
  succeed; on parse/load failure stay in the folder and show a SnackBar —
  do **not** push Reader.
  - **Subtitles:** `parseSubtitles` → `originalCues` (+ optional linked
    subtitle translation).
  - **EPUB (`documentFormatEpub`):** `parseEpub` → `originalChapters` +
    `documentFormat: epub` (book-mode; empty `originalCues`). Do **not**
    restore video attachment for EPUB (LD-19).
  Injectable `ParseSubtitlesFn` /
  `ParseEpubFn` (`parse_subtitles_fn.dart` / `parse_epub_fn.dart`) keep
  widget tests off real FFI; copy in `parse_error_messages.dart`
  (`messageForParseError` / `messageForEpubParseError`).
- **Linked translation restore (LD-16):** if `doc.translationDocumentId != null`,
  best-effort load + parse **same family** (subtitle→`parseSubtitles`,
  EPUB→`parseEpub`). Success → pass translation cues/chapters + id.
  Failure / empty MUST NOT block original open — still pass
  `translationDocumentId` when linked. Linked rows stay hidden
  (`DocumentRepository.listInFolder`).
- **Reading anchor (LD-17 / LD-28 / RS-12):** best-effort
  `loadReadingAnchor(doc.id)` → `initialReadingAnchor`. EPUB: `startMs` =
  chapter index. Fail must not block open.
- **Video attachment (`reader/video-frames` / LD-18…):** best-effort
  `getVideoAttachment` for **subtitle** originals only → pass into
  `ReaderScreen`. Fail / dead link MUST NOT block open. Details:
  `work-with-reader` /
  [video-frames.md](../work-with-reader/video-frames.md).
- Empty state (root or folder): clear CTA to create a folder (and import when inside a folder) — not the scaffold Rust demo form.

## Layering

```text
LibraryScreen
  → orchestration (list folders/documents, import bytes/path, delete)
      → Drift repositories / DAOs   # user persist source of truth
  → on document tap:
        if epub: loadBytes → FRB parseEpub → ReaderScreen(
              documentFormat: epub, originalChapters, …, parseEpub?)
        else: loadBytes → FRB parseSubtitles → (optional linked translation)
              → optional getVideoAttachment (best-effort)
              → ReaderScreen(originalCues, …, parseSubtitles?, tokenizeCues?)
  → original fail: SnackBar; no Reader route
  → translation fail: ignore; still push Reader with original (+ link id if any)
  → reading-anchor / video-meta fail: ignore; still push Reader
```

- Flutter: pick file, Drift CRUD for folders/documents, show list, open-flow
  orchestration, Reader UI; pass `dictionaryRepository` into Reader for
  add-vocab sheet.
- Rust: `parse_subtitles` / `parse_epub` — compute only; **not** document store.
- Do **not** parse subtitle/EPUB formats in Dart.

## Current tree

`lib/features/upload/library_screen.dart` is the Library destination (default
shell tab). Wire persist through `lib/data/library/*` — see `work-with-drift`.

## Out of scope here

- Full A/V **playback** / in-player sync (product is cues-first +
  optional paged still-frame column — not a video player).
- Cloud sync of libraries.
- Content-catalog / marketplace UI (not a product surface).
