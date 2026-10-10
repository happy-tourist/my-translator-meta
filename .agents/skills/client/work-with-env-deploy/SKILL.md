---
name: work-with-env-deploy
description: >-
  Use when running or releasing my_translator_app (Translation Reader):
  flutter pub get, flutter devices / flutter run, flutter build apk /
  appbundle / ipa / web, FRB codegen before a dirty API, WASM/web notes
  (COOP/COEP + Drift Web assets), rust_builder / cargokit glue, or release
  verification — Flutter + Rust/FRB + Drift toolchain.
---

# Work With Env And Deploy

## Overview

Use this skill for **local run**, **release builds**, and **FRB/WASM glue** of
**my_translator_app** (product: **Translation Reader**).

Stack (from `{app}/AGENTS.md`): Flutter UI (`lib/`), Rust core (`rust/`),
**flutter_rust_bridge** 2.13.0 (FFI native, WASM web), glue in
`rust_builder/` (cargokit). Deploy through Flutter build targets.

Product targets: **Android, iOS, Web**. Other Flutter runners (linux /
macos / windows) exist as scaffold — do not treat as product targets unless
the user asks.

Working directory for all commands: **`{app}` root**
(`../my_translator_app` from meta per `docs/projects-map.md`).

## Hard rules

| Do | Don't |
|----|--------|
| Run Flutter / Cargo / FRB CLI from `{app}` root (Cargo in `rust/`) | Use a non-Flutter toolchain for this app |
| Prefer product builds: `apk` / `appbundle` / `ipa` / `web` | Treat desktop runners as required product gates |
| After public Rust API change: FRB codegen, then build/run | Hand-edit `lib/src/rust/**` or `rust/src/frb_generated.rs` |
| Leave `rust_builder/` as glue/cargokit only | Put business logic in `rust_builder/` |
| Verify with analyze / test / build before claiming done | Skip verification or assume pass without running |

## Env meaning

“Env” here means **toolchain / device / build flavor** (Flutter SDK, target
device, `dart-define` / flavors when added). Optional AI providers (later) are
an explicit product decision — not this skill’s default deploy path.

If a future OpenSpec change adds dart-define / flavors / secrets, document them
in `{app}/AGENTS.md` and wire them into `flutter run` / `flutter build`.

## Prerequisites (agent)

From `{app}`:

```bash
flutter pub get
```

When Rust public API under `rust/src/api/` changed:

```bash
flutter_rust_bridge_codegen generate   # CLI ≈ 2.13.0; config: flutter_rust_bridge.yaml
```

Optional Rust sanity (from `rust/`):

```bash
cargo check
cargo test
```

## Devices and run

```bash
flutter devices
flutter run                    # default connected device
flutter run -d chrome          # Web — prefer COOP/COEP flags below
flutter run -d <deviceId>      # Android / iOS device or emulator
```

**Web (FRB WASM + Drift):** without COOP/COEP, `RustLib.init` fails
(`SharedArrayBuffer` / `crossOriginIsolated`). From app root:

```bash
flutter run -d chrome \
  --web-header=Cross-Origin-Opener-Policy=same-origin \
  --web-header=Cross-Origin-Embedder-Policy=require-corp
```

[`implement-change`](../../implement-change/SKILL.md) **обязательно** гоняет этот
командный шаблон как финальный **web-smoke после commit**, и mid-apply/verify —
когда нужна проверка Web. Не использовать голый `flutter run -d chrome` в этих
фазах.

Drift Web assets (required beside FRB WASM): `web/sqlite3.wasm`,
`web/drift_worker.js` — see `work-with-drift` / `{app}/AGENTS.md`.

Startup contract: `main()` must `await RustLib.init()` before
`TranslationReaderApp` (which then opens Drift). If init fails, fix
FRB/WASM/native glue — do not bypass init for “UI-only” smoke.

## Release builds (product targets)

