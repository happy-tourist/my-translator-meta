# Tokenize (shipped)

Capability consumer: `reader/tap` (+ highlight). Morph fields
(`lemma_key` / `is_in_vocab`) and mark / lemma-keys helpers live in the same
FRB module — see [lemmatize.md](lemmatize.md).

## Layout

```text
my_translator_app/rust/src/
  tokenize/mod.rs          # panko wrap → TokenSpan (char spans)
  lemmatize/mod.rs         # morph keys (called from api/tokenize mapping)
  api/tokenize.rs          # TokenDto, CueTokensDto, tokenize_cues,
                           # mark_vocab_tokens, lemma_keys_for_surfaces
```

- Pin: `panko = "0.2.0"` in `rust/Cargo.toml` (pure Rust; WASM/mobile OK).
- Thin FRB only in `api/`; domain mapping stays in `tokenize/` + `lemmatize/`.
- After `api/` changes → `flutter_rust_bridge_codegen generate` (never hand-edit
  generated files).

## Locked FRB surface

```rust
pub struct TokenDto {
    pub surface: String,
    pub char_start: u32, // inclusive Unicode scalar index
    pub char_end: u32,   // exclusive
    pub token_index: u32, // 0-based among word tokens in this cue
    pub lemma_key: String,
    pub is_in_vocab: bool, // false after tokenize_cues; set by mark_vocab_tokens
}

pub struct CueTokensDto {
    pub cue_index: u32, // index into input `texts`
    pub tokens: Vec<TokenDto>,
}

pub fn tokenize_cues(texts: Vec<String>) -> Vec<CueTokensDto>;
pub fn mark_vocab_tokens(
    cues: Vec<CueTokensDto>,
    vocab_lemmas: Vec<String>,
) -> Vec<CueTokensDto>;
pub fn lemma_keys_for_surfaces(surfaces: Vec<String>) -> Vec<String>;
```

- Infallible for valid UTF-8 `String`; empty / whitespace-only → empty `tokens`.
- Do not invent alternate morph field names.
- Reader MUST call **batch** `tokenize_cues` once on open (not per-row rebuild),
  then `mark_vocab_tokens` with Drift lemmas (morph singles **and** phrase keys).
  Mark logic lives in `lemmatize::mark_vocab_flags` — see [lemmatize.md](lemmatize.md).

## Algorithm (display-string contract)

1. Input `S` = same string Flutter paints: `cue.text_lines.join('\n')`.
2. `panko::tokenize(&S)`; export only `Token::Word`; `Symbol` / `Newline`
   advance the cursor only.
3. Char spans from byte offsets via `chars().count()`; invariant:
   slice of `S` by char range == `surface` (RT-12).
4. Map each word span → `TokenDto` with `lemma_key =
   lemmatize::lemma_key_for_surface(&surface)` and `is_in_vocab = false`.
5. Keep typical EN apostrophe forms (e.g. `don't`) as one word when panko does
   (RT-11); Cyrillic words as separate tokens.

## Flutter consumer

- Injectable seams: `TokenizeCuesFn` / `MarkVocabTokensFn` /
  `LemmaKeysForSurfacesFn` (`tokenize_cues_fn.dart`,
  `lemma_keys_for_surfaces_fn.dart`).
- Paint/hit-test with Unicode scalar slicing (`cueCharSlice`) — not UTF-16
  `substring`.
- FRB/call failure is a Flutter UX concern (plain text + SnackBar /
  soft mark-refresh SnackBar), not a typed domain `Result` from tokenize.

## Tests

- `cargo test` in `{app}/rust` on `tokenize/` + `lemmatize/` + `api/tokenize`
  mark helpers. Do not prove span / morph rules only via Flutter widget tests.
