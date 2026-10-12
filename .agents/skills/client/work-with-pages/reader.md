# Reader, tap→vocab sheet, highlight, split view

Read with the [core pages skill](SKILL.md) for navigation. Folder roles:
`client-work-with-structure` → `lib/features/reader/` (split may live here or
`lib/features/split/`). Tap/selection canon: `work-with-reader` / `tap.md`.
Highlight: `work-with-reader` / `highlight.md`.

## Screens / surfaces

| Surface | Path | Role |
|---------|------|------|
| Reader (cues + split + tap + highlight) | `lib/features/reader/reader_screen.dart` | Stateful; batch tokenize + mark; single-token Add; in-vocab edit; AppBar split toggle; delete translation in bottom-pane top chrome |
| Cue row | `lib/features/reader/cue_row.dart` | Timestamp + text; original pane token hit-targets + vocab highlight paint |
| Char slice helper | `lib/features/reader/cue_char_slice.dart` | Unicode scalar slice matching Rust char spans |
| Tokenize / mark seams | `lib/features/reader/tokenize_cues_fn.dart` (+ mark / lemma-keys typedefs) | Injectable FRB seams for tests |
| Nearest-time helper | `lib/features/reader/nearest_cue_index.dart` | Pure `nearestCueIndex` (`BigInt` `startMs`) |
| Time format | `lib/features/reader/cue_time_format.dart` | `HH:MM:SS,mmm` |
| Add-vocab sheet | local in `reader_screen.dart` (`_AddVocabSheet`) | Single-token surface + translation + context → Drift morph lemma |
| Edit-vocab sheet | local in `reader_screen.dart` | In-vocab tap: lemma/surface RO + translation/context + delete |
| Highlight paint | `cue_row` + `ReaderColors` | `is_in_vocab` → `lemmaHighlight` theme token |
| Split view (MVP: `reader/split`) | Reader **mode** (same screen; `scrollable_positioned_list` panes) | Top: original; bottom: loaded translation; time-nearest + alignment settle; RS-10/11 scroll preserve on Show **and** Hide; RS-12 resume via `initialReadingAnchor` |

Reader is usually **pushed** from the library ([upload.md](upload.md)), not a
shell tab. Ctor requires `dictionaryRepository` (Shell → Library → Reader).

## Reader UX canon

### Cues + split (current: `reader/cues`, `reader/split`)

- Show subtitle **cues** with timestamps start→end and `text_lines`.
- Full-list scroll; AppBar title = document title; back pops to Library.
- Open only after successful parse with ≥1 cue; empty / parse fail stays in Library.
- Split + reading resume — see [Split view](#split-view) below.

### Tap → vocabulary (current: `reader/tap`)

- Batch-tokenize **original** cues on open (`tokenizeCues`); mark with Drift
  morph lemmas (`markVocabTokens`); cache; plain text until ready / on FRB
  failure (`readerTokenizeErrorSnack`).
- **Single-token** selection on original → bottom CTA → Add sheet (manual
  translation; context = cue text) → `dictionaryRepository.add` with morph
  `lemmaKey` + `surfaceKey`. No contiguous multi-token Add in this change.
- Tap `is_in_vocab` → edit sheet (no Add CTA); see highlight section.
- Translation pane: no token selection / no vocab highlight.
- Details: `work-with-reader` / `tap.md`.

### Highlight (current: `reader/highlight`)

- Paint `is_in_vocab` tokens with `ReaderColors.lemmaHighlight` (light+dark).
- Refresh marks after Reader Add/edit save or delete (re-`markVocabTokens`).
- Details: `work-with-reader` / `highlight.md`.

### Later

- Offline translate / richer word-card chrome; phrase contiguous Add + phrase
  highlight.

## Add-vocab sheet

Minimum fields:

- Surface (read-only from single-token selection)
- Translation (required, initially empty — user-entered)
- Context (editable; prefill = full cue text)
- Save → Drift upsert by morph `lemmaKey` (+ `surfaceKey`); same policy as Vocab UI

Not a shell destination. Success closes sheet, clears selection, refreshes marks.

## Edit-vocab sheet (in-vocab)

- Prefill from Drift lookup by `lemma_key`; lemma/surface read-only;
  translation/context editable; Save upsert; Delete removes entry.
- Locked Keys: see OpenSpec change `add-reader-lemma-highlight` design D5.

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
ReaderScreen (single-pane or split mode)
  → widgets: cue list, token spans, CTA, add/edit sheets, empty+load, divider
  → orchestration: DocumentRepository + parseSubtitles; tokenizeCues +
      markVocabTokens (FRB); DictionaryRepository for sheet save/delete
      → FRB api (parse + tokenize + lemma mark); theme ReaderColors for paint
```

## Do not

- Invent unrelated chrome (catalog tiles, non-reader dashboards) outside reader UX.
- Parse SRT in the widget tree.
- Hand-edit generated FRB types for tokens — extend Rust `api/` + codegen.
- Open an instant word card with invented translation on every tap (Add = CTA + sheet; in-vocab = edit sheet).
- Contiguous multi-token Add / Dart-only membership as sole highlight truth.
