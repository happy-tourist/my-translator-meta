# Reader, tap→vocab sheet, highlight, split view

Read with the [core pages skill](SKILL.md) for navigation. Folder roles:
`client-work-with-structure` → `lib/features/reader/` (split may live here or
`lib/features/split/`). Tap/selection canon: `work-with-reader` / `tap.md`.
Highlight: `work-with-reader` / `highlight.md`.

## Screens / surfaces

| Surface | Path | Role |
|---------|------|------|
| Reader (cues / EPUB book-mode + split + tap + highlight) | `lib/features/reader/reader_screen.dart` | Stateful; cues **or** chapter paragraphs + bottom chapter pager; batch tokenize + mark; contiguous Add (phrase); in-vocab edit; AppBar split toggle (+ load/remove video for **subtitles** only); delete translation in bottom-pane top chrome |
| Cue row | `lib/features/reader/cue_row.dart` | Timestamp + text; original pane token hit-targets + vocab highlight paint; under-frame embed: `showTimestamp: true` (file start→end), `padding: EdgeInsets.zero`, `keySuffix` (VF-04 / VF-12) |
| Char slice helper | `lib/features/reader/cue_char_slice.dart` | Unicode scalar slice matching Rust char spans |
| Tokenize / mark seams | `lib/features/reader/tokenize_cues_fn.dart` (+ mark / lemma-keys typedefs) | Injectable FRB seams for tests |
| Nearest-time helper | `lib/features/reader/nearest_cue_index.dart` | Pure `nearestCueIndex` (`BigInt` `startMs`); reuse for page→translation sync |
| Time format | `lib/features/reader/cue_time_format.dart` | `HH:MM:SS,mmm` |
| Video frame pages (`reader/video-frames`) | `lib/features/reader/video_frame_pages.dart` (+ `video_frame_page_math.dart`, `video_second_math.dart`, `cue_text_at_second.dart`, `video_frame_cache.dart`) | Vertical column of ≤3 frames + bottom pager **replacing** original cues; injectable extractor/cache |
| Video extract facade | `lib/data/video/` (`video_frame_extractor*.dart`, `video_usable_path*.dart`, `reader_video_picker.dart`, `video_frame_errors.dart`) | Native FFmpegKit / web canvas; mirror translate platform split; no Rust video API |
| Add-vocab sheet | local in `reader_screen.dart` (`_AddVocabSheet`) | Surface (single or phrase join) + translation + context → Drift (morph / phrase key) |
| Edit-vocab sheet | local in `reader_screen.dart` | In-vocab tap: lemma/surface RO + translation/context + delete |
| Highlight paint | `cue_row` + `ReaderColors` | `is_in_vocab` → `lemmaHighlight` theme token |
| Split view (`reader/split` / EPUB chapters) | Reader **mode** (same screen; `scrollable_positioned_list` panes) | Subtitles: time-nearest + alignment settle; EPUB: shared chapter index + pager; RS-10/11 scroll preserve on Show/Hide (subtitles); resume via `initialReadingAnchor` (cue ms or chapter index) |

Reader is usually **pushed** from the library ([upload.md](upload.md)), not a
shell tab. Ctor requires `dictionaryRepository` (Shell → Library → Reader).

## Reader UX canon

### Cues + split (current: `reader/cues`, `reader/split`)

