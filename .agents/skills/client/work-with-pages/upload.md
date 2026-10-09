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
- Selecting a document row does **not** open Reader in the Drift-library change (no-op / «Reader later»); Reader open is a later change.
- Empty state (root or folder): clear CTA to create a folder (and import when inside a folder) — not the scaffold Rust demo form.

## Layering

```text
LibraryScreen
  → orchestration (list folders/documents, import bytes/path, delete)
      → Drift repositories / DAOs   # user persist source of truth
  → (later) Navigator → ReaderScreen(documentId)
  → (later) FRB: parse cues when Reader lands
```

- Flutter: pick file, Drift CRUD for folders/documents, show list, navigate.
- Rust (later): parse SRT/VTT/ASS for Reader cues — compute only; **not** document store.
- Do **not** parse subtitle formats in Dart.

## Current tree

`lib/features/upload/library_screen.dart` is the Library destination (default
shell tab). Wire persist through `lib/data/library/*` — see `work-with-drift`.

## Out of scope here

- In-player video sync (AGENTS: out of scope).
- Cloud sync of libraries.
- Content-catalog / marketplace UI (not a product surface).
