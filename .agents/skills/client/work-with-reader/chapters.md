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
   `ListView`) so Show/Hide split can preserve each pane's own mid-chapter
   scroll — [split-scroll.md](split-scroll.md).
2. **Top-pane-only chapter pager** (prev / next / title) in split (design
   D10 / RC-10/11) — session state is still **dual indices:**
   `_originalChapterIndex` and `_translationChapterIndex`, but only the
   **top** pager drives `_goToChapter(fromOriginal: true, …)`. The bottom
   (translation) pane renders **no** pager and **no** chapter title — plain
   paragraphs only. In single-pane (non-split) book-mode, the pager is on
   the one visible pane as before.
3. Changing chapter via the top pager remounts **both** panes' paragraph
   lists and rewinds **both** to paragraph 0 (RC-02 / RC-12) — this is a
   hard rewind, not a captured-anchor jump; clear any Show/Hide preserve
   seed on both panes so the new chapter opens at para 0 for each.
   Invalidate in-flight tokenize/mark (`_tokenGen`) so a slower prior
   chapter cannot bind onto the next. Show/Hide split remount (without a
   chapter change) MUST **preserve** each pane's **own** mid-chapter
   paragraph scroll (RS-10/11/24) — capture/restore + distinct single/split
   list Keys / `PageStorageKey` (`readerBookParagraphList` vs
   `readerOriginalBookParagraphList` vs `readerTranslationBookParagraphList`),
   not leave at para 0.
4. **No align on Show split / Load EPUB translation** (design D9, RS-19 /
   RS-20): the bottom pane never jumps to clamp(original chapter) +
   original's paragraph on Show or on a fresh Load — it keeps/seeds its
   **own** chapter + paragraph (reopen is the one exception — chapter clamps
   to the restored original at that point, per D10; paragraph position
   still comes from the bottom's own persisted anchor, LD-32). After any
   state change, chapter indices **and** paragraph scrolls stay independent
   until the next top-pager press (RS-21). No new Rust/FRB — use existing
   `EpubChapterDto.paragraphs`.
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
| Restore | Clamp index to `0…chapters.length-1`; open that **original** chapter; translation chapter clamps to it (D10) but paragraph position restores from the translation's **own** anchor — no scroll sync (RS-12 / LD-32) |

Subtitle documents keep cue `startMs` semantics. Call sites must be
format-gated.

## Load translation (book-mode)

- Accept only `.epub` (`isEpubFormat`); reject subtitle with
  `incompatibleTranslationFormatMessage` (LD-26 / RS-08).
- Parse via `_parseEpub`; empty chapters → `messageForEpubParseError(Empty)`.
- Unequal chapter counts vs original → still attach, clamp, show
  `mismatchedEpubChapterCountMessage` (RS-16 / D5); load itself does **not**
  align translation chapter/paragraph from the original (D9 / RS-20) — only
  a later top-pager press links the chapter.
- Errors: `EpubParseErrorKind` → `messageForEpubParseError`.

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Opening EPUB with empty cues + subtitle UI | Pass `originalChapters` + `documentFormat: epub` |
| Pager / chapter title on the bottom (translation) pane | Top pane only; bottom is plain paragraphs (D10 / RC-11) |
| Aligning translation chapter/paragraph to original on Show / Load | No align ever on those triggers (D9); only the top pager links the chapter, and only reopen clamps it |
| Captured-anchor jump on chapter change | Hard rewind: both lists → paragraph 0 (RC-12) |
| New FRB pairing for paragraph align | UI index clamp on `EpubChapterDto.paragraphs` |
| Video frames on EPUB | Hide video; skip attachment restore |
| Persisting cue `startMs` for EPUB resume | Persist **original** chapter index in `lastReadStartMs`; translation paragraph index on its own (hidden) Documents row (LD-32) |
| Tokenizing all chapters on open | Tokenize **current** chapter paragraphs only |
| Mixing EPUB↔SRT in split | EPUB↔EPUB only (`attachTranslation` + Reader load) |

## Checklist

- [ ] Book-mode gated by `documentFormat == epub`
- [ ] Paragraph list; top-pane-only chapter pager in split (no pager/title on
      bottom, RC-11); chapter change remounts + rewinds **both** lists to
      para 0 (RC-10/12); Show/Hide (no chapter change) preserves **each
      pane's own** mid-chapter scroll
- [ ] Dual chapter indices; **no** align to original on Show / Load (D9);
      bottom chapter clamps to original only at reopen (D10)
- [ ] Tokenize/mark current chapter; invalidate on chapter switch
- [ ] No video chrome / no video restore for EPUB
- [ ] Resume: `lastReadStartMs` = **original** chapter index; translation's
      own paragraph+alignment anchor restores independently (LD-32)
- [ ] Translation: EPUB-only, zero sync, own-anchor persist ([split-scroll.md](split-scroll.md))
- [ ] Widget tests: book-mode + EPUB split (e.g. `reader_book_mode_test`,
      `reader_epub_split_test`)
