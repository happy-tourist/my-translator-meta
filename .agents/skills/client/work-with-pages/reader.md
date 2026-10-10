# Reader, word card, highlight, split view

Read with the [core pages skill](SKILL.md) for navigation. Folder roles:
`client-work-with-structure` → `lib/features/reader/` (split may live here or
`lib/features/split/`).

## Screens / surfaces

| Surface | Path | Role |
|---------|------|------|
| Reader (cues-first + split MVP) | `lib/features/reader/reader_screen.dart` | Stateful; single-pane default; AppBar split toggle; delete translation in bottom-pane top chrome (D8) |
| Cue row | `lib/features/reader/cue_row.dart` | Timestamp + `text_lines`; key prefixes for original / translation |
| Nearest-time helper | `lib/features/reader/nearest_cue_index.dart` | Pure `nearestCueIndex` (`BigInt` `startMs`) |
| Time format | `lib/features/reader/cue_time_format.dart` | `HH:MM:SS,mmm` |
| Word / phrase card (later) | local widget e.g. `word_card.dart` or bottom sheet | Translation, lemma, «add to vocabulary» |
| Highlight paint (later) | reader widgets + data from Rust | Lemma-matched saved words tinted in text |
| Split view (MVP: `reader/split`) | Reader **mode** (same screen; `scrollable_positioned_list` panes) | Top: original; bottom: loaded translation; time-nearest + alignment settle; RS-10/11 scroll preserve on Show **and** Hide; RS-12 resume via `initialReadingAnchor` |

Reader is usually **pushed** from the library ([upload.md](upload.md)), not a
shell tab.

## Reader UX canon (target)

### Cues-first Reader (current product step: `reader/cues`)

- Show subtitle **cues** in order with **required** timestamps start→end
  (`HH:MM:SS,mmm` from `start_ms` / `end_ms`) and plain `text_lines`.
- Full-list scroll (no paging). AppBar title = document title; back pops to Library.
- Open only after successful parse with ≥1 cue; empty / parse fail stays in Library.
- **Do not** require tokenize / word card / lemma highlight in the cues-first change.

### Later Reader (tokenize / vocab — separate changes)

Split view file-load + time sync is **current** (capability `reader/split`) —
see [Split view](#split-view) below; do not treat split as “later” in this file.

- Text is tappable by **token** (and selectable for multi-word phrases).
- Tap / selection → open word/phrase card (overlay, sheet, or side panel —
  product choice; keep one pattern).
- Saved dictionary lemmas are **highlighted** wherever they appear, including
  other word forms — match key is **lemma from Rust**, not raw string equality
  in Dart.
- Reading resume for the original pane is **current** (LD-17 / RS-12) — see
  Split view below; do not treat scroll persist as “later” for the original.

## Word card

Minimum fields (target):

- Surface form + **lemma**
- Translation(s) (offline path from Rust)
- Context snippet (cue / phrase) when available
- Primary action: **add to vocabulary** (Rust dictionary API)
- Optional later: AI explain — only when Settings/AI is in scope

Card is a **widget**, not a top-level shell destination. Dismiss returns to
Reader without losing scroll when possible.

## Highlight

- Rust prepares token → lemma (and which lemmas are in the dictionary).
- Flutter paints styles (color/weight) from that data.
- Do not maintain a parallel Dart stemmer or substring highlighter for
  morphology.

## Split view

- Two panes: **original** and **translation**, typically stacked (mobile) or
  side-by-side (wide) — prefer stacked first unless a mock says otherwise.
- **MVP (`reader/split`):** translation = second subtitle file → same FRB
  `parseSubtitles` as Library open (not offline word-translate / AI). Sync by
  nearest `CueDto.startMs` then pixel settle; bottom empty + load until attached.
  Persist hidden linked document via Drift (`DocumentRepository`). Errors map
  like Library open (`ParseErrorKind`).
- **Delete** translation: bottom-pane top chrome only (not AppBar). AppBar keeps
  the split toggle. Stuck link without cues: Delete in that chrome, then Load.
- **Show / Hide split:** preserve original scroll (capture top-visible →
  post-frame `jumpTo`); do not reset the original list to cue 0 (RS-10, RS-11).
- **Reading resume:** optional `initialReadingAnchor` from Library; restore
  after layout; save on settle/dispose via `DocumentRepository.saveReadingAnchor`
  (RS-12 / LD-17).
- **Target (later):** sync **by cue index** when Rust supplies paired lists;
  placeholders keep alignment — see `work-with-reader` / `split-scroll.md`.
- Toggle Reader ↔ Split from Reader chrome (app bar action or mode switch);
  same document session. Inject `DocumentRepository` (+ optional parse fn for
  tests) into Reader like Library — do not parse SRT in widgets.

## Layering

```text
ReaderScreen (single-pane or split mode; optional split_*.dart widgets)
  → widgets: cue list, pane scrollables, empty+load, divider; later token/card
  → orchestration: Library open + Reader attach/parse translation via
      DocumentRepository + parseSubtitles; later tokenize/dict/translate
      → FRB api (parse reuse; tokenize/lemmatize/dict/translate later)
```

## Do not

- Invent unrelated chrome (catalog tiles, non-reader dashboards) outside reader UX.
- Parse SRT in the widget tree.
- Hand-edit generated FRB types for tokens — extend Rust `api/` + codegen.
