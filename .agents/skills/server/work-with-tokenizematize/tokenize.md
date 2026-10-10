# Tokenize (shipped MVP)

Capability consumer: `reader/tap`. Lemmatize / `lemma_key` / `in_vocab` are
**out of this module’s MVP** — see [lemmatize.md](lemmatize.md).

## Layout

```text
my_translator_app/rust/src/
  tokenize/mod.rs          # panko wrap → TokenSpan (char spans)
  api/tokenize.rs           # TokenDto, CueTokensDto, tokenize_cues
```

- Pin: `panko = "0.2.0"` in `rust/Cargo.toml` (pure Rust; WASM/mobile OK).
- Thin FRB only in `api/`; domain mapping stays in `tokenize/`.
- After `api/` changes → `flutter_rust_bridge_codegen generate` (never hand-edit
  generated files).

## Locked FRB surface

```rust
pub struct TokenDto {
    pub surface: String,
    pub char_start: u32, // inclusive Unicode scalar index
    pub char_end: u32,   // exclusive
    pub token_index: u32, // 0-based among word tokens in this cue
}

pub struct CueTokensDto {
    pub cue_index: u32, // index into input `texts`
    pub tokens: Vec<TokenDto>,
}

pub fn tokenize_cues(texts: Vec<String>) -> Vec<CueTokensDto>;
```

- Infallible for valid UTF-8 `String`; empty / whitespace-only → empty `tokens`.
- No `lemma_key`, no `in_vocab`, no byte fields in this MVP.
- Reader MUST call **batch** `tokenize_cues` once on open (not per-row rebuild).

## Algorithm (display-string contract)

1. Input `S` = same string Flutter paints: `cue.text_lines.join('\n')`.
2. `panko::tokenize(&S)`; export only `Token::Word`; `Symbol` / `Newline`
   advance the cursor only.
3. Char spans from byte offsets via `chars().count()`; invariant:
   slice of `S` by char range == `surface` (RT-12).
4. Keep typical EN apostrophe forms (e.g. `don't`) as one word when panko does
   (RT-11); Cyrillic words as separate tokens.

## Flutter consumer

- Injectable seam: `TokenizeCuesFn` / `ReaderScreen.tokenizeCues`.
- Paint/hit-test with Unicode scalar slicing (`cueCharSlice`) — not UTF-16
  `substring`.
- FRB/call failure is a Flutter UX concern (plain text + SnackBar), not a typed
  domain `Result` from tokenize.

## Tests

- `cargo test` in `{app}/rust` on `tokenize/` (RT-11, RT-12, multiline join,
  batch order). Do not prove span rules only via Flutter widget tests.