| Target | Command | Typical output |
|--------|---------|----------------|
| Android APK | `flutter build apk` | `build/app/outputs/flutter-apk/` |
| Android Play | `flutter build appbundle` | `build/app/outputs/bundle/` |
| iOS | `flutter build ipa` | requires macOS + Xcode signing |
| Web | `flutter build web` | `build/web/` (includes WASM via FRB) |

Notes:

- **iOS**: needs Apple toolchain and signing; on non-macOS hosts, do not claim
  `ipa` success — report the platform limit.
- **Web**: FRB **WASM** (`lib/src/rust/frb_generated.web.dart`) plus Drift
  (`web/sqlite3.wasm`, `web/drift_worker.js`). Prefer `flutter build web` /
  COOP/COEP `flutter run -d chrome` over hand-editing generated wasm packs.
- **Android**: first build may compile Rust via **cargokit** under
  `rust_builder/` — slow is normal; failures often point at NDK/Rust toolchain,
  not Dart UI.
- Desktop runners (`linux` / `macos` / `windows`) are scaffold-only unless
  explicitly requested.

## FRB / WASM / rust_builder

| Piece | Role |
|-------|------|
| `flutter_rust_bridge.yaml` | `rust_input` / `rust_root` / `dart_output` |
| `rust/src/api/` | Public FFI surface (source of truth) |
| `lib/src/rust/` | Generated Dart — do not hand-edit |
| `rust/src/frb_generated.rs` | Generated Rust — do not hand-edit |
| `rust_builder/` | Path package `rust_lib_my_translator_app`; cargokit build glue |
| `rust_builder/cargokit/` | Native/WASM build tooling — leave alone unless fixing glue |
| Web | WASM bindgen path; run/build through Flutter |

Native: FFI. Web: WASM. Same Dart call sites after codegen.

## Files map

| Concern | Path |
|---------|------|
| Agent / tool canon | `{app}/AGENTS.md` → Development Tools |
| Dart deps | `{app}/pubspec.yaml` |
| FRB config | `{app}/flutter_rust_bridge.yaml` |
| UI entry | `{app}/lib/main.dart` |
| Generated bridge | `{app}/lib/src/rust/` |
| Rust crate | `{app}/rust/` |
| Build glue | `{app}/rust_builder/` (+ `cargokit/`) |
| Web host | `{app}/web/` |
| Platform runners | `{app}/android/` `ios/` `web/` (product); others scaffold |
| Meta path | `docs/projects-map.md` → key `app` |

## Agent workflow

1. Confirm task: deps, device run, platform build, or FRB/WASM glue failure.
2. `cd` to `{app}`; run `flutter pub get` if deps or lockfile may be stale.
3. If `rust/src/api/` changed → FRB codegen before Flutter build/run.
4. Pick the product command (`run` vs `build apk|appbundle|ipa|web`).
5. On failure: read the first real toolchain error (Rust/cargokit/NDK/Xcode/WASM),
   fix glue or API — do not “fix” by deleting `RustLib.init()` or editing generated files.
6. Re-run verification; only then claim done.

### Typical verification (agent runs)

```bash
flutter pub get
flutter analyze
flutter test
flutter build apk          # or appbundle / web / ipa as relevant
```

Web smoke: `flutter run -d chrome` or `flutter build web`.
Rust-only: `cargo test` in `rust/`.

## Anti-patterns

- Hand-editing generated FRB Dart/Rust instead of codegen.
- Putting parse / NLP / user-Drift / translate logic into `rust_builder/`.
- Treating linux/macos/windows desktop builds as required product gate.
- Claiming iOS `ipa` success without a macOS+signing environment.
- Skipping `flutter pub get` / analyze / build after glue or API changes.
- Committing secrets, `build/`, `rust/target/`, or `.dart_tool/`.

## Related

- App commands canon → `{app}/AGENTS.md` (Development Tools)
- UI placement → `client-work-with-structure`
- Paths → `docs/projects-map.md` (`app`)
