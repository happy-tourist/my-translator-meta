# Split View — Independent Panes, Own-Anchor Persist, Linked Chapter Pager

## Layout

- Two panes: **original** (top) and **translation** (bottom) — cues **or**
  EPUB paragraphs.
- Flutter owns two scrollables. **No sync between panes, ever** (design D9) —
  each pane only restores **its own** previous position; the only link
  between panes is the **top** chapter pager clamping the bottom's chapter
  index (design D10, EPUB only).
- **Subtitles (capability `reader/split`):** independent cue lists from two
  `parseSubtitles` calls; each pane keeps its own session seed
  (`_originalInitialScrollIndex` / `_translationInitialScrollIndex`) — **no**
  Rust pairing / align API, **no** `nearestCueIndex`-driven cross-pane jump.
- **EPUB (capability `reader/chapters` + split):** EPUB↔EPUB chapter lists
  from `parse_epub`. Chapter **index** is linked top→bottom via the top
  pager only (D10); paragraph **scroll position** inside that chapter is
  each pane's own seed, never derived from the other pane. No Rust pairing
  API — index clamp only. Details: [chapters.md](chapters.md).

## Implementation anchors (present)

| Piece | Path / dep |
|-------|------------|
| Shared cue / paragraph row | `lib/features/reader/cue_row.dart` (`CueRow`, key prefixes) |
| Pane lists (subtitles) | `scrollable_positioned_list` → `ItemScrollController` + `ItemPositionsListener` |
| Pane lists (EPUB book) | Same package: **book-specific** `ItemScrollController` + `ItemPositionsListener` per pane (not bare `ListView`) — required for RS-10/11/24 own-anchor preserve |
| Orchestration | `ReaderScreen` (`lib/features/reader/reader_screen.dart`) StatefulWidget |
| EPUB chapter session | `_originalChapterIndex` + `_translationChapterIndex`; only `_goToChapter(fromOriginal: true, …)` exists — top pager is the **only** driver (RC-10/11) |
| Bottom own-anchor seed | `_translationInitialScrollIndex`/`_translationInitialAlignment` (cues); `_bookTranslationInitialScrollIndex`/`_bookTranslationInitialAlignment` (EPUB paragraphs) |
| Bottom own-anchor persist | `_persistTranslationReadingAnchor` → `saveReadingAnchor` on the translation Documents row (D11 columns); Hide/dispose `force` falls back to session seed if list unmounted (same harden on original `_persistReadingAnchor`) |

## Zero cross-pane sync (D9)

There is **no** one-shot align, no `_syncPanes`, no `_syncTranslationToTimeMs`,
no `_syncBookPanes`, no `_alignTranslationChapterToOriginal` — these helpers
were **removed**, not just disabled. Do not reintroduce them.

1. Showing split, hiding split, successfully loading a translation file, and
   reopening the document from Library all MUST NOT move the bottom pane to
   match the top pane's time / cue index / chapter / paragraph (RS-05 / RS-18
   / RS-19 / RS-20 / RS-24).
2. The bottom pane's scroll position comes from exactly one of: (a) this
   session's own capture (Hide / settle), or (b) the persisted translation
   `ReadingAnchor` loaded at Library-open time (`initialTranslationReadingAnchor`
   → `_seedTranslationAnchor()`); never from the top pane's listener or anchor.
3. A freshly **loaded** translation (first attach in this session, cues or
   EPUB chapters) starts at index 0 / its own default chapter — not clamped
   or jumped to the original's current position.
4. After any of the above, settle/fling on one pane MUST NOT move the other
   (RS-06 / RS-21) — this was already true pre-change and still holds.
5. Video page change MUST NOT sync translation (unchanged; `reader/video-frames`).

### Subtitles — independent cue lists (`reader/split`)

- Each pane's `ScrollablePositionedList` seeds from its own
  `_originalInitialScrollIndex` / `_translationInitialAlignment` pair (top) or
  `_translationInitialScrollIndex` / `_translationInitialAlignment` pair
  (bottom) — see `_buildCueList(fromOriginal: …)`.
- `nearestCueIndex` (`lib/features/reader/nearest_cue_index.dart`) is used
  **only** to map a saved `ReadingAnchor.startMs` (ms, `BigInt`) onto the
  current translation cue list when seeding from persist
  (`_seedTranslationAnchor`) — never to jump one pane from the other's live
  position.

