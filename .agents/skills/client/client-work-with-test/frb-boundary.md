# FRB Boundary Testing

Use with the [core test skill](SKILL.md) when tests touch
**flutter_rust_bridge** (`RustLib`, `package:my_translator_app/src/rust/…`).

## Hard rules

1. **Never hand-edit** generated Dart: `lib/src/rust/**` (including
   `frb_generated.dart` / `.io` / `.web`).
2. **Never hand-edit** `rust/src/frb_generated.rs`.
3. Change the public contract only in `rust/src/api/**/*.rs`, then run
   `flutter_rust_bridge_codegen generate`, then update UI **and** tests.
4. Prefer testing **UI with fakes** at an injectable boundary. Do not patch
   generated methods in place for unit tests.

## Preferred seam (widget tests)

Keep FRB I/O in a thin orchestration layer (screen method, controller, or
small repository) as in `client-work-with-structure`. Widgets under test take
callbacks or an interface.

**Product Reader / Vocab seams (prefer these over demo `processText`):**

| Typedef | File | Default |
|---------|------|---------|
| `TokenizeCuesFn` | `lib/features/reader/tokenize_cues_fn.dart` | FRB `tokenizeCues` |
| `MarkVocabTokensFn` | same | FRB `markVocabTokens` |
| `LemmaKeysForSurfacesFn` | `lib/data/dictionary/lemma_keys_for_surfaces_fn.dart` | FRB `lemmaKeysForSurfaces` |
| `ParseSubtitlesFn` | `lib/data/library/parse_subtitles_fn.dart` | FRB `parseSubtitles` |

Wire via `ReaderScreen` / `LibraryScreen` / `AppShell` / `VocabularyScreen`
optional ctor params. In tests, inject fakes that return fixed
`CueTokensDto` / lemma keys — do **not** reimplement WordNet or panko in Dart.

```dart
typedef ProcessTextFn = Future<String> Function({required String input});

// Demo-only pattern (scaffold). Prefer the product typedefs above for Reader.
class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key, required this.processText});
  final ProcessTextFn processText;
}
```

Assert:

- Fake received expected args (texts / vocabLemmas / surfaces).
- UI shows success text / error surface / loading cleared / highlight paint.
- Do **not** assert Rust morph / span semantics here — that is `cargo test`
  (or a thin integration smoke).

## When the scaffold still calls generated API directly

Current demo `MyHomePage` may import
`package:my_translator_app/src/rust/api/simple.dart` inside the widget. Options,
in order:

1. **Extract** an injectable function/controller in the same product change
   (preferred; matches structure skill).
2. Cover the screen with an **integration** test that calls real
   `RustLib.init()` → [integration-tests.md](integration-tests.md).
3. Avoid brittle reflection / generated-file edits.

Do not introduce a global mock framework solely to stub generated top-level
functions unless AGENTS / OpenSpec adopts one.

## Integration / bridge smoke

Real boundary:

```dart
setUpAll(() async => await RustLib.init());
// pump MyApp, drive UI, assert visible result from real processText / addNumbers
```

Use sparingly: proves FFI/WASM wiring and codegen alignment. Keep algorithmic
coverage in Rust tests.

## What belongs where

| Concern | Test layer |
|---------|------------|
| Button enables, error banner, list empty | Widget + fake |
| DTO → view-model mapping in Dart | Unit (no pump) or widget |
| Parse / lemma / dictionary rules | `cargo test` in `rust/` |
| `RustLib.init` + one happy path UI | `integration_test/` |
| Regenerated signatures after API change | Update call sites + tests in same change |

## Checklist

- No edits under `lib/src/rust/**` for test convenience.
- Widget tests inject fakes; integration tests may use real bridge.
- API change ⇒ codegen ⇒ Dart call sites ⇒ tests updated together.
- Success and error UI paths covered when the SUT branches.

## Anti-Patterns

```dart
// Editing generated API to add test hooks
// lib/src/rust/api/simple.dart  ← FORBIDDEN

// Reimplementing lemma/parse in a Dart fake “for realism”
class FakeParser { List<Cue> parseSrt(String s) => /* Dart parser */ }

// Widget test that requires native/WASM without documenting why
await RustLib.init(); // inside every reader_screen_test.dart
```
