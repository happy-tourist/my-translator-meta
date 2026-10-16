---
name: work-with-reader
description: >-
  Use when implementing or changing Translation Reader UX: cue/line rendering,
  EPUB book-mode (chapter pager + paragraph list; OpenSpec `reader/chapters`),
  word tap / contiguous token selection (phrase Add), bottom CTA + add-vocab
  sheet, vocab lemma/phrase highlight paint, edit sheet on in-vocab tap, split
  view original↔translation (one-shot time or EPUB chapter align; independent
  panes after), or optional paged video frames (OpenSpec `reader/video-frames`).
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
| Split view + one-shot sync + reading resume | [split-scroll.md](split-scroll.md) |
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
- Split view + one-shot align (subtitle time-nearest **or** EPUB chapter
  index on Show / load translation only) — [split-scroll.md](split-scroll.md)
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
| Scroll controllers + one-shot align (Show / load translation; no settle / video-page realtime) | Flutter |
| Line pairing original↔translation | Target: Rust (paired data); MVP (`reader/split`): Flutter time-nearest one-shot (subtitles) or dual EPUB chapter indices — no Rust pairing API |

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

## Split View + One-Shot Sync

- Top: original cues **or** EPUB paragraphs; bottom: translation (same family).
- **Subtitles (`reader/split`):** two independent cue lists (two
  `parseSubtitles`); **one-shot** align by **nearest `CueDto.startMs`**
  (`BigInt`) only on Show split or successful load translation — empty bottom
  + load CTA until attached; **no** settle/fling realtime sync; **no** Rust
  pairing API.
- **EPUB (`reader/chapters` + split):** EPUB↔EPUB only; **one-shot** clamp
  translation chapter to original on Show / load; after that **independent**
  `_originalChapterIndex` / `_translationChapterIndex` (pager is per-pane);
  unequal chapter counts → clamp + notice —
  [chapters.md](chapters.md), [split-scroll.md](split-scroll.md).
- **AppBar:** split toggle; when `reader/video-frames` is in scope also
  Load/Remove video **next to** split (**hidden for EPUB**). **Delete
  translation** stays in bottom-pane top chrome (not AppBar).
- **Show / Hide split (subtitles):** remounting the original list resets
  scroll — capture top-visible anchor and `jumpTo` (RS-10, RS-11); on Show
  with translation, one-shot bottom (RS-18). Book-mode: dual chapter indices
  in session state; Show one-shot-aligns translation (RS-19). With video,
  keep `VideoFramePages` / key outside remount ([video-frames.md](video-frames.md)).
- **Reading resume:** Library loads `ReadingAnchor` → `initialReadingAnchor`;
  subtitles = cue `startMs`; EPUB = **original** chapter index (same Drift
  column); do **not** sync translation on open alone (RS-12).
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
| Toggle split Show/Hide (subtitles) | Preserve original top-visible cue across remount; on Show with translation, one-shot bottom (RS-18) |
| Reopen document | Subtitles: cue `startMs` + alignment; EPUB: original chapter index; fail must not block open; no translation sync until Show |
| EPUB unequal chapter counts | Clamp; show `mismatchedEpubChapterCountMessage`; one-shot still runs |
| Delete translation (MVP) | Bottom-pane top chrome → empty + load; never AppBar Delete |
| Scroll after one-shot (subtitles) | Panes stay independent (RS-06); avoid raw `offset` copy |
| Chapter pager (EPUB) | Independent indices after one-shot (RS-14); pager is per-pane |
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
- [ ] Split one-shot: subtitles time-nearest (`startMs`); EPUB dual chapter indices
- [ ] Show **and** Hide split preserve original scroll (subtitles); Delete in
      bottom-pane chrome (not AppBar)
- [ ] Reading anchor: restore on open; save on settle/dispose (format-gated)
- [ ] If video frames in scope (subtitles only): follow
      [video-frames.md](video-frames.md) checklist
- [ ] Empty / long / no-translation / tokenize degrade / desync / EPUB mismatch
      handled
- [ ] Not extending scaffold `MyHomePage` demo
