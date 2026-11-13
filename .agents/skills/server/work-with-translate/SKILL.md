---
name: work-with-translate
description: >-
  Offline word/phrase translation in Translation Reader: native Rust ORT
  (OPUS-MT EN→RU ONNX) behind thin FRB, Web Transformers.js exception behind
  the same Dart facade; session warm-up / shared ensure (see warmup.md). Use
  when implementing or changing offline translate modules, translate FRB API,
  language-pair args, AppShell warm-up hook, or distinguishing local offline
  engines from optional future AI providers. Not for Flutter card chrome alone
  (work-with-reader), not for ARB UI locale (work-with-localization), not for
  AI/cloud translate implementation.
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
- Session warm-up / shared `ensureOfflineTranslateReady` / responsive cold path
  ([warmup.md](warmup.md))

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
- Native **product** load (design D8 revised / D13): Flutter extracts required
  **flat** basenames from `AssetBundle` →
  `getApplicationSupportDirectory()/opus-mt-en-ru` (reuse by size heuristic;
  chunked write + yields; `path_provider`) → FRB
  `init_offline_translate_from_dir(model_dir)` → ORT `Session` from files.
  Bytes `init_offline_translate(Vec<OfflineModelAsset { name, bytes }>)` MAY
  remain for unit tests / compat — MUST NOT be the product cold path
  (avoids ~200MB Dart→FRB encode stall; TO-11 / TO-14).
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
  api/translate.rs     # thin FRB: init_offline_translate_from_dir (+ bytes), translate_text, kinds
  translate/           # OfflineTranslator impl(s), ORT session, errors
  tokenize/ lemmatize/ # peers — not owned here
  frb_generated.rs     # GENERATED — never edit
```

```text
api (FRB) → translate → (ORT session; no dict writes)
Flutter facade (lib/data/translate/):
  offline_translate_platform → createPlatform + ensure/warmUpOfflineTranslate
  offline_translate_native.dart       # process-wide ensure + extract→path init
  offline_translate_web.dart          # js_interop; awaits ensure before translate
  offline_translate_web_ensure.dart   # VM-testable shared Future (no js_interop)
  warm-up: AppShell fullscreen gate → warmUpOfflineTranslate (see warmup.md)
UI: Vocab + Reader sheets inject OfflineTranslateFn
    (archive add-offline-opus-translate design D4; not warm-up D4)
```

## Session warm-up (product path)

**Details:** [warmup.md](warmup.md) (+ shell start hook in
[work-with-pages/shell.md](../../client/work-with-pages/shell.md)).

Summary: **fullscreen loading gate** in `AppShell` blocks Library/Vocab until
`warmUpOfflineTranslate` resolves (success or fail — no SnackBar from the gate
either way; design D1); `ensureOfflineTranslateReady` is process-wide and
shared with Translate; native cold path **extract → app-local dir →
`init_offline_translate_from_dir`** with chunked disk write + yields (no
`Isolate.run` on bundle views; bytes FRB not the product path); Web uses
bridge `warmup` + one worker `translatorPromise`. Widget-test escape hatch
skips the gate entirely.
Canon: OpenSpec `translate/offline` (TO-09…TO-14).

## Thin FRB API (native — locked shapes)

| Export | Responsibility |
|--------|----------------|
| `init_offline_translate_from_dir` | `model_dir: String` → `Result<(), TranslateErrorKind>` — **product** cold path |
| `init_offline_translate` | `Vec<OfflineModelAsset>` → `Result<(), TranslateErrorKind>` — tests / compat |
| `translate_text` | `text, source_lang, target_lang` → `Result<TranslationResult, TranslateErrorKind>` |
| `is_offline_translate_ready` | `() -> bool` — skip re-init when session already live |

`TranslationResult { primary_gloss: String }` — non-empty on `Ok`.

Thin `api/` maps `model_dir` / named bytes into `translate/`; ORT file I/O
lives in the domain module (`from_model_dir` / `from_assets`).

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
2. Loading on button covers warm-up / lazy init + inference.
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
| Product cold path = full ONNX bytes→FRB | Extract → `init_offline_translate_from_dir`; bytes init MAY for tests |
| Heavy load in `init_app` | Session warm-up after first UI frame + shared ensure ([warmup.md](warmup.md)); lazy init on Translate as fallback (still not in `init_app`) |
| `Isolate.run` on AssetBundle model bytes | Chunked disk write + yields on UI isolate ([warmup.md](warmup.md)) |
| Gloss logic in `api/` or Dart (native) | `translate/` + thin export |
| Auto-save every lookup | Explicit dict Add/Save |
| Phrase = join of word glosses | One `translate_text` |
| Extending `process_text` into MT | New `api/translate` + domain module |
| Hand-edit generated FRB/Dart | Codegen after `api/` change |

## Checklist

- [ ] Offline-only path; no required API keys
- [ ] Native = Rust ORT; Web = Transformers.js exception documented
- [ ] `init_offline_translate_from_dir` (product) + bytes `init_offline_translate` (tests) + `translate_text`; wasm stubs
- [ ] Pair args explicit; en→ru only in v1
- [ ] Surface query; dict save separate
- [ ] Typed errors including `InferenceFailed`
- [ ] Fullscreen shell gate (success/fail both dismiss, no SnackBar) + process-wide `ensure`/`warmUp`; test escape hatch skips gate ([warmup.md](warmup.md))
- [ ] `cargo test` for TO-*; scaffold demo not abused
