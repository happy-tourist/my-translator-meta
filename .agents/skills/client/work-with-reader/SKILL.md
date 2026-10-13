---
name: work-with-reader
description: >-
  Use when implementing or changing Translation Reader UX: cue/line rendering,
  word tap / contiguous token selection (phrase Add), bottom CTA + add-vocab
  sheet, vocab lemma/phrase highlight paint, edit sheet on in-vocab tap, split
  view original↔translation, or sync scroll. Not for scaffold demo TextField /
  processText. Cross-cut: work-with-pages (placement), work-with-frb
  (API/codegen), work-with-styles (tokens/theme).
---

# Work With Reader

## Overview

**Reader** contracts for **Translation Reader** (`my_translator_app`).
UI under `lib/features/reader/`: Stateful `ReaderScreen` (optional split),
`cue_row.dart`, `cue_char_slice.dart`, `tokenize_cues_fn.dart`,
`cue_time_format.dart`, `nearest_cue_index.dart`. Ctor takes already-parsed
`originalCues` + title + `originalDocumentId` + injected `DocumentRepository`
+ required `DictionaryRepository` (optional translation cues/id,
`initialReadingAnchor`, `ParseSubtitlesFn`, `TokenizeCuesFn`, optional
`MarkVocabTokensFn` / lemma-keys seam, file picker).
Library owns open/parse of the original (+ best-effort linked translation +
reading anchor) and passes the dictionary repo through. Scaffold demo in older
docs is **not** the reader — do not grow demo chrome into product UX.

Heavy compute stays in **Rust** (parse, tokenize, lemmatize / mark); Flutter
paints, gestures, thin FRB orchestration, and Drift persist for vocab. Path
package: `package:my_translator_app/...`.

## When to Use

- Cue/line list, tap selection on original tokens, add-to-vocab sheet
- Contiguous tap Add (expand / shrink / gap-flash; one cue); in_vocab → edit sheet
- Vocab highlight by morph lemma **and** phrase windows (see highlight.md;
  OpenSpec `add-reader-lemma-highlight`)
- Split view + synchronized scroll
- Empty / loading / no-translation / tokenize-degrade reader states

**Not this skill:** file pick/upload-only, vocab list screen CRUD chrome alone,
settings, FRB codegen mechanics, global `lib/` folder placement — see related
skills by name.

## Flutter vs Rust

| Concern | Owner |
|---------|--------|
| Parse SRT/VTT/ASS → cues | Rust |
| Batch tokenize cue display strings (`tokenize_cues`) | Rust (`panko` wrap); Flutter caches |
| Morph lemmatize / lemma_key / is_in_vocab (morph + phrase) | Rust (`add-reader-lemma-highlight`); Flutter paints flags + passes morph/phrase key into Drift |
| Offline word/phrase translate (Vocab + Reader sheets) | Native: Rust FRB (`work-with-translate`); Web: JS behind `lib/data/translate/` facade |
| Dictionary CRUD persist | Flutter Drift (`DictionaryRepository`; morph lemma from call site) |
| Token spans for hit-test | Rust prepares; Flutter paints |
| Line list, gestures, selection chrome, CTA/Add/edit sheets | Flutter |
| Scroll controllers + sync | Flutter |
| Line pairing original↔translation | Target: Rust (paired data); MVP (`reader/split`): Flutter time-nearest on two parse results — no Rust pairing API |

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

## Split View + Sync Scroll

- Top: original cues; bottom: translation cues (or L/R on wide — same rules).
- **MVP (`reader/split`):** two independent cue lists (two `parseSubtitles`);
  sync by **nearest `CueDto.startMs`** (`BigInt`), then optional pixel settle —
  empty bottom + load CTA until a translation file is attached; **no** per-cue
  placeholders and **no** Rust pairing API.
- **AppBar:** split toggle only. **Delete translation** lives in bottom-pane
  top chrome (not AppBar) when cues or a stuck link exist.
- **Show / Hide split:** remounting the original list resets scroll — capture
  the top-visible anchor and `jumpTo` after layout on **both** directions
  (RS-10, RS-11).
- **Reading resume (RS-12 / LD-17):** Library loads `ReadingAnchor` and passes
  `initialReadingAnchor`; Reader restores after layout; persist on user settle
  / dispose (skip while `_syncing` unless forced pop).
- **Target (later):** line-index aligned when Rust supplies paired lists;
  missing translation → placeholder row that keeps index alignment.
- Token selection applies to the **original** pane only (RT-08).

Details: [split-scroll.md](split-scroll.md).

## Edge Cases

| Case | Contract |
|------|----------|
| Empty file / zero cues | **Do not open Reader** (Library SnackBar / empty-parse error); in-Reader empty state only if a later change allows opening empty docs |
| Long lines | Wrap; token hit-test still correct |
| No translation yet (MVP) | Single-pane OK; split shows empty + load CTA; top scrolls alone |
| Toggle split Show/Hide (MVP) | Preserve original top-visible cue across remount; on Show with translation, sync bottom after restore |
| Reopen document (MVP) | Resume original pane at saved `startMs` + alignment; fail must not block open |
| Delete translation (MVP) | Bottom-pane top chrome → empty + load; never AppBar Delete |
| Scroll desync (MVP) | Re-run time-nearest sync from last scrolled pane; avoid raw `offset` copy |
| Scroll desync (target paired) | Re-anchor both panes to same cue index |
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

- [ ] Original cues from Rust parse; tokenize batch once on open (cache)
- [ ] Char spans → `cueCharSlice`; no Dart NLP / UTF-16 substring traps
- [ ] Contiguous Add selection (expand/shrink/gap-flash) + CTA + sheet → Drift
      (single morph / phrase casefold(join))
- [ ] Translation pane without token selection / vocab highlight
- [ ] Highlight (morph + phrase) + edit sheet (D8) from Rust `is_in_vocab`;
      refresh marks after dict mutation
- [ ] Split sync: MVP time-nearest (`startMs`) + settle; target index+placeholders when pairing exists
- [ ] Show **and** Hide split preserve original scroll; Delete in bottom-pane chrome (not AppBar)
- [ ] Reading anchor: restore on open; save on settle/dispose (not while syncing)
- [ ] Empty / long / no-translation (empty+load MVP) / tokenize degrade / desync handled
- [ ] Not extending scaffold `MyHomePage` demo
