# EPUB book-mode (`reader/chapters`)

Read with [SKILL.md](SKILL.md). Capability: OpenSpec `reader/chapters`
(change `add-epub-reader`). Parse grammar → `work-with-parse` /
[epub.md](../../server/work-with-parse/epub.md). Split chapter+paragraph
one-shot → [split-scroll.md](split-scroll.md). Persist / open-flow →
`work-with-drift`, `work-with-pages` (`upload.md`).

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
   (`startMs`/`endMs` unused; one paragraph → one row). Prefer
   `ScrollablePositionedList` + book-specific controllers (not bare
   `ListView`) so Show/Hide split can preserve mid-chapter scroll and one-shot
   can jump by paragraph index — [split-scroll.md](split-scroll.md).
2. **Bottom chapter pager** (prev / next / title) per pane — session state is
   **dual indices:** `_originalChapterIndex` and `_translationChapterIndex`.
   Pager on one pane MUST NOT change the other (RS-14 / RC-10).
3. Changing chapter remounts that pane’s paragraph list (scroll resets to
   start — RC-02); clear any Show/Hide preserve seed so the new chapter opens
   at para 0. Invalidate in-flight tokenize/mark (`_tokenGen`) so a slower
   prior chapter cannot bind onto the next. Show/Hide split remount MUST
   **preserve** mid-chapter paragraph scroll of the original (RS-10/11) —
   capture/restore + distinct single/split list Keys / `PageStorageKey`
   (`readerBookParagraphList` vs `readerOriginalBookParagraphList`), not
   leave at para 0.
4. **One-shot align** on Show split / successful load EPUB translation
   (RS-19 / RS-20 / RS-22): (1) `_translationChapterIndex =
   clamp(_originalChapterIndex, 0, translationN-1)`; (2) jump translation
   paragraph list to `clamp(sourceParagraphIndex, …)` (+ alignment) from the
   **captured** anchor (do not re-read the listener after remount). After
   that, chapter indices **and** paragraph scrolls stay independent (RS-21).
   No new Rust/FRB — use existing `EpubChapterDto.paragraphs`.
5. **No video chrome** for EPUB (RC-05): hide Load / Replace / Remove video;
   do not restore `VideoAttachment` on Library open for epub (LD-19).
6. Tap / highlight / Add-edit sheets apply to **paragraph tokens** the same
   way as cue tokens ([tap.md](tap.md), [highlight.md](highlight.md)).

## Tokenize / hit-test (D11–D12)

On open and on each **original** chapter change: progressive tokenize of
**current chapter paragraphs** (`tokenizeCues` — prefix then idle remainder),
then `markVocabTokens` (highlight MAY lag; tap on ready prefix is OK).
In-memory `Map<chapterIndex, tokens>` for the Reader session; pager return
is a cache hit; re-mark after vocab mutate. Translation pane stays plain
(no highlight), same as subtitle split.

Book paragraph rows use **lightweight hit-test** (`CueRow.lightweightHitTest`:
`Text.rich` TextSpans + one gesture → char → `tokenIndex`). The hit-test
`TextPainter` MUST match ambient `Directionality` / `MediaQuery.textScaler` /
`Localizations.locale` (same metrics as painted `Text.rich`) so offset→char
stays accurate under text scale / RTL (D11). Do **not** use WidgetSpan-per-word
in book mode; subtitle CueRow path stays WidgetSpan.

## Reading resume (LD-28 / RC-06)

| Piece | EPUB semantics |
|-------|----------------|
| Drift column | Same `Documents.lastReadStartMs` |
| Meaning | 0-based **chapter index** (not milliseconds) |
| Save | On pager settle / dispose — write `_originalChapterIndex` only |
| Restore | Clamp index to `0…chapters.length-1`; open that **original** chapter; translation aligns on next Show / load (RS-12) |

Subtitle documents keep cue `startMs` semantics. Call sites must be
format-gated.

## Load translation (book-mode)

- Accept only `.epub` (`isEpubFormat`); reject subtitle with
  `incompatibleTranslationFormatMessage` (LD-26 / RS-08).
- Parse via `_parseEpub`; empty chapters → `messageForEpubParseError(Empty)`.
- Unequal chapter counts vs original → still attach, clamp, show
  `mismatchedEpubChapterCountMessage` (RS-16 / D5); **one-shot** still sets
  translation chapter **and** paragraph from original (RS-20 / RS-22).
- Errors: `EpubParseErrorKind` → `messageForEpubParseError`.

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Opening EPUB with empty cues + subtitle UI | Pass `originalChapters` + `documentFormat: epub` |
| Shared `_chapterIndex` syncing both panes | Dual indices + per-pane pager |
| Realtime pager / paragraph scroll sync after one-shot (RS-14 / RS-21) | Independent chapter indices + independent paragraph scroll |
| New FRB pairing for paragraph align | UI index clamp on `EpubChapterDto.paragraphs` |
| Video frames on EPUB | Hide video; skip attachment restore |
| Persisting cue `startMs` for EPUB resume | Persist **original** chapter index in `lastReadStartMs` |
| Tokenizing all chapters on open | Tokenize **current** chapter paragraphs only |
| Mixing EPUB↔SRT in split | EPUB↔EPUB only (`attachTranslation` + Reader load) |

## Checklist

- [ ] Book-mode gated by `documentFormat == epub`
- [ ] Paragraph list + bottom chapter pager; list remount on chapter change; preserve mid-chapter scroll on Show/Hide split
- [ ] Dual chapter indices; one-shot chapter+paragraph on Show / load translation only
- [ ] Tokenize/mark current chapter; invalidate on chapter switch
- [ ] No video chrome / no video restore for EPUB
- [ ] Resume: `lastReadStartMs` = **original** chapter index
- [ ] Translation: EPUB-only + one-shot chapter+paragraph align ([split-scroll.md](split-scroll.md))
- [ ] Widget tests: book-mode + EPUB split (e.g. `reader_book_mode_test`,
      `reader_epub_split_test`)
