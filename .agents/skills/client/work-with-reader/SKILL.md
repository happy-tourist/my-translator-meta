---
name: work-with-reader
description: >-
  Use when implementing or changing Translation Reader UX: cue/line rendering,
  EPUB book-mode (chapter pager + paragraph list; OpenSpec `reader/chapters`),
  word tap / contiguous token selection (phrase Add), bottom CTA + add-vocab
  sheet, vocab lemma/phrase highlight paint, edit sheet on in-vocab tap, split
  view original↔translation (zero cross-pane sync — each pane restores only
  its own anchor; EPUB chapter index links top→bottom only via the top
  pager; Delete lives in the AppBar), or optional paged video frames (OpenSpec
  `reader/video-frames`).
  Not for
  scaffold demo TextField / processText. Cross-cut: work-with-pages (placement),
  work-with-frb (API/codegen), work-with-styles (tokens/theme), work-with-drift
  (video meta / epub bytes).
---

# Work With Reader

## Overview

**Reader** contracts for **Translation Reader** (`my_translator_app`).
UI under `lib/features/reader/`: Stateful `ReaderScreen` (optional split /
EPUB book-mode), `cue_row.dart`, `cue_char_slice.dart`,
`tokenize_cues_fn.dart`, `cue_time_format.dart`, `nearest_cue_index.dart`.
Ctor takes already-parsed `originalCues` **or** EPUB `originalChapters` +
`documentFormat` + title + `originalDocumentId` + injected
`DocumentRepository` + required `DictionaryRepository` (optional translation
cues/chapters/id, `initialReadingAnchor`, `ParseSubtitlesFn` /
`ParseEpubFn`, `TokenizeCuesFn`, optional `MarkVocabTokensFn` / lemma-keys
seam, file picker). Library owns open/parse of the original (+ best-effort
linked translation + reading anchor) and passes the dictionary repo through.
Scaffold demo in older docs is **not** the reader — do not grow demo chrome
into product UX.

Heavy compute stays in **Rust** (parse, tokenize, lemmatize / mark / native
translate); Flutter paints, gestures, thin FRB orchestration, and Drift persist
for vocab / library. **Exception:** video frame probe/extract is **FE-only**
under `lib/data/video/` (FFmpegKit native + web canvas) — no Rust/FRB video API.
Path package: `package:my_translator_app/...`.

## Specialized Topics

| Topic | File |
|-------|------|
| Tap selection + add/edit vocab sheets | [tap.md](tap.md) |
| Lemma / phrase highlight | [highlight.md](highlight.md) |
| Split view: zero cross-pane sync + own-anchor reading resume | [split-scroll.md](split-scroll.md) |
| EPUB book-mode (chapter pager + paragraphs) | [chapters.md](chapters.md) |
| Paged video frames + extract facade | [video-frames.md](video-frames.md) |

## When to Use

- Cue/line list, tap selection on original tokens, add-to-vocab sheet
- EPUB book-mode: paragraph list + bottom chapter pager; no video chrome
  (`reader/chapters`) — [chapters.md](chapters.md)
- Contiguous tap Add (expand / shrink / gap-flash; one cue/paragraph);
  in_vocab → edit sheet
- Vocab highlight by morph lemma **and** phrase windows (see highlight.md;
  OpenSpec `add-reader-lemma-highlight`)
- Split view: zero cross-pane sync, each pane preserves/restores **only its
  own** anchor (Show/Hide/Load/reopen); EPUB chapter index links top→bottom
  only through the top pager — [split-scroll.md](split-scroll.md)
- Optional paged video frames (`reader/video-frames`) — [video-frames.md](video-frames.md)
  — not for EPUB documents
- Empty / loading / no-translation / tokenize-degrade reader states

**Not this skill:** file pick/upload-only, vocab list screen CRUD chrome alone,
settings, FRB codegen mechanics, global `lib/` folder placement — see related
skills by name.

## Flutter vs Rust

| Concern | Owner |
|---------|--------|
| Parse SRT/VTT/ASS → cues; EPUB → chapters/paragraphs | Rust (`parse_subtitles` / `parse_epub`) |
| Batch tokenize cue/paragraph display strings (`tokenize_cues`) | Rust (`panko` wrap); Flutter caches |
| Morph lemmatize / lemma_key / is_in_vocab (morph + phrase) | Rust (`add-reader-lemma-highlight`); Flutter paints flags + passes morph/phrase key into Drift |
| Offline word/phrase translate (Vocab + Reader sheets) | Native: Rust FRB (`work-with-translate`); Web: JS behind `lib/data/translate/` facade |
| Video frame probe / JPEG extract | Flutter only — `lib/data/video/` (`VideoFrameExtractor`; FFmpegKit / web canvas) — **not** Rust |
| Video attachment + frame JPEG cache | Flutter Drift (`DocumentRepository` + `DocumentVideoFrames`) — see [video-frames.md](video-frames.md) |
| Dictionary CRUD persist | Flutter Drift (`DictionaryRepository`; morph lemma from call site) |
| Token spans for hit-test | Rust prepares; Flutter paints |
| Line list, gestures, selection chrome, CTA/Add/edit sheets | Flutter |
| Scroll controllers; each pane restores only its own anchor (no settle / video-page / cross-pane sync) | Flutter |
| Line pairing original↔translation | Target: Rust (paired data); MVP (`reader/split`): no pairing — each pane (subtitle cues or EPUB chapter/paragraph) keeps its own position; EPUB chapter index only links top→bottom via the top pager — no Rust pairing API |