### EPUB — chapter index linked top→bottom; paragraph scroll independent

1. **Session state = two chapter indices:** `_originalChapterIndex` and
   `_translationChapterIndex` (0-based). **Only** the top pager calls
   `_goToChapter(fromOriginal: true, next:)` (D10 / RC-10/11) — there is no
   bottom pager and no chapter title on the translation pane; `fromOriginal:
   false` is a no-op guard, not a second control path.
2. Top prev/next (when a translation is already loaded) SHALL: set
   `_translationChapterIndex = clamp(next, 0, translationN-1)` **and** rewind
   **both** paragraph lists to index 0 / alignment 0 (RC-12) — chapter change
   is a hard rewind, not a one-shot paragraph jump to a captured anchor.
3. Loading a translation while already on some chapter does **not** align the
   new translation's chapter to the original (D9 / RS-20) — it clamps its
   **own** current `_translationChapterIndex` into range and starts fresh;
   only a **later** top pager press links it.
4. On reopen (LD-32 / D10): the bottom's chapter index SHALL follow the
   restored **original** chapter (clamp) — that is the one allowed
   top→bottom link at open time, same rule as the live pager — but the
   **paragraph position inside that chapter** is seeded from the bottom's own
   persisted anchor (`_seedTranslationAnchor`), not from the original's
   paragraph offset.
5. Each pane's paragraph scroll after any of the above is independent
   (RS-21); do **not** use `nearestCueIndex` / cue `startMs` for EPUB; do
   **not** add a Rust pairing / paragraph-align FRB — index clamp in UI only.
6. Unequal chapter counts on load: attach anyway, clamp, show
   `mismatchedEpubChapterCountMessage` (RS-16) — no one-shot paragraph jump
   either way. Unequal **paragraph** counts: silent clamp when restoring an
   anchor (RS-22), no error chrome.
7. Reject subtitle translation against EPUB original (and vice versa) —
   `incompatibleTranslationFormatMessage` (LD-26 / RS-08).

## Toggle split — each pane preserves its OWN scroll (RS-10, RS-11, RS-24)

