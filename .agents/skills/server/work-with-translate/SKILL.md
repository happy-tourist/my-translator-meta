---
name: work-with-translate
description: >-
  Offline word/phrase translation in Translation Reader: native Rust ORT
  (OPUS-MT EN→RU ONNX) behind thin FRB, Web Transformers.js exception behind
  the same Dart facade. Use when implementing or changing offline translate
  modules, translate FRB API, language-pair args, or distinguishing local
  offline engines from optional future AI providers. Not for Flutter card
  chrome alone (work-with-reader), not for ARB UI locale
  (work-with-localization), not for AI/cloud translate implementation.
---

# Work With Translate (offline)

Target **offline word/phrase translation** for **Translation Reader**
(`my_translator_app`). Resolve `{app}` via `docs/projects-map.md` (key `app`).
Read `{app}/AGENTS.md` before changing the crate.

**Product engine (locked for current OpenSpec):** OPUS-MT **en→ru** ONNX
(`onnx-community/opus-mt-en-ru`, CC-BY 4.0) in app assets.

| Platform | Compute | Notes |
|----------|---------|--------|
| **Native** | Rust `ort` + tokenizer in `{app}/rust/src/translate/`, thin FRB | Must follow this skill |
| **Web** | Transformers.js worker + `dart:js_interop` | **Intentional exception** — not Rust WASM; same Dart facade + error kinds |

Demo `process_text` / `add_numbers` are unrelated — do not grow them into MT.

## When to Use

- Offline translate of a **word** or **phrase** (surface as displayed)
- Language-pair args for translate direction (learning content source → target)
- Designing / changing thin `rust/src/api/translate` exports
- Wiring gloss into Vocab / Reader Add-Edit (save still Drift, not translate)

**Not this skill:**

| Concern | Use instead |
|---------|-------------|
| Word/phrase **card UI**, gestures, tap/select | `work-with-reader` (+ `tap.md`) |
| Flutter error chrome (SnackBar / card error slot) | `client-work-with-errors` |
| FRB codegen / `RustLib.init` mechanics | `work-with-frb` |
| UI chrome locale (ARB) | `work-with-localization` |
| Rust test harness layout | `server-work-with-test` |
| **Optional AI** explain/translate providers | Out of scope — boundary only (below) |
| Web worker / COOP-COEP run | `work-with-env-deploy` + OpenSpec design |

## Core Principles

1. **Offline by default.** No network for gloss after install with assets. No
   required API keys at app start.
2. **Native heavy logic in Rust.** On native, Flutter must not invent glosses
   or run ONNX in Dart. On Web, JS engine is the documented exception behind
   one facade.
3. **Thin FRB API.** Public surface in `rust/src/api/`; algorithms in
   `translate/`. Never hand-edit `frb_generated.rs` or `lib/src/rust/**`.
4. **One string in → one gloss out.** Word and phrase share `translate_text`
   (MT string in); do not join per-token glosses in UI.
5. **Scaffold is not product.** Add `translate` + thin `api` + tests together.

## Boundary: Optional AI (out of scope)

- Do **not** implement cloud providers, API-key settings, or prompt pipelines
  under this skill.
- Offline path must remain usable **without** any AI provider.
- Mention AI only as “later / explicit / optional”.

## Engine + packaging (locked)

- Model: OPUS-MT en→ru ONNX in `{app}/assets/models/opus-mt-en-ru/`
  (`onnx-community/opus-mt-en-ru`, CC-BY 4.0). See that folder’s `README.md`.
- **Native required basenames** (FRB `OfflineModelAsset.name`):
  `tokenizer.json`, `encoder_model_uint8.onnx`, `decoder_model_uint8.onnx`
  (paths under `assets/models/opus-mt-en-ru/` + `onnx/`; pubspec must list
  both dirs — Flutter assets are non-recursive).
- **Web-only extra:** `decoder_model_merged_uint8.onnx` (+ `config.json` /
  `generation_config.json` / `tokenizer_config.json` for Transformers.js local
  load). Worker: `web/translate_worker.js`; bridge: `web/translate_bridge.js`
  (script in `web/index.html`); lib pin under `web/transformers/`.
- Native load: Flutter `AssetBundle` bytes → FRB
  `init_offline_translate(Vec<OfflineModelAsset { name, bytes }>)`.
  Public API MUST NOT take OS model paths.
- Cargo: ORT/tokenizer deps under
  `[target.'cfg(not(target_arch = "wasm32"))'.dependencies]`
  (`ort` 2.0.0-rc.14; `tokenizers` with `fancy-regex`, no default `onig`;
  `ndarray`); wasm32 stubs of the same FRB symbols → `EngineUnavailable`.
- Do **not** put heavy model load in `#[frb(init)] init_app`.

Trait (internal; names may vary):

```rust
// rust/src/translate/mod.rs

pub trait OfflineTranslator: Send + Sync {
    fn translate(
        &self,
        text: &str,
        source_lang: &str,
        target_lang: &str,
    ) -> Result<TranslationResult, TranslateError>;
}
```

## Language Pair Settings

