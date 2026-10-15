# EPUB parse (`parse/epub`)

Read with [SKILL.md](SKILL.md). Capability: OpenSpec `parse/epub`
(change `add-epub-reader`). Flutter book-mode UX → `work-with-reader`
([chapters.md](../../client/work-with-reader/chapters.md)).

## Placement

```text
rust/src/parse/epub/
  mod.rs      # parse_epub(&[u8]) → EpubDocument { chapters }
  strip.rs    # XHTML → plain paragraphs (no rich HTML)
  error.rs    # EpubParseError { UnsupportedFormat, Malformed, Empty }
  tests.rs + fixtures/*.epub
rust/src/api/epub.rs   # Thin FRB: parse_epub(bytes) → ParsedEpub / EpubParseErrorKind
```

Register `pub mod epub` in `api/mod.rs`. Domain stays under `parse/epub/`;
do **not** put TOC/strip logic in `api/`.

## Crate pin

| Dep | Pin / features | Notes |
|-----|----------------|-------|
| **rbook** | `0.7`, `default-features = false`, features `threadsafe` + `prelude` | Read-only; **no** `write`. Must stay wasm-friendly (`cargo check --target wasm32-unknown-unknown`). |

Do not casually swap crates. Open via `Epub::read(Cursor::new(bytes.to_vec()))`
(bytes-in; no filesystem path). `Cursor<&[u8]>` will not compile — rbook
needs `'static` Read+Seek.

## Output model (FFI-flat)

| Type | Fields |
|------|--------|
| `EpubChapterDto` | `index: u32` (0-based document order), `title: String`, `paragraphs: Vec<String>` (non-empty plain) |
| `ParsedEpub` | `chapters: Vec<EpubChapterDto>` |
| `EpubParseErrorKind` | `UnsupportedFormat` \| `Malformed` \| `Empty` |

**Separate** from subtitle `ParseErrorKind` — do not reuse the subtitle enum.

Empty title after strip → stable fallback `"Chapter {n}"` (1-based display n).
Chapters with zero paragraphs after strip are dropped; if none remain → `Empty`.

## Chapter discovery (design D3)

1. **TOC path:** walk `Epub::toc().contents()` preorder; each non-root entry
   with a resolvable `resource()` and non-empty plain text after strip → one
   chapter (nested TOC included).
2. **No TOC / empty TOC result:** fall back to spine order (one chapter per
   readable spine item with text).
3. **Fragment (`#…`) in href:** ignore — read the whole XHTML resource.
4. Missing/unreadable resource: skip that entry (do not hard-fail the book
   unless nothing extractable remains).

## Plain strip (D9)

- Strip tags / scripts / styles; collapse whitespace; emit **paragraphs**
  (block boundaries), not a single blob.
- No images, footnotes, or rich HTML on the FFI surface for v1.
- Reader tokenizes paragraph strings later (`tokenize_cues`) — do not tokenize
  here.

## Errors → Flutter

| Kind | When | Typical UI copy |
|------|------|-----------------|
| `UnsupportedFormat` | Empty bytes / not a readable EPUB container | “Could not open this EPUB file” |
| `Malformed` | Opened but structure/content unusable mid-parse | same open-fail copy |
| `Empty` | No chapter with ≥1 non-empty paragraph | “No extractable text found in this EPUB” |

Flutter shared mapping: `messageForEpubParseError` in
`lib/data/library/parse_error_messages.dart`. Hard fail → stay on Library
(no Reader push).

## Tests

- Fixtures under `rust/src/parse/epub/fixtures/` (TOC, nested, no-TOC,
  markup strip, empty text, fragment href).
- `#[cfg(test)]` in `parse/epub/`; run `cargo test` from `{app}/rust`.
- Do **not** prove EPUB grammar only via `flutter test`.

## Related

- Thin export / codegen → `work-with-frb-api`
- Library Web base64 persist → `work-with-drift`
- Book-mode UI → `work-with-reader` / `chapters.md`
