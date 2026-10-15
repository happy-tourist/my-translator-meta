---
name: work-with-config
description: >-
  Use when adding, changing, reviewing, or debugging Translation Reader Rust /
  app config: rust/Cargo.toml (crate name, cdylib+staticlib, deps, features,
  FRB pin), flutter_rust_bridge.yaml keys (rust_input / rust_root /
  dart_output), or aligning flutter_rust_bridge Dart ↔ Rust ↔ CLI ≈ 2.13.0.
  Not for Flutter UI bind/init (client work-with-frb) or public FFI surface
  design (server work-with-frb-api).
---

# Work With Config

Use this skill for **Rust crate and FRB toolchain configuration** in
**my_translator_app** (product: **Translation Reader**).

The “server” here is the in-process **Rust core** (`rust/`) linked into
Flutter via **flutter_rust_bridge** (FRB). Contour knobs are Cargo
deps/features and FRB YAML paths.

Canon: `{app}/AGENTS.md` (`docs/projects-map.md` → key `app` →
`../my_translator_app`).

Stack: Rust edition `2021`, crate `rust_lib_my_translator_app`
(`cdylib` + `staticlib`), FRB **2.13.0** (Dart `pubspec.yaml` + Rust
`Cargo.toml` + codegen CLI ≈ same). Path package `rust_builder` → crate
`rust/`.

**This skill owns:** Cargo deps/features, crate name / `[lib]` crate-types,
`flutter_rust_bridge.yaml` keys, version-pin alignment Dart ↔ Rust ↔ CLI.

**Defer:** Flutter-side bind / `RustLib.init` / codegen call-site workflow →
client **`work-with-frb`**. Public FFI API surface design (`rust/src/api/`) →
server **`work-with-frb-api`**. Run/build/release → client
**`work-with-env-deploy`**.

## When to Use

- Add / bump / remove a Rust dependency in `rust/Cargo.toml`
- Introduce or change Cargo `[features]` (optional NLP/DB/translate crates)
- Touch crate name, `[lib]` `crate-type`, or FRB-related rustc lints
- Edit `flutter_rust_bridge.yaml` (`rust_input` / `rust_root` / `dart_output`)
- Diagnose FRB version drift across Dart, Cargo, and CLI
- Review config before adding lemma / translate crates (compute stack); do **not**
  add `rusqlite` for user data (persist = Drift in Flutter)

**Not this skill:** inventing product API functions; hand-editing generated
FRB files; inventing web `.env` / URL config for core; putting business logic
in `rust_builder/`.

## Quick Reference

| Case | Preferred pattern |
| --- | --- |
| Crate identity | `[package] name = "rust_lib_my_translator_app"` — keep in sync with `pubspec` path dep + `rust_builder` package name |
| Lib types | `crate-type = ["cdylib", "staticlib"]` — required for FRB FFI (mobile/desktop) |
| FRB Rust pin | `flutter_rust_bridge = "=2.13.0"` in `rust/Cargo.toml` |
| FRB Dart pin | `flutter_rust_bridge: 2.13.0` in app `pubspec.yaml` |
| FRB CLI | `flutter_rust_bridge_codegen` ≈ **2.13.0** (same major.minor.patch family) |
| FRB YAML | `rust_input: crate::api`, `rust_root: rust/`, `dart_output: lib/src/rust` |
| New Rust dep | Add under `[dependencies]` (or `[dev-dependencies]`); prefer features for optional/heavy crates |
| Secrets / `.env` | **None** for core Cargo/FRB config |
| Generated files | Never “fix” by editing `lib/src/rust/**` or `rust/src/frb_generated.rs` |

## Current Truth (scaffold)

### `rust/Cargo.toml`

```toml
[package]
name = "rust_lib_my_translator_app"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "staticlib"]

[dependencies]
flutter_rust_bridge = "=2.13.0"

[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ['cfg(frb_expand)'] }
```

Notes:

- Exact pin `=2.13.0` on the Rust side matches the Dart pin and avoids silent
  minor upgrades that break codegen.
- `[lints.rust] unexpected_cfgs` + `cfg(frb_expand)` is FRB-codegen hygiene —
  keep it when editing Cargo.toml unless FRB docs say otherwise for a bump.
- No `[features]` yet; add when optional crates (e.g. NLP, offline translate)
  need compile-time gates. User persistence is **not** a Cargo/SQLite feature —
  it is Drift in Flutter.
- Product compute pins (do not casually swap): `fasrt = "=0.5.0"` (subtitles),
  `rbook` `0.7` with `default-features = false` + `threadsafe`/`prelude`
  (**no** `write`; wasm-safe EPUB — see `work-with-parse` / `epub.md`).

### `flutter_rust_bridge.yaml` (app root)

```yaml
rust_input: crate::api
rust_root: rust/
dart_output: lib/src/rust
```