**Subtitles:** enabling **or** hiding split remounts the original list.
Before `setState` on either direction, capture the top-visible anchor from
**that pane's own** `ItemPositionsListener` (`index` + `itemLeadingEdge`) —
`_captureTranslationCueAnchor()` for the bottom, same pattern as the existing
original-pane capture — and seed that pane's own
`initialScrollIndex`/`initialAlignment` on remount; after the first frame
(`ItemScrollController.isAttached`), `jumpTo` that anchor as backup
(`_restoreTranslationCueScroll`, retries like the original). **Hide** also
persists the bottom's anchor immediately (`_persistTranslationReadingAnchor(force:
true)`) since its list is about to dispose. Use **distinct list Keys** for
single vs split original (`readerCueList` vs `readerOriginalCueList`) plus
matching `PageStorageKey` slots so PageStorage from one mode does not override
the other's `initialScrollIndex`.

**EPUB book-mode:** Show/Hide remounts `_buildBookPane` ↔ `_buildBookSplitBody`
and disposes the original paragraph list — capture top-visible **paragraph**
index + leading edge **per pane** before `setState` (`_captureBookTranslationAnchor()`
for the bottom), seed each pane's own `initialScrollIndex`/`initialAlignment`,
post-frame `jumpTo` on remount (RS-10/11, `_restoreTranslationBookScroll` for
the bottom). Distinct Keys: `readerBookParagraphList` (single) vs
`readerOriginalBookParagraphList` (split top) vs
`readerTranslationBookParagraphList` (split bottom), with
`PageStorageKey('${listKeyPrefix}_$chapterIndex')`. **Neither** direction
aligns the bottom's chapter or paragraph to the top (D9) — Show restores the
bottom's own seed; Hide persists it. Changing chapter via the top pager still
remounts both panes' lists → rewind to start of chapter (RC-02/RC-12) and
MUST clear the Show/Hide preserve seed for both panes so the new chapter does
not reopen mid-list.

When implementing `reader/video-frames` (**subtitle docs only**), original
pane is either the cue list **or** `VideoFramePages` (not both). Keep the
pages `GlobalKey` / state outside the Show/Hide remount branch. Full
contracts: [video-frames.md](video-frames.md). EPUB hides video chrome
entirely ([chapters.md](chapters.md)).

## Reading resume — original AND translation, each from its own anchor (RS-12/24, LD-17/28/31…33)

Persist **both** panes' positions, independently, on the **same generic**
`ReadingAnchor` / `saveReadingAnchor` API. Translation still reuses the
translation document row's `lastReadStartMs`/`lastReadAlignment` (D11 — no
*extra* translation-only table). EPUB **original** mid-chapter needs schema 8
`lastReadParagraphIndex` (D14 / LD-33) beside chapter-in-`lastReadStartMs`:

| Piece | Contract |
|-------|----------|
| Columns | `lastReadStartMs`, `lastReadAlignment`, and (original EPUB) `lastReadParagraphIndex` — read/written per `documentId` |
| Repo | `ReadingAnchor` (`startMs`, `alignment?`, `paragraphIndex?`) + `saveReadingAnchor` / `loadReadingAnchor` |
| Semantics (original) | **Subtitles:** cue `startMs` (ms). **EPUB:** chapter in `startMs` + mid-chapter `paragraphIndex` + alignment (LD-28/33) |
| Semantics (translation) | **Subtitles:** cue `startMs`. **EPUB:** paragraph index in `startMs` within the linked chapter (LD-32) — not a chapter index; `paragraphIndex` column unused / cleared |
| Open | Library best-effort load for **both** ids → `initialReadingAnchor` + `initialTranslationReadingAnchor` |
| Restore (original, subtitles) | After layout: `nearestCueIndex` → `jumpTo` **original only** |
| Restore (original, EPUB) | Clamp chapter; seed `_bookOriginalInitialScrollIndex` / alignment from `paragraphIndex` (null → 0) via list `initialScrollIndex` (RS-12 / LD-33) |
| Restore (translation, subtitles) | `_seedTranslationAnchor()` from translation cue `startMs` — not derived from original |
| Restore (translation, EPUB) | Clamp paragraph into `clamp(_originalChapterIndex)` chapter (D10 link at reopen only) |
| Save (subtitles) | Each pane on settle + Hide/dispose (`force: true`); if list already unmounted, fall back to that pane's session seed (RS-24) |
| Save (EPUB) | Original: settle/dispose capture paragraph+alignment with chapter (**force** → session seed if unmounted); pager → `_persistBookChapterAnchor(…, paragraph: 0)` (RC-12). Translation: settle / Hide / dispose with the same **force fallback** (LD-32) |

Do **not** derive one pane's save/restore value from the other's.

## Book split divider (D13 / RS-17)

In EPUB book split, drive pane heights with a `ValueNotifier` (or equivalent)
so drag updates only `SizedBox` heights — do **not** `setState` the whole
`ReaderScreen` on every pointer move (that rebuilds both paragraph lists).
Subtitle split MAY keep the older setState divider path.

## Delete chrome lives in the AppBar (RS-07 / D12)

- **AppBar** actions: split toggle, Delete translation (**new location**,
  design D12), + Load/Remove video when `reader/video-frames` (hidden for
  EPUB).
- Delete is visible whenever a translation is shown (cues or EPUB paragraphs)
  **or** a stuck link exists (`translationDocumentId` without content) —
  `_hasAnyTranslationLink`. Single Key `readerTranslationDelete` for both
  cases now (no separate `*Empty` variant — there is no bottom chrome to vary).
- The bottom pane itself has **no** delete bar, no top chrome strip, no
  `kMinInteractiveDimension` floor reserved in the divider-ratio clamp — it is
  plain text flush to the divider (see next section).

## Bottom pane is text-only, flush to the divider (RS-23 / D12)

- No `Column` wrapping a delete-chrome `Material` bar + `Expanded(content)` —
  the bottom pane **is** the list/empty-state widget directly.
- No chapter title, no prev/next pager on the bottom EPUB pane (RC-11) — the
  original (top) pane renders pager only (no fixed `chapter.title`, D15 /
  RC-13); `fromOriginal: false` returns the bare paragraph list.
- Padding: top pane keeps `EdgeInsets.symmetric(vertical: 8)`; bottom pane
  uses `EdgeInsets.only(bottom: 8)` (no top padding) so content sits right
  under the divider.
- Divider-ratio clamp (`_minRatio`-based `ratioMin`/`maxTop`) no longer
  reserves a `kMinInteractiveDimension` chrome floor for the bottom pane —
  Delete moved to the AppBar, so the plain ratio bound is enough (both
  subtitle and EPUB split layout code paths).

## Missing translation

- **MVP:** bottom pane empty until a translation file is loaded; no per-cue
  placeholders required (subtitle lists may differ in length). EPUB: empty
  until linked/loaded `.epub`.
- **Target (paired subtitle lists):** cue *i* without translation → placeholder
  row at index *i*. **Do not** drop the row (that shifts indices).
- Single-pane mode when split is off: no second controller required.

## Edge cases

| Case | Behavior |
|------|----------|
| Empty translation pane (MVP) | Show empty + load CTA; top pane scrolls alone |
| Show split after scroll (subtitles) | Restore original top-visible cue (RS-10); restore bottom's **own** saved/seeded anchor if translation present (RS-18/24) — no sync |
| Hide split after scroll (subtitles) | Same capture+`jumpTo` per pane (RS-11); persist bottom's own anchor; do not reset either to 0 |
| Show split after scroll (EPUB mid-chapter) | Restore original paragraph index (RS-10); restore bottom's **own** anchor at its (linked-on-reopen) chapter (RS-19/24) — no chapter/paragraph align to top |
| Hide split after scroll (EPUB mid-chapter) | Same capture+`jumpTo` per pane (RS-11); persist bottom's own anchor; do not reset either to para 0 |
| Unequal EPUB paragraph counts on restore | Silent clamp to last available (RS-22); no error chrome |
| Reopen from Library | Subtitles: each pane's own cue `startMs`; EPUB: original chapter **+ mid-chapter paragraph** + bottom paragraph at clamp(original chapter); fail on either → open at top / empty bottom; no sync |
| Delete translation | AppBar, next to split toggle (D12) — never bottom-pane chrome |
| Top EPUB pager, translation loaded | Bottom chapter = clamp(top); **both** paragraph lists rewind to 0 (RC-10/12) |
| Video pages present | Pages key outside remount; page change does **not** sync translation — **not** for EPUB |
| One cue only | Trivial — nothing to seed/restore beyond index 0 |
| Scroll / fling, either pane | Other pane stays independent (RS-06/21) |
| Unequal cue lengths (subtitles) | Expected; panes are fully independent — do **not** invent Rust pairing |
| Unequal EPUB chapter counts | Clamp index; show mismatch notice (RS-16) |
| Mixed EPUB↔subtitle load | Reject with incompatible-format message |

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| Any one-shot / sync helper (`_syncPanes`, `_syncTranslationToTimeMs`, `_syncBookPanes`, `_alignTranslationChapterToOriginal`) | Removed entirely (D9) — each pane restores only its own anchor |
| Settle / fling realtime sync (RS-06/21) | Independent panes, always |
| Aligning bottom chapter/paragraph to top on Show / Load | Bottom keeps its own chapter (clamped at reopen only) + own paragraph seed |
| Bottom pager / chapter title widget | Top-only pager; `_goToChapter(fromOriginal: true, …)` is the sole driver (RC-11) |
| `translationController.jumpTo(original.offset)` or any cross-pane jump | Jump each pane only from its own captured/persisted anchor |
| Bare `ListView` for EPUB paragraphs under split | Book `ScrollablePositionedList` + per-pane capture/restore (RS-10/11/24) |
| Same Key / PageStorage slot for single↔split lists | Distinct Keys (`readerCueList`/`readerOriginalCueList`; book `readerBookParagraphList`/`readerOriginalBookParagraphList`/`readerTranslationBookParagraphList`) |
| Inventing a Rust pairing API for split | Two `parseSubtitles` / `parse_epub` + Flutter-only index/time clamp |
| Delete translation in bottom-pane chrome | AppBar, next to split toggle (D12) |
| Fixed `chapter.title` chrome on original, or title/pager on bottom | Original: pager only (D15 / RC-13); bottom: text-only (RC-11) |
| `kMinInteractiveDimension` floor reserved for bottom delete chrome | No chrome floor — bottom is text-only (D12) |
| Persisting only the original's chapter (no mid-chapter) / only original pane | Persist original chapter+paragraph (LD-33) **and** bottom own anchor (LD-31/32) |
| New Drift table solely for the translation anchor | Reuse reading-anchor columns on the translation Documents row (D11); original mid-chapter uses `lastReadParagraphIndex` (schema 8 / D14) |
| Disposing video pages key with remounted cue lists | Keep `VideoFramePages` / key outside remount |
| Blocking open / settle on save/load fail | Best-effort; assert-log; continue |
