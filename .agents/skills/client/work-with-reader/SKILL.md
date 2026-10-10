---
name: work-with-reader
description: >-
  Use when implementing or changing Translation Reader UX: cue/line rendering,
  word tap / contiguous selection, bottom CTA + add-vocab sheet, vocab lemma
  highlight paint (later), split view original↔translation, or sync scroll.
  Not for scaffold demo TextField / processText. Cross-cut: work-with-pages
  (placement), work-with-frb (API/codegen), work-with-styles (tokens/theme).
---

# Work With Reader

## Overview

**Reader** contracts for **Translation Reader** (`my_translator_app`).
UI under `lib/features/reader/`: Stateful `ReaderScreen` (optional split),
`cue_row.dart`, `cue_char_slice.dart`, `tokenize_cues_fn.dart`,
`cue_time_format.dart`, `nearest_cue_index.dart`. Ctor takes already-parsed
`originalCues` + title + `originalDocumentId` + injected `DocumentRepository`
+ required `DictionaryRepository` (optional translation cues/id,
`initialReadingAnchor`, `ParseSubtitlesFn`, `TokenizeCuesFn`, file picker).
Library owns open/parse of the original (+ best-effort linked translation +
reading anchor) and passes the dictionary repo through. Scaffold demo in older
docs is **not** the reader — do not grow demo chrome into product UX.

Heavy compute stays in **Rust** (parse, tokenize); Flutter paints, gestures,
thin FRB orchestration, and Drift persist for vocab. Path package:
`package:my_translator_app/...`.

## When to Use

- Cue/line list, tap selection on original tokens, add-to-vocab sheet
- Contiguous multi-tap phrase selection (one cue)
- Vocab highlight by lemma across word forms (**later** — see highlight.md)
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
| Morph lemmatize / lemma_key / in_vocab flags | **Later** Rust — MVP uses Flutter `normalizeLemma` stub on save |
| Offline word/phrase translate | **Later** Rust |
| Dictionary CRUD persist | Flutter Drift (`DictionaryRepository`) |
| Token spans for hit-test | Rust prepares; Flutter paints |
| Line list, gestures, selection chrome, CTA/sheet | Flutter |
| Scroll controllers + sync | Flutter |
| Line pairing original↔translation | Target: Rust (paired data); MVP (`reader/split`): Flutter time-nearest on two parse results — no Rust pairing API |

Do **not** re-tokenize or invent morph rules in Dart. Char-index slicing for
paint uses `cueCharSlice` (runes), matching Rust Unicode scalar spans.

## Line Rendering

1. Reader shows **cues as lines** (one cue ≈ one row / block), not a raw blob.
2. **Cues-first (`reader/cues`):** timestamps (`HH:MM:SS,mmm`) + plain
   `text_lines` joined with `\n` when tokenize is not ready.
3. **Tokenize (`reader/tap`):** after successful batch, original pane builds
   selectable spans from cached `TokenDto` (surface + `charStart`/`charEnd`).
   Interstitial text between words is restored from the cue string. Never split
   words with naive Dart regex for product tap. Translation pane stays plain.
4. Prefer feature UI under `lib/features/reader/` (see work-with-pages /
   client-work-with-structure).

## Interactions (summary)

| Gesture / action | Behavior (MVP) |
|------------------|----------------|
| Single tap on original token | Select word; show bottom CTA **Add to vocabulary** |
| Tap adjacent token (same cue) | Expand contiguous selection |
| Tap inside multi-token selection | Shrink to that word |
| Tap with gap / other cue | Reset anchor (gap → brief flash) |
| CTA | Open bottom sheet → manual translation + context → Drift save |
| Drag / long-press phrase | **Not** in MVP |
| Instant word card + offline gloss | **Later** |

Details: [tap.md](tap.md).

## Highlight by Lemma

- **Later** capability (`reader/highlight`). Match key = **lemma**, not surface.
- MVP save does **not** refresh highlight marks.
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
- `work-with-tokenizematize` — Rust tokenize (+ later lemmatize)
- `work-with-dictionary` / `work-with-drift` — vocab persist
- `work-with-styles` — colors/typography for highlight & selection
- `client-work-with-structure` — `lib/` feature map

Canon product scenarios: `{app}/AGENTS.md`.

## Checklist

- [ ] Original cues from Rust parse; tokenize batch once on open (cache)
- [ ] Char spans → `cueCharSlice`; no Dart NLP / UTF-16 substring traps
- [ ] Tap selection state machine + CTA + sheet → Drift `dictionaryRepository`
- [ ] Translation pane without token selection
- [ ] Highlight by lemma flags from Rust — when that change lands
- [ ] Split sync: MVP time-nearest (`startMs`) + settle; target index+placeholders when pairing exists
- [ ] Show **and** Hide split preserve original scroll; Delete in bottom-pane chrome (not AppBar)
- [ ] Reading anchor: restore on open; save on settle/dispose (not while syncing)
- [ ] Empty / long / no-translation (empty+load MVP) / tokenize degrade / desync handled
- [ ] Not extending scaffold `MyHomePage` demo