| Key | Meaning | Change carefully |
| --- | --- | --- |
| `rust_input` | Module path for public API (`crate::api` → `rust/src/api`) | Breaking for codegen; keep unless relocating the FFI surface |
| `rust_root` | Relative path to the Rust crate | Must stay the Cargo package root |
| `dart_output` | Generated Dart tree | Must stay under `lib/src/rust` (imports + AGENTS canon) |

Do **not** point `dart_output` at hand-written UI folders. Do **not** move
`rust_root` into `rust_builder/` (glue only).

### Dart / glue cross-check

| File | Relevant pin / name |
| --- | --- |
| `{app}/pubspec.yaml` | `flutter_rust_bridge: 2.13.0`; path dep `rust_lib_my_translator_app: path: rust_builder` |
| `{app}/rust_builder/pubspec.yaml` | `name: rust_lib_my_translator_app` (FFI plugin glue) |
| `{app}/rust/Cargo.toml` | `name = "rust_lib_my_translator_app"`; `flutter_rust_bridge = "=2.13.0"` |

Rename of the crate requires coordinated edits across these three — treat as
one config change, then rebuild.

## Version Pin Alignment (Dart ↔ Rust ↔ CLI)

All three must stay on **≈ 2.13.0**:

```text
pubspec.yaml          flutter_rust_bridge: 2.13.0
        ↕
rust/Cargo.toml       flutter_rust_bridge = "=2.13.0"
        ↕
flutter_rust_bridge_codegen   --version  ≈ 2.13.0
```

Rules:

1. Bump FRB only as an intentional trio change (Dart + Cargo + CLI), then
   regenerate (`flutter_rust_bridge_codegen generate`) and `cargo check` /
   `flutter analyze`.
2. Prefer exact / tight pins over floating `^` on the Rust crate when matching
   a known good scaffold.
3. CLI mismatch symptoms: codegen errors, missing symbols, or Dart/Rust type
   drift — fix pins first, not by hand-patching generated files.
4. After pin change: update `{app}/AGENTS.md` Core Stack if the documented
   version changes (docs live with the app; this skill states the current
   canon).

## Cargo: Dependencies And Features

### Adding a dependency (parse / lemma / translate)

1. Prefer crates that work on **Android / iOS / WASM** targets when the feature
   is product-facing; flag desktop-only crates explicitly.
2. Add to `rust/Cargo.toml` `[dependencies]` (or feature-gated — see below).
3. Put implementation in Rust modules **outside** thin `api/` (structure /
   domain skills); expose via `api/` only as needed → then FRB codegen
   (client `work-with-frb` / server `work-with-frb-api`).
4. Run from `{app}/rust/`: `cargo check` / `cargo test`.
5. Optional AI / cloud keys later are an explicit product decision — not
   default Cargo/FRB config.
6. **Do not** add `rusqlite` / `diesel` for user folders/documents/dictionary —
   that persist path is **Drift** in Flutter (`{app}/AGENTS.md`).

### Features (when to introduce)

Use `[features]` when a crate is optional, heavy, or platform-gated:

```toml
[features]
default = []
# example shape only — add when product needs compute crates:
# lemmatize = ["dep:…"]
# translate = ["dep:…"]
```

Do **not** feature-gate user SQLite in Rust; user store = Drift.

Guidelines:

- Keep `default` minimal so WASM / CI `cargo check` stays lean.
- Document new features in `{app}/AGENTS.md` if agents must pass
  `--features` for local work.
- Do not mirror Node `NODE_ENV` with Cargo features named `development` /
  `production` unless there is a real compile-time split.

### `[lib]` crate-types

Keep:

```toml
[lib]
crate-type = ["cdylib", "staticlib"]
```

- **`cdylib`** — dynamic library for FFI (Android/desktop-style loaders).
- **`staticlib`** — static link paths used by iOS / some FRB/cargokit flows.

Removing either breaks platform glue. Do not switch to `rlib`-only for the
app crate.

## Persistence / secrets

- User persistence: **Drift** in Flutter `lib/` (folders / documents /
  dictionary) — **not** `rusqlite` / `DATABASE_URL` / `rust/src/db`.
- Do not invent `.env.example` for core config. If a future change adds
  dart-define / optional AI keys, that is product + deploy skill territory —
  not Cargo/FRB yaml defaults.

## Patterns for Changing Config

### 1. Add or bump a Rust dependency

1. Edit `rust/Cargo.toml` (version + optional `features`).
2. `cargo check` / `cargo test` in `rust/`.
3. If the dep affects the public FFI types/signatures → update `rust/src/api/`
   (server `work-with-frb-api`) → codegen (client `work-with-frb`).
4. Leave `rust_builder/` alone unless cargokit/build glue breaks.

### 2. Change FRB YAML paths

