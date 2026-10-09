# Upload / subtitle library

Read with the [core pages skill](SKILL.md) for shell/nav. Folder roles:
`client-work-with-structure` → `lib/features/upload/`.

## Target screens

| Screen | Suggested path | Role |
|--------|----------------|------|
| Library / list | `lib/features/upload/library_screen.dart` (name flexible) | Saved / recent subtitle documents |
| Import / pick | same feature folder (dialog, sheet, or dedicated screen) | Pick SRT / VTT / ASS(SSA) file |

Default post-scaffold **home** destination in [shell.md](shell.md).

## UX canon (target)

- Show a list of documents the user can open (title, format, optional duration /
  cue count when Rust provides them).
- Affordance to **add / import** a subtitle file (platform file picker).
- Selecting a row opens **Reader** ([reader.md](reader.md)) for that document.
- Empty state: clear CTA to import — not the scaffold Rust demo form.
- Delete / rename — only when Rust persistence API exists; UI stays thin.

## Layering

```text
LibraryScreen
  → orchestration (list documents, import bytes/path)
      → FRB: parse / store document (Rust)   # when API exists
  → Navigator → ReaderScreen(documentId)
```

- Flutter: pick file, show list, navigate.
- Rust: parse SRT/VTT/ASS, persist document metadata/cues (SQLite target).
- Do **not** parse subtitle formats in Dart.

## Scaffold note

Today there is no upload UI — only `MyHomePage` demo. First product landing
should replace that demo with this library screen (or a shell whose first tab
is this screen).

## Out of scope here

- In-player video sync (AGENTS: out of scope).
- Cloud sync of libraries.
- Content-catalog / marketplace UI (not a product surface).