- Show subtitle **cues** with timestamps start→end and `text_lines`.
- Full-list scroll; AppBar title = document title; back pops to Library.
- Open only after successful parse with ≥1 cue; empty / parse fail stays in Library.
- Split + reading resume — see [Split view](#split-view) below.

### EPUB book-mode (current: `reader/chapters`)

- `documentFormat == epub`: paragraph list of current chapter + **bottom
  chapter pager**; no cue timestamps; no video chrome.
- Library passes `originalChapters` (not empty-cue hack). Tokenize current
  chapter paragraphs; invalidate on chapter change.
- Split: EPUB↔EPUB only; sync by chapter index; mismatch notice when counts
  differ. Details: `work-with-reader` /
  [chapters.md](../work-with-reader/chapters.md).

### Tap → vocabulary (current: `reader/tap`)

- Batch-tokenize **original** cues on open (`tokenizeCues`); mark with Drift
  lemmas (`markVocabTokens`: morph + phrase); cache; plain text until ready /
  on FRB failure (`readerTokenizeErrorSnack`).
- **Contiguous tap** selection on original (expand / shrink / gap-flash; one
  cue) → bottom CTA → Add sheet (manual translation; context = cue text) →
  `dictionaryRepository.add`: single → morph `lemmaKey` + `surfaceKey`;
  phrase → casefold(join) for both.
- Tap `is_in_vocab` → edit sheet (D8 longest-phrase resolve; no Add CTA).
- Translation pane: no token selection / no vocab highlight.
- Details: `work-with-reader` / `tap.md`.

### Highlight (current: `reader/highlight`)

- Paint `is_in_vocab` tokens with `ReaderColors.lemmaHighlight` (light+dark)
  — morph singles and all tokens in matched phrase spans.
- Refresh marks after Reader Add/edit save or delete (re-`markVocabTokens`).
- Details: `work-with-reader` / `highlight.md`.

### Later

- Instant word/phrase card chrome (beyond Translate-on-sheet); drag /
  long-press select.

## Add-vocab sheet

Minimum fields:

- Surface (read-only from selection: single surface or phrase join)
- **Translate** (`readerAddVocabTranslate`) → injectable `OfflineTranslateFn`
  (surface as displayed); fills Translation; loading on button; stays disabled
  until sheet closes after first start; errors → SnackBar
  `readerAddVocabTranslateErrorSnack` (not validation slot)
- Attribution / disclaimer Keys (`readerAddVocabTranslateAttribution`) —
  copy from `lib/data/translate/offline_translate.dart` (D4.6)
- Translation (required, initially empty — user or Translate)
- Context (editable; prefill = full cue text)
- Save → Drift upsert: single morph `lemmaKey` + `surfaceKey`; phrase
  casefold(join); same policy as Vocab UI

Not a shell destination. Success closes sheet, clears selection, refreshes marks.
Engine details: meta `work-with-translate`.

## Edit-vocab sheet (in-vocab)

- Prefill from Drift lookup by `lemma_key`; lemma/surface read-only;
  translation/context editable; Save upsert; Delete removes entry.
- Same Translate pattern: `readerEditVocabTranslate` /
  `readerEditVocabTranslateErrorSnack` / `readerEditVocabTranslateAttribution`.
- Locked Keys: see OpenSpec `reader/tap` + `add-reader-lemma-highlight` design D5.

## Split view

- Two panes: **original** and **translation**, typically stacked (mobile) or
  side-by-side (wide) — prefer stacked first unless a mock says otherwise.
- **Subtitles (`reader/split`):** translation = second subtitle file →
  `parseSubtitles`. Sync by nearest `CueDto.startMs` then pixel settle;
  bottom empty + load until attached. Persist via Drift. Errors:
  `ParseErrorKind`.
- **EPUB:** translation = second `.epub` → `parseEpub`. Sync by shared chapter
  index; reject mixed families; unequal counts → clamp + notice.
- **Delete** translation: bottom-pane top chrome only (not AppBar). AppBar keeps
  the split toggle (+ video load/remove for **subtitles** when
  `reader/video-frames`). Stuck link: Delete in that chrome, then Load.
- **Show / Hide split:** preserve original scroll (capture top-visible →
  post-frame `jumpTo`); do not reset the original list to cue 0 (RS-10, RS-11).
  With video pages, keep `VideoFramePages` / key outside the remount branch so
  page state survives Show/Hide.
- **Reading resume:** optional `initialReadingAnchor` from Library; restore
  after layout; save on settle/dispose via `DocumentRepository.saveReadingAnchor`
  (RS-12 / LD-17).
- **Video frames (`reader/video-frames`):** attach/detach via Drift; Library open
  best-effort restores attachment; original pane = paged vertical triple
  (replaces cues); one-way page→translation when split — details
  `work-with-reader` / [video-frames.md](../work-with-reader/video-frames.md).
- **Target (later):** sync **by cue index** when Rust supplies paired lists;
  placeholders keep alignment — see `work-with-reader` / `split-scroll.md`.
- Toggle Reader ↔ Split from Reader chrome (app bar action or mode switch);
  same document session. Inject `DocumentRepository` (+ optional parse fn for
  tests) into Reader like Library — do not parse SRT in widgets.

## Layering

```text
ReaderScreen
  → single-pane or split mode body
  → original pane: VideoFramePages (when attached) OR cue list
  → widgets: cue list / frame slots, token spans, CTA, add/edit sheets,
      empty+load, divider
  → orchestration: DocumentRepository (+ video attach/cache); parseSubtitles;
      tokenizeCues + markVocabTokens (FRB); DictionaryRepository; VideoFrameExtractor
      → FRB api (parse + tokenize + lemma mark); theme ReaderColors for paint
      → video extract: Dart facade only (no FRB video)
```

## Do not

- Invent unrelated chrome (catalog tiles, non-reader dashboards) outside reader UX.
- Parse SRT in the widget tree.
- Hand-edit generated FRB types for tokens — extend Rust `api/` + codegen.
- Open an instant word card with invented translation on every tap (Add = CTA + sheet; in-vocab = edit sheet).
- Dart-only membership as sole highlight truth (use FRB `is_in_vocab`).
