# Reader, word card, highlight, split view

Read with the [core pages skill](SKILL.md) for navigation. Folder roles:
`client-work-with-structure` → `lib/features/reader/` (split may live here or
`lib/features/split/`).

## Target screens / surfaces

| Surface | Suggested path | Role |
|---------|----------------|------|
| Reader | `lib/features/reader/reader_screen.dart` | Cue list / scrolling text; primary study surface |
| Word / phrase card | local widget e.g. `word_card.dart` or bottom sheet | Translation, lemma, «add to vocabulary» |
| Highlight paint | reader widgets + data from Rust | Lemma-matched saved words tinted in text |
| Split view | `split_reader_screen.dart` or Reader **mode** | Top: original; bottom: translation; sync scroll |

Reader is usually **pushed** from the library ([upload.md](upload.md)), not a
shell tab.

## Reader UX canon (target)

- Show subtitle **cues** in order (text; timestamps optional chrome).
- Text is tappable by **token** (and selectable for multi-word phrases).
- Tap / selection → open word/phrase card (overlay, sheet, or side panel —
  product choice; keep one pattern).
- Saved dictionary lemmas are **highlighted** wherever they appear, including
  other word forms — match key is **lemma from Rust**, not raw string equality
  in Dart.
- Scroll position / open document = reader session (persist later via Rust when
  specified).

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
- **Sync scroll**: scrolling one pane keeps the other aligned **by cue/line**,
  not by raw pixel guess alone when cue indices exist.
- Translation lines come from Rust (offline / later AI) — Flutter does not
  invent a second parser.
- Toggle Reader ↔ Split from Reader chrome (app bar action or mode switch);
  same document session.

## Layering

```text
ReaderScreen / SplitReaderScreen
  → widgets: cue list, token span, word card, pane scrollables
  → orchestration: load document, request tokens/highlights, translate, dict add
      → FRB api (parse already done; tokenize/lemmatize/dict/translate)
```

## Do not

- Invent unrelated chrome (catalog tiles, non-reader dashboards) outside reader UX.
- Parse SRT in the widget tree.
- Hand-edit generated FRB types for tokens — extend Rust `api/` + codegen.
