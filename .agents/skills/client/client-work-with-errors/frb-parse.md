# FRB parse / Library open errors

Read with the [core errors skill](SKILL.md). Open flow lives in Library
orchestration (`lib/features/upload/library_screen.dart`), not inside
`ReaderScreen`.

## Contract

1. `DocumentRepository.loadBytes(doc)` (Drift / content store).
2. FRB `parseSubtitles(bytes:, formatHint:)` → `ParsedSubtitles`.
3. Success with ≥1 cue → `Navigator.push(ReaderScreen)`.
4. Any hard fail → stay on Library folder; SnackBar (`libraryErrorSnack`);
   **no** Reader route.

Inject `ParseSubtitlesFn` on `LibraryScreen` in widget tests — do not hit real
FFI from `flutter test` unless an integration harness requires it.

## Map `ParseErrorKind` (prefer typed switch)

| Kind | User copy (EN, Library style) |
|------|-------------------------------|
| `unsupportedFormat` | Unsupported or unrecognized subtitle format |
| `encoding` | File encoding is not valid UTF-8 |
| `empty` | No subtitle cues found in this file |
| `malformed` | Could not parse this subtitle file |

Catch `on ParseErrorKind` **before** `PanicException` / bare `catch`. Do **not**
scrape free-form anyhow strings when the typed enum is on the bridge.

## Other failures on open

| Failure | UX |
|---------|-----|
| Load bytes / missing sandbox file | Generic “Could not load document”; log details |
| `PanicException` | Generic “Could not load document”; never show panic body |
| Unexpected | Same generic; assert-debug log |
| Zero cues if API returned ok-empty | Treat as empty — SnackBar; no push (defense in depth) |

Loading overlay on open must be **non-wiping** (list stays visible under
progress). Clear `_opening` in `finally` / after successful push.

## Do not

- Push Reader on parse/load failure.
- Parse SRT/VTT/ASS in Dart to “avoid” FRB errors.
- Show Rust crate paths, panic text, or backtraces in SnackBar copy.
