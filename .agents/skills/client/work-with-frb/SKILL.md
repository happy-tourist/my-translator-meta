---
name: work-with-frb
description: >-
  Use when working with flutter_rust_bridge in my_translator_app: RustLib.init,
  FRB codegen (`flutter_rust_bridge_codegen generate`), generated API under
  lib/src/rust, public Rust surface in rust/src/api, frb_generated.rs,
  FFI vs WASM, rust_builder glue, or binding Flutter UI to Rust async / streams
  after an API change.
---

# Work With FRB (flutter_rust_bridge)

How agents change and call the **generated** Flutter ↔ Rust bridge in
**my_translator_app** (Translation Reader). Canon: `{app}/AGENTS.md`.

**Version:** `flutter_rust_bridge` **2.13.0** (pin in `pubspec.yaml` and
`rust/Cargo.toml`). CLI for codegen should match ≈ `2.13.0`.

UI folder layout → **`client-work-with-structure`**. Rust core modules outside
`api/` → app Rust skills / AGENTS. This skill owns the **bridge contract** only.

## When to Use

- Add/rename/remove a public FFI function or type
- Call generated Dart after `RustLib.init`
- Run or reason about codegen
- Fix “library not loaded” / wrong platform (FFI vs WASM)
- Wire UI to Rust `Future` / stream (loading, first event, dispose)

**Not this skill:** inventing product UX layout; putting parse/lemma/dict logic
in Dart; editing cargokit for business rules.

## Layout (truth vs generated)

| Role | Path | Edit? |
|------|------|-------|
| Public FFI API (source of truth) | `rust/src/api/**/*.rs` | **Yes** |
| Generated Rust glue | `rust/src/frb_generated.rs` | **Never** |
| Generated Dart API | `lib/src/rust/**` | **Never** |
| FRB config | `flutter_rust_bridge.yaml` | Rarely |
| App entry | `lib/main.dart` | Init + `runApp` only |
| Platform glue | `rust_builder/` | Glue/cargokit only — **no** business logic |

Config today:

```yaml
rust_input: crate::api
rust_root: rust/
dart_output: lib/src/rust
```

Import generated API as
`package:my_translator_app/src/rust/api/<module>.dart` and
`package:my_translator_app/src/rust/frb_generated.dart`.

## Init Before UI

```dart
Future<void> main() async {
  await RustLib.init(); // runs Rust #[frb(init)] init_app
  runApp(const MyApp());
}
```

- **Always** `await RustLib.init()` **before** `runApp` (and before any FRB call).
- Scaffold: `init_app` → `flutter_rust_bridge::setup_default_user_utils()`.
- Tests without native lib: `RustLib.initMock(api: …)` only when you supply a mock API — not a substitute for real init in the app.

## Change API → Codegen → Call Sites

```text
1. Edit rust/src/api/… (+ mod in api/mod.rs)
2. Keep heavy logic in rust modules; api/ stays a thin export
3. From app root: flutter_rust_bridge_codegen generate
4. Update Dart call sites / tests to new generated names
5. flutter analyze / flutter test; cargo check|test in rust/
```

Naming: Rust `snake_case` → Dart `camelCase` with named params
(`process_text` → `processText(input:)`). Prefer `async` Rust / Dart `Future`
for work that must not block the UI isolate.

**Do not** hand-patch `lib/src/rust/**` or `frb_generated.rs` “just this once”
— regenerate. If codegen and sources disagree, fix `api/` or re-run codegen.

## FFI vs WASM

| Target | Mechanism | Notes |
|--------|-----------|--------|
| Android / iOS / desktop | FFI via `rust_builder` + cargokit | Native lib built with the Flutter build |
| Web (`flutter run -d chrome`) | WASM | Same Dart API; `frb_generated.io.dart` vs `.web.dart` via conditional import |

Agents do not maintain two hand-written Dart surfaces — FRB emits both.
Touch `rust_builder/` only for build/glue breakage, never for parse/NLP/dict.

## Async / Stream Pitfalls (UI binding)

| Risk | Do |
|------|----|
| Call FRB before `RustLib.init` | Init in `main` first |
| Await gap: UI paints empty before first result | Loading/empty state until first successful bind |
| Sync heavy work on UI isolate | Expose async from Rust; await in orchestration |
| Stream: miss first event | Subscribe **before** starting work that emits; handle replay/seed if API design needs it |
| `setState` / notify after dispose | Guard with `mounted` (or cancel subscription in `dispose`) |
| Scatter FRB in leaf widgets | Thin orchestration / screen method (see structure skill) |
| Stale integration test vs new API | Update `integration_test/` / widget tests in the same change |

Generated sync-looking Rust may still surface as `Future` in Dart (scaffold:
`processText` / `addNumbers`). Always `await` and treat as async for UI.

## Quick Reference

| Task | Action |
|------|--------|
| New FFI fn | `rust/src/api/` → codegen → call generated Dart |
| Rename API | Same; update all Dart imports/callers/tests |
| Init | `await RustLib.init()` then `runApp` |
| Codegen | `flutter_rust_bridge_codegen generate` (CLI ≈ 2.13.0) |
| Business logic | Rust modules; thin `api/` only |
| Glue only | `rust_builder/` |

Scaffold contract (current): `processText` / `addNumbers` / `RustLib.init` ↔
`process_text` / `add_numbers` / `init_app`. Product APIs (parse, lemma, dict,
translate) land the same way — not by editing generated files.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Hand-edit `lib/src/rust/**` or `rust/src/frb_generated.rs` | Change `rust/src/api/` → codegen |
| `runApp` before `RustLib.init` | Await init in `main` |
| Business logic in `rust_builder/` | Move to `rust/`; rebuild |
| Dart reimplements parse/lemma/dict | Rust + thin FRB export |
| Skip codegen after `api/` change | Regenerate; then fix call sites |
| CLI / crate version drift from 2.13.0 | Align pubspec, Cargo.toml, and codegen CLI |
| Ignore first stream event / no loading | Bind subscription + UI state before/with start |
| Grow demo `MyHomePage` instead of feature call sites | New screens + orchestration; replace scaffold |

## Not This Skill

| Skill | Difference |
|-------|------------|
| `client-work-with-structure` | Where screens/widgets/orchestration live under `lib/` |
| `align-implementability-front` | Whether meta UI claims are buildable (incl. FRB shapes) |
| `align-code` | Audit races / compliance after implementation |