| Setting | Role | Owner |
|---------|------|--------|
| Source / target lang | Learning pair for gloss | Explicit FRB/facade args (`en` / `ru` v1) |
| UI locale (ARB) | App chrome strings | Flutter — **not** this skill |

Rules:

- Prefer passing pair on each call; normalize casefold to lower ASCII.
- v1 supported pair only: `en` → `ru`. Else `UnsupportedLanguagePair`.
- Do not conflate UI locale with learning pair.

## Integration: Tokenize / Lemmatize / Dictionary

- **Translate query** for Vocab/Reader Translate button = **surface as
  displayed** (no separate morph re-stem in translate for this product path).
- Lemma remains for dict identity / highlight — owned by tokenize/lemmatize +
  Drift; translate does not write dict.
- Phrase: one `translate_text` for the whole string; never token-join glosses.
- Save to vocab is explicit user Add/Save — never auto on translate.

## Target Module Layout

```text
my_translator_app/rust/src/
  api/translate.rs     # thin FRB: init_offline_translate, translate_text, kinds
  translate/           # OfflineTranslator impl(s), ORT session, errors
  tokenize/ lemmatize/ # peers — not owned here
  frb_generated.rs     # GENERATED — never edit
```

```text
api (FRB) → translate → (ORT session; no dict writes)
Flutter facade (lib/data/translate/):
  offline_translate_platform → native | web
  native: AssetBundle bytes → init_offline_translate → translate_text
  web: JS worker (no FRB translate calls)
UI: Vocab + Reader sheets inject OfflineTranslateFn (design D4)
```

## Thin FRB API (native — locked shapes)

| Export | Responsibility |
|--------|----------------|
| `init_offline_translate` | `Vec<OfflineModelAsset>` → `Result<(), TranslateErrorKind>` |
| `translate_text` | `text, source_lang, target_lang` → `Result<TranslationResult, TranslateErrorKind>` |
| `is_offline_translate_ready` | `() -> bool` — skip re-init when session already live |

`TranslationResult { primary_gloss: String }` — non-empty on `Ok`.

Keep `api/` free of file-path I/O; only named bytes.

## Errors

| Kind | When |
|------|------|
| `EmptyQuery` | Empty / whitespace-only text |
| `UnsupportedLanguagePair` | Not en→ru after normalize |
| `EngineUnavailable` | No init / bad assets / session fail / wasm stub |
| `InferenceFailed` | Session up, inference failed |
| `Internal` | Unexpected — safe message outward |

`NotFound` is for dictionary-style packs only — **not** used for OPUS-MT.

Rules:

- `Result` / typed kinds over panics for user input.
- Flutter Translate button: **SnackBar** + Keys (design D4.5) — not
  `vocabFormError` / sheet validation slots; see `client-work-with-errors`.
- No network-required errors on the offline path.

## Tests

Follow **server-work-with-test**. `#[cfg(test)]` next to `translate/`:

- Word + phrase → non-empty `primary_gloss` (fixture/fake or real after spike)
- Empty → `EmptyQuery`; bad pair → `UnsupportedLanguagePair`
- No init / bad assets → `EngineUnavailable`; inference fail → `InferenceFailed`
- Fake `OfflineTranslator` for api tests without full ORT in CI when needed

Run `cargo test` from `{app}/rust`. OpenSpec TO-* must map to `cargo test`,
not `cargo check` alone. Web UX covered by Flutter injectable facade tests.

## Flutter UX Expectations (do not implement UI here)

1. Translate button on Vocab add + Reader Add/Edit sheets → fill Translation.
2. Loading on button covers lazy init + inference.
3. No Dart-invented glosses on native; Web uses JS engine only via facade.
4. Attribution OPUS-MT + accuracy disclaimer — UI copy, not Rust.
5. Placement, Keys, injectable `OfflineTranslateFn`, SnackBar errors, disable
   rules — OpenSpec change design **D4** (`add-offline-opus-translate` or
   synced main specs); keep skills from inventing alternate Keys.

## Anti-Patterns

| Avoid | Prefer |
|-------|--------|
| Cloud/AI as default gloss | Offline OPUS path; AI later, separate |
| ORT linked into wasm32 FRB | target-conditional deps + stubs |
| Model OS paths on FRB | Flutter bytes → `init_offline_translate` |
| Heavy load in `init_app` | Explicit init on first Translate |
| Gloss logic in `api/` or Dart (native) | `translate/` + thin export |
| Auto-save every lookup | Explicit dict Add/Save |
| Phrase = join of word glosses | One `translate_text` |
| Extending `process_text` into MT | New `api/translate` + domain module |
| Hand-edit generated FRB/Dart | Codegen after `api/` change |

## Checklist

- [ ] Offline-only path; no required API keys
- [ ] Native = Rust ORT; Web = Transformers.js exception documented
- [ ] `init_offline_translate` + `translate_text`; wasm stubs
- [ ] Pair args explicit; en→ru only in v1
- [ ] Surface query; dict save separate
- [ ] Typed errors including `InferenceFailed`
- [ ] `cargo test` for TO-*; scaffold demo not abused