1. Edit only if relocating the API module or generated Dart tree (rare).
2. Move/rename Rust `api` module and Dart imports in the **same** change.
3. Run `flutter_rust_bridge_codegen generate` from `{app}` root.
4. Update AGENTS / skills references if paths change.

### 3. Align or bump FRB 2.13.x

1. Set Dart `pubspec.yaml` + Rust `Cargo.toml` to the same version.
2. Install/use matching `flutter_rust_bridge_codegen`.
3. Regenerate; fix compile errors in `api/` or call sites — never in generated
   trees by hand.
4. Smoke: `cargo check`, `flutter analyze`, optionally `flutter test`.

### 4. Rename crate / path package

1. `rust/Cargo.toml` `[package].name`
2. `rust_builder/pubspec.yaml` `name`
3. App `pubspec.yaml` path dependency key
4. Rebuild; fix any leftover import/package strings

## Examples

### Keep FRB pin exact on Rust

```toml
[dependencies]
flutter_rust_bridge = "=2.13.0"
```

### Canonical FRB yaml (do not casually edit)

```yaml
rust_input: crate::api
rust_root: rust/
dart_output: lib/src/rust
```

### Do not add user-store SQLite in Rust

User library/dictionary persistence is Drift. Skip `rusqlite` / `diesel`
Cargo examples unless a future OpenSpec change explicitly requires a Rust
store (compute caches ≠ user CRUD).

## Common Mistakes

| Mistake | Why it hurts | Fix |
| --- | --- | --- |
| Dart FRB 2.13.0 but Cargo/`codegen` different | Codegen / link failures | Align all three ≈ 2.13.0 |
| Drop `staticlib` or `cdylib` | iOS / FFI glue breaks | Keep both crate-types |
| Hand-edit `lib/src/rust/**` after yaml change | Drift from codegen | Regenerate from `api/` |
| Put parse/NLP logic in `rust_builder/` | Wrong layer | `rust/` modules + thin `api/` |
| Change `dart_output` into `lib/features/…` | Mixes UI and generated bridge | Keep `lib/src/rust` |
| Rename crate in only one of Cargo / rust_builder / pubspec | Path package mismatch | Coordinated rename |
| Float Rust FRB with `^` while Dart is exact | Surprise upgrades | Prefer `=2.13.0` on Cargo |

## Checklist

When changing or reviewing config-related work:

- [ ] `rust/Cargo.toml` crate name still `rust_lib_my_translator_app`
- [ ] `[lib] crate-type` still `["cdylib", "staticlib"]`
- [ ] `flutter_rust_bridge` Rust pin `=2.13.0` (or intentionally bumped trio)
- [ ] App `pubspec.yaml` FRB Dart pin matches; CLI ≈ same
- [ ] `flutter_rust_bridge.yaml` keys still valid (`crate::api`, `rust/`, `lib/src/rust`)
- [ ] New deps justified for mobile/WASM; optional work behind features if heavy
- [ ] `rust_builder/` unchanged except true glue/name sync
- [ ] After FRB or API-affecting dep change: codegen + `cargo check` / analyze

## Agent Workflow

1. Confirm the task is Cargo / FRB yaml / version pins — not UI bind or API
   shape design.
2. Edit only the config files required (map below).
3. Verify from `{app}` / `rust/` with toolchain commands; fix failures before
   claiming done.

Typical commands (agent runs):

```bash
# from {app}/rust/
cargo check
cargo test

# from {app}/
flutter pub get
flutter_rust_bridge_codegen generate   # when yaml or FRB pin / api surface changed
flutter analyze
```

## Where Things Live

| Concern | Location |
| --- | --- |
| Rust package manifest | `{app}/rust/Cargo.toml` |
| Lockfile | `{app}/rust/Cargo.lock` |
| FRB codegen config | `{app}/flutter_rust_bridge.yaml` |
| Dart FRB pin + path dep | `{app}/pubspec.yaml` |
| FFI plugin glue name | `{app}/rust_builder/pubspec.yaml` |
| Public FFI source (not this skill’s design) | `{app}/rust/src/api/` |
| Generated Dart / Rust | `{app}/lib/src/rust/**`, `{app}/rust/src/frb_generated.rs` |
| Agent canon | `{app}/AGENTS.md` → Core Stack / Data And Utilities |
| Meta path | `docs/projects-map.md` → `app` |

## Related Skills

| Skill | Difference |
| --- | --- |
| client `work-with-frb` | `RustLib.init`, codegen workflow, Dart call sites, FFI vs WASM bind |
| server `work-with-frb-api` | Design of public Rust `api/` surface / types |
| client `work-with-env-deploy` | `flutter run` / build / release / devices |
| server `server-work-with-structure` | Where Rust modules live under `rust/src/` |
| client `work-with-frb` + structure | Flutter layout vs bridge — not Cargo pins |