Do **not** re-tokenize or invent morph rules in Dart. Char-index slicing for
paint uses `cueCharSlice` (runes), matching Rust Unicode scalar spans.

## Line Rendering

1. Reader shows **cues as lines** (one cue ≈ one row / block), not a raw blob.
2. **Cues-first (`reader/cues`):** timestamps (`HH:MM:SS,mmm`) + plain
   `text_lines` joined with `\n` when tokenize is not ready.
3. **Tokenize + mark (`reader/tap` / `reader/highlight`):** after successful
   batch + `markVocabTokens`, original pane builds selectable spans from cached
   `TokenDto` (surface + spans + `lemmaKey` + `isInVocab`). Interstitial text
   between words is restored from the cue string. Never split words with naive
   Dart regex. Translation pane stays plain (no highlight).
4. Prefer feature UI under `lib/features/reader/` (see work-with-pages /
   client-work-with-structure).

## Interactions (summary)

| Gesture / action | Behavior |
|------------------|----------|
| Tap non-vocab (empty) | Select `[T,T]`; show Add CTA |
| Tap adjacent non-vocab (same cue) | Expand contiguous range |
| Tap inside multi-token range | Shrink to `[T,T]` |
| Gap tap (same cue) | Reset to `[T,T]` + ~300ms flash |
| Other cue non-vocab | Move to that cue’s `[T,T]` |
| Tap in-vocab (`is_in_vocab`) | Edit sheet (D8 longest-phrase resolve; no Add CTA) — [highlight.md](highlight.md) |
| CTA | Add sheet → manual translation + context → Drift (single: morph; phrase: casefold(join)) |
| After Add/edit save or delete | Refresh marks via `markVocabTokens` |
| Drag / long-press select | **Later** (tap-contiguous is shipped) |
| Translate button on Add/Edit sheets (fill Translation) | Shipped path: OpenSpec `translate/offline` + design D4; not instant word card |
| Instant word card on every tap | **Later** (out of this translate change) |

Details: [tap.md](tap.md).

## Highlight by Lemma / Phrase

- Capability `reader/highlight` (OpenSpec `add-reader-lemma-highlight`).
- Match = morph **lemma** singles **or** consecutive phrase windows; paint via
  `ReaderColors.lemmaHighlight`.
- Details: [highlight.md](highlight.md).

## Split View — Zero Cross-Pane Sync

- Top: original cues **or** EPUB paragraphs; bottom: translation (same family).
- **Design D9: no one-shot sync, ever.** Show split, Hide split, successful
  Load translation, and reopen all leave each pane's scroll untouched by the
  other pane — no `nearestCueIndex`-driven jump, no chapter/paragraph align.
  `_syncPanes` / `_syncTranslationToTimeMs` / `_syncBookPanes` /
  `_alignTranslationChapterToOriginal` do not exist.
- **Subtitles (`reader/split`):** two fully independent cue lists (two
  `parseSubtitles`); each pane seeds/restores **only its own** position —
  empty bottom + load CTA until a translation is loaded; **no** settle/fling
  realtime sync; **no** Rust pairing API.
- **EPUB (`reader/chapters` + split):** EPUB↔EPUB only; chapter **index** is
  linked top→bottom **only** via the top pager (prev/next) or at reopen
  (clamp to the restored original chapter) — design D10; the bottom pane has
  **no pager/title of its own** (RC-11). Changing chapter via the top pager
  rewinds **both** paragraph lists to index 0 (RC-12), it does not jump to a
  captured anchor. Paragraph **position within the chapter** is each pane's
  own seed/persist, never derived from the other. Unequal chapter counts →
  clamp + notice; unequal paragraph counts on restore → silent clamp (RS-22) —
  [chapters.md](chapters.md), [split-scroll.md](split-scroll.md).
- **AppBar:** split toggle **and Delete translation** (design D12, moved out
  of bottom-pane chrome); when `reader/video-frames` is in scope also
  Load/Remove video **next to** split (**hidden for EPUB**).
- **Bottom pane is text-only**, flush to the divider — no delete bar, no
  chapter title/pager chrome (RS-23 / D12).
