# Upload / subtitle library

Read with the [core pages skill](SKILL.md) for shell/nav. Folder roles:
`client-work-with-structure` → `lib/features/upload/`. Persist:
`work-with-drift` → `lib/data/library/`.

## Target screens

| Screen | Suggested path | Role |
|--------|----------------|------|
| Library / list | `lib/features/upload/library_screen.dart` (name flexible) | Folder tree + documents in selected folder |
| Import / pick | same feature folder (dialog, sheet, or dedicated screen) | Pick SRT / VTT / ASS(SSA) file into a folder |

Default post-scaffold **home** destination in [shell.md](shell.md).

## UX canon (target)

- Navigate **folders** (`library/folders`): create root/nested, open folder, cascade delete with confirm.
- Inside a selected folder: list child folders + **documents** (`library/documents`); import via platform file picker.
- Import / delete go through **Drift** in Flutter — do **not** wait for a Rust persistence API.
- Selecting a document row **opens Reader** after `loadBytes` + FRB
  `parseSubtitles` succeed; on parse/load failure stay in the folder and show a
  SnackBar (`libraryErrorSnack`) — do **not** push Reader. Injectable
  `ParseSubtitlesFn` (shared typedef in
  `lib/data/library/parse_subtitles_fn.dart`) keeps widget tests off the real
  FFI; parse copy lives in `parse_error_messages.dart`.
- **Linked translation restore (`reader/split` / LD-16):** if
  `doc.translationDocumentId != null`, best-effort
  `loadLinkedTranslation` → `loadBytes` → `parseSubtitles`. Success with cues →
  pass `translationCues` + id into `ReaderScreen`. Failure / empty cues MUST
  NOT block original open — still pass `translationDocumentId` when linked so
  Reader can show stuck-link empty (Delete in bottom-pane chrome → Load).
  Linked rows stay hidden in the folder list (`DocumentRepository.listInFolder`).
- **Reading anchor (LD-17 / RS-12):** best-effort `loadReadingAnchor(doc.id)` →
  pass `initialReadingAnchor` into `ReaderScreen`. Fail must not block open.
- **Video attachment (`reader/video-frames` / LD-18…):** best-effort
  `getVideoAttachment(doc.id)` → pass into `ReaderScreen` so the strip can
  restore. Fail / dead link MUST NOT block original open (keep JPEG cache;
  re-pick in Reader). Details: `work-with-reader` /
  [video-frames.md](../work-with-reader/video-frames.md).
- Empty state (root or folder): clear CTA to create a folder (and import when inside a folder) — not the scaffold Rust demo form.

## Layering

```text
LibraryScreen
  → orchestration (list folders/documents, import bytes/path, delete)
      → Drift repositories / DAOs   # user persist source of truth
  → on document tap: loadBytes → FRB parseSubtitles → (optional linked translation)
      → optional getVideoAttachment (best-effort)
      → Navigator.push(ReaderScreen(
            originalCues, originalDocumentId, documentRepository,
            dictionaryRepository,   # pass-through from AppShell (reader/tap)
            translationCues?, translationDocumentId?,
            initialReadingAnchor?, video attachment?, parseSubtitles?, tokenizeCues?))
  → original fail: SnackBar; no Reader route
  → translation fail: ignore; still push Reader with original (+ link id if any)
  → reading-anchor / video-meta fail: ignore; still push Reader
```

- Flutter: pick file, Drift CRUD for folders/documents, show list, open-flow orchestration, Reader UI; pass `dictionaryRepository` into Reader for add-vocab sheet.
- Rust: parse SRT/VTT/ASS for Reader cues — compute only; **not** document store.
- Do **not** parse subtitle formats in Dart.

## Current tree

`lib/features/upload/library_screen.dart` is the Library destination (default
shell tab). Wire persist through `lib/data/library/*` — see `work-with-drift`.

## Out of scope here

- Full A/V **playback** / in-player sync timeline (product is cues-first +
  optional still-frame strip — not a video player).
- Cloud sync of libraries.
- Content-catalog / marketplace UI (not a product surface).
