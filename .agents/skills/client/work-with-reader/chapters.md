# EPUB book-mode (`reader/chapters`)

Read with [SKILL.md](SKILL.md). Capability: OpenSpec `reader/chapters`
(change `add-epub-reader`). Parse grammar → `work-with-parse` /
[epub.md](../../server/work-with-parse/epub.md). Split chapter sync →
[split-scroll.md](split-scroll.md). Persist / open-flow → `work-with-drift`,
`work-with-pages` (`upload.md`).

## When book-mode applies

`ReaderScreen.documentFormat == 'epub'` (`isEpubFormat` /
`documentFormatEpub`). Library open uses `parse_epub` (not `parse_subtitles`)
and passes chapter payloads — do **not** push EPUB as empty `originalCues`.

| Param | Role |
|-------|------|
| `documentFormat` | `'epub'` gates book-mode |
| `originalChapters` | `List<EpubChapterDto>` from Library `parse_epub` |
| `translationChapters?` | Linked EPUB translation (best-effort) |
| `originalCues` | **Empty** in book-mode (D4) |
| `parseEpub` | Injectable `ParseEpubFn` (`lib/data/library/parse_epub_fn.dart`) |
| `initialReadingAnchor` | `startMs` = **0-based chapter index** (LD-28 / D6) |

## UI contract

1. **Paragraph list** of the current chapter (not cue timestamps). Reuse
   `CueRow` / tokenize paint via display-only `CueDto` adapters
   (`startMs`/`endMs` unused; one paragraph → one row).
2. **Bottom chapter pager** (prev / next / title) — shared session
   `_chapterIndex` for original and translation panes.
3. Changing chapter remounts the paragraph list (scroll resets to start —
   RC-02). Invalidate in-flight tokenize/mark (`_tokenGen`) so a slower
   prior chapter cannot bind onto the next.
4. **No video chrome** for EPUB (RC-05): hide Load / Replace / Remove video;
   do not restore `VideoAttachment` on Library open for epub (LD-19).
5. Tap / highlight / Add-edit sheets apply to **paragraph tokens** the same
   way as cue tokens ([tap.md](tap.md), [highlight.md](highlight.md)).

## Tokenize

On open and on each **original** chapter change: batch-tokenize
**current chapter paragraphs** (`tokenizeCues` with paragraph strings), then
`markVocabTokens`. Translation pane stays plain (no highlight), same as
subtitle split.

## Reading resume (LD-28 / RC-06)

| Piece | EPUB semantics |
|-------|----------------|
| Drift column | Same `Documents.lastReadStartMs` |
| Meaning | 0-based **chapter index** (not milliseconds) |
| Save | On pager settle / dispose — write `_chapterIndex` (not `nearestCueIndex`) |
| Restore | Clamp index to `0…chapters.length-1`; open that chapter |

Subtitle documents keep cue `startMs` semantics. Call sites must be
format-gated.

## Load translation (book-mode)

- Accept only `.epub` (`isEpubFormat`); reject subtitle with
  `incompatibleTranslationFormatMessage` (LD-26 / RS-08).
- Parse via `_parseEpub`; empty chapters → `messageForEpubParseError(Empty)`.
- Unequal chapter counts vs original → still attach, clamp pager, show
  `mismatchedEpubChapterCountMessage` (RS-16 / D5).
- Errors: `EpubParseErrorKind` → `messageForEpubParseError`.

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Opening EPUB with empty cues + subtitle UI | Pass `originalChapters` + `documentFormat: epub` |
| Video frames on EPUB | Hide video; skip attachment restore |
| Persisting cue `startMs` for EPUB resume | Persist chapter index in `lastReadStartMs` |
| Tokenizing all chapters on open | Tokenize **current** chapter paragraphs only |
| Mixing EPUB↔SRT in split | EPUB↔EPUB only (`attachTranslation` + Reader load) |

## Checklist

- [ ] Book-mode gated by `documentFormat == epub`
- [ ] Paragraph list + bottom chapter pager; list remount on chapter change
- [ ] Tokenize/mark current chapter; invalidate on chapter switch
- [ ] No video chrome / no video restore for EPUB
- [ ] Resume: `lastReadStartMs` = chapter index
- [ ] Translation: EPUB-only + chapter-index sync ([split-scroll.md](split-scroll.md))
- [ ] Widget tests: book-mode + EPUB split (e.g. `reader_book_mode_test`,
      `reader_epub_split_test`)