- **Show / Hide split:** remounting a list resets its scroll — capture that
  pane's own top-visible anchor and `jumpTo` on remount (RS-10/11) for
  **both top and bottom**, subtitles and EPUB book alike; Hide also persists
  the bottom's own anchor immediately. With video, keep `VideoFramePages` /
  key outside remount ([video-frames.md](video-frames.md)).
- **Reading resume (RS-24 / LD-31/32):** Library loads `ReadingAnchor` for
  **both** original and (if linked) translation document ids →
  `initialReadingAnchor` **and** `initialTranslationReadingAnchor`; subtitles
  = each pane's own cue `startMs`; EPUB = original chapter index (unchanged,
  LD-28) + bottom's own paragraph index within the chapter resolved by
  clamp(original) (design D11 — reused `lastReadStartMs`/`lastReadAlignment`
  columns on the translation's own hidden Documents row; **no schema
  change**). Settle/Hide/dispose persist each pane's own anchor independently.
- **Video frames:** subtitle documents only; page change does **not** sync
  translation — [video-frames.md](video-frames.md).
- Token selection applies to the **original** pane only (RT-08).

Details: [split-scroll.md](split-scroll.md).

## Edge Cases

| Case | Contract |
|------|----------|
| Empty file / zero cues / empty EPUB | **Do not open Reader** (Library SnackBar / parse error) |
| Long lines | Wrap; token hit-test still correct |
| No translation yet (MVP) | Single-pane OK; split shows empty + load CTA; top scrolls alone |
| Toggle split Show/Hide (subtitles) | Each pane preserves/restores its own top-visible cue across remount; no sync (RS-10/11/18/24) |
| Toggle split Show/Hide (EPUB book) | Each pane preserves/restores its own mid-chapter paragraph scroll; no align to the other (RS-10/11/19/24) |
| Reopen document | Subtitles: each pane's own cue `startMs` + alignment; EPUB: original chapter index + bottom paragraph index at clamp(original); fail on either must not block open (RS-24 / LD-31/32) |
| EPUB unequal chapter counts | Clamp; show `mismatchedEpubChapterCountMessage` |
| EPUB unequal paragraph counts | Silent clamp on restore (RS-22); no error chrome |
| Delete translation | AppBar, next to split toggle (D12) — never bottom-pane chrome |
| Scroll, either pane | Other pane stays independent, always (RS-06/21) |
| Top EPUB pager, translation loaded | Bottom chapter = clamp(top); both paragraph lists rewind to 0 (RC-10/12) — bottom has no pager of its own (RC-11) |
| Video pages / dead link / extract / bad pick | See [video-frames.md](video-frames.md) — N/A for EPUB |
| Tokenize pending | Plain text; taps/selection off until cache ready |
| Tokenize FRB failure | Plain text + `readerTokenizeErrorSnack`; no CTA; no auto-retry |
| Non-word gaps | Not exported as tokens; ignore / no hit target |

## Related Skills (by name only)

- `work-with-pages` — screen/widget placement
- `work-with-frb` — FFI surface, codegen, call sites
- `work-with-tokenizematize` — Rust tokenize + lemmatize/mark
- `work-with-dictionary` / `work-with-drift` — vocab persist
- `work-with-styles` — colors/typography for highlight & selection
- `client-work-with-structure` — `lib/` feature map

Canon product scenarios: `{app}/AGENTS.md`.

## Checklist

- [ ] Original cues **or** EPUB chapters from Rust parse; tokenize batch on
      open (cache); book-mode → current chapter paragraphs
      ([chapters.md](chapters.md))
- [ ] Char spans → `cueCharSlice`; no Dart NLP / UTF-16 substring traps
- [ ] Contiguous Add selection (expand/shrink/gap-flash) + CTA + sheet → Drift
      (single morph / phrase casefold(join))
- [ ] Translation pane without token selection / vocab highlight
- [ ] Highlight (morph + phrase) + edit sheet (D8) from Rust `is_in_vocab`;
      refresh marks after dict mutation
- [ ] Split: zero cross-pane sync (D9) — no `_syncPanes`/`_syncTranslationToTimeMs`/
      `_syncBookPanes`/`_alignTranslationChapterToOriginal`; EPUB chapter index
      links top→bottom only via the top pager (D10 / RC-10/11/12)
- [ ] Show **and** Hide split preserve/restore **each pane's own** scroll
      (subtitles **and** EPUB mid-chapter, top **and** bottom, RS-10/11/24);
      Delete in AppBar next to split (D12, not bottom-pane chrome); bottom
      pane text-only flush to divider (RS-23)
- [ ] Reading anchor: restore both panes on open from their own persist;
      save each pane's own anchor on settle/Hide/dispose (format-gated,
      RS-24 / LD-31/32 — same generic columns, no schema change)
- [ ] If video frames in scope (subtitles only): follow
      [video-frames.md](video-frames.md) checklist
- [ ] Empty / long / no-translation / tokenize degrade / desync / EPUB mismatch
      handled
- [ ] Not extending scaffold `MyHomePage` demo
