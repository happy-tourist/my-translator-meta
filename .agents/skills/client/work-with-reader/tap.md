# Tap and Phrase Selection

## Word tap

1. Hit-test against **Rust token spans** for the cue (offsets / span ids), not
   against a single undifferentiated `String`.
2. Resolve: surface form, lemma id, cue/context id.
3. Orchestration calls Rust: lookup / offline translate for that lemma (and
   surface if needed).
4. Show **word card**: lemma, translation(s), context snippet, **Add to vocab**.
5. Add → Rust dictionary API → refresh highlight marks for the open document.

## Phrase selection

1. User selects a contiguous range of tokens (drag or platform select).
2. Selection is a list of token ids / start–end within the cue (or across cues
   only if product later allows — default: **within one cue**).
3. Orchestration sends the phrase string (and optional token ids) to Rust
   translate-phrase; do not translate token-by-token in Dart and concatenate.
4. Show **phrase card**: full phrase, translation, optional “add phrase” if API
   supports it; avoid forcing a single-lemma add for multi-word spans unless
   Rust defines that behavior.

## Word / phrase card

| UI | Data source |
|----|-------------|
| Headword / phrase text | Selection / token surface |
| Lemma | Rust |
| Translation | Rust offline (later: optional AI — explicit only) |
| Add / remove vocab | Rust dictionary |
| Loading / error | Flutter orchestration flags |

Card is Flutter chrome; no dictionary writes in Dart-only storage.

## Gesture rules

- Short tap without drag → word path (even if the OS starts a tiny selection —
  prefer token under pointer).
- Clear drag beyond threshold → phrase path.
- Second tap on same token while card open → keep or refresh; do not stack cards.
- Tap on already-highlighted vocab word → still open card (edit/remove path OK).

## Anti-patterns

| Avoid | Prefer |
|-------|--------|
| `onTap` on whole line only | Per-token (or span) hit-test |
| Dart stemming for “which word” | Rust lemma id from span |
| Card that invents translation locally | FRB translate / lookup |
| Phrase = join of per-word cards | One phrase translate call |
