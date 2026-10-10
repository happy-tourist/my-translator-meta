---
name: work-with-reader
description: >-
  Use when implementing or changing Translation Reader UX: cue/line rendering,
  word tap, phrase selection, word/phrase card, vocab lemma highlight paint,
  split view original↔translation, or sync scroll. Not for scaffold demo
  TextField / processText. Cross-cut: work-with-pages (placement), work-with-frb
  (API/codegen), work-with-styles (tokens/theme).
---

# Work With Reader

## Overview

**Reader** contracts for **Translation Reader** (`my_translator_app`).
**Cues-first** UI is under `lib/features/reader/`: Stateful `ReaderScreen`
(optional split), `cue_row.dart`, `cue_time_format.dart`,
`nearest_cue_index.dart`. Ctor takes already-parsed `originalCues` + title +
`originalDocumentId` + injected `DocumentRepository` (optional translation
cues/id, `ParseSubtitlesFn`, file picker). Library owns open/parse of the
original (+ best-effort linked translation). Scaffold demo in older docs is
**not** the reader — do not grow demo chrome into product UX.

Heavy logic stays in **Rust**; Flutter paints, gestures, and thin FRB
orchestration. Path package: `package:my_translator_app/...`.

## When to Use

- Cue/line list, tap-to-word, phrase drag-select
- Word/phrase card (lemma, translate, add to vocab)
- Vocab highlight by lemma across word forms
- Split view + synchronized scroll
- Empty / loading / no-translation reader states

**Not this skill:** file pick/upload-only, vocab list screen CRUD chrome alone,
settings, FRB codegen mechanics, global `lib/` folder placement — see related
skills by name.

## Flutter vs Rust

| Concern | Owner |
|---------|--------|
| Parse SRT/VTT/ASS → cues | Rust |
| Tokenize + lemmatize per line | Rust |
| Dictionary truth + lemma match set | Rust |
| Offline word/phrase translate | Rust |
| Token spans / highlight flags for UI | Rust prepares; Flutter paints |
| Line list, gestures, selection chrome | Flutter |
| Word/phrase card UI | Flutter |
| Scroll controllers + sync | Flutter |
| Line pairing original↔translation | Target: Rust (paired data); MVP (`reader/split`): Flutter time-nearest on two parse results — no Rust pairing API |

Do **not** re-tokenize, lemmatize, or invent dictionary match rules in Dart.

## Line Rendering

1. Reader shows **cues as lines** (one cue ≈ one row / block), not a raw blob.
2. **Cues-first (`reader/cues`):** render plain `text_lines` + timestamps
   (`HH:MM:SS,mmm` start→end). Token spans / tap / highlight are **later**
   changes — do not invent Dart tokenizers to “prepare” for them.
3. **When tokenize lands:** each line is built from **token spans** from Rust
   (text + lemma id / offsets + “in vocab” flag). Flutter maps spans to
   `TextSpan` / custom painter — never split words with naive Dart regex for
   product highlight/tap. Long lines wrap; tap hit-testing must still resolve
   to the correct token (span geometry from the same token list).
4. Prefer feature UI under `lib/features/reader/` (see work-with-pages /
   client-work-with-structure).

## Interactions (summary)

| Gesture | Behavior |
|---------|----------|
| Single tap on token | Open **word card** for that token (lemma + translate + add) |
| Drag / long-press select across tokens | **Phrase** selection → phrase translate card (whole span) |
| Tap outside / dismiss | Close card; clear transient selection |

Details: [tap.md](tap.md).

## Highlight by Lemma

- Match key = **lemma**, not surface form (`run` / `running` / `ran`).
- Rust marks which tokens are in the user’s dictionary; Flutter only applies
  highlight style.
- After add/remove vocab, refresh marks from Rust (or accept a pushed update) —
  do not keep a divergent Dart “highlighted lemmas” set as source of truth.

Details: [highlight.md](highlight.md).

## Split View + Sync Scroll

- Top: original cues; bottom: translation cues (or L/R on wide — same rules).
- **MVP (`reader/split`):** two independent cue lists (two `parseSubtitles`);
  sync by **nearest `CueDto.startMs`** (`BigInt`), then optional pixel settle —
  empty bottom + load CTA until a translation file is attached; **no** per-cue
  placeholders and **no** Rust pairing API.
- **Target (later):** line-index aligned when Rust supplies paired lists;
  missing translation → placeholder row that keeps index alignment.

Details: [split-scroll.md](split-scroll.md).

## Edge Cases

| Case | Contract |
|------|----------|
| Empty file / zero cues | **Do not open Reader** (Library SnackBar / empty-parse error); in-Reader empty state only if a later change allows opening empty docs |
| Long lines | Wrap; token hit-test and highlight still correct |
| No translation yet (MVP) | Single-pane OK; split shows empty + load CTA; top scrolls alone |
| Scroll desync (MVP) | Re-run time-nearest sync from last scrolled pane; avoid raw `offset` copy |
| Scroll desync (target paired) | Re-anchor both panes to same cue index |
| Tokenize pending | Show plain text or skeleton; disable tap until spans ready |
| Unknown / non-word tap | Ignore or no-op — do not open empty card |

## Related Skills (by name only)

- `work-with-pages` — screen/widget placement
- `work-with-frb` — FFI surface, codegen, call sites
- `work-with-styles` — colors/typography for highlight & card
- `client-work-with-structure` — `lib/` feature map

Canon product scenarios: `{app}/AGENTS.md`.

## Checklist

- [ ] Lines from Rust cues + token spans; no Dart NLP
- [ ] Tap → word; selection → phrase; card wired via FRB orchestration
- [ ] Highlight by lemma flags from Rust
- [ ] Split sync: MVP time-nearest (`startMs`) + settle; target index+placeholders when pairing exists
- [ ] Empty / long / no-translation (empty+load MVP) / desync handled
- [ ] Not extending scaffold `MyHomePage` demo
