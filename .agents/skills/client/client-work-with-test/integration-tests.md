# Integration Tests

Use with the [core test skill](SKILL.md) for tests under
`{app}/integration_test/` run by `flutter test integration_test`. These exercise
a fuller app binding and, when intentional, the real FRB bridge.

## When to use

| Use integration tests | Prefer widget tests instead |
|-----------------------|-----------------------------|
| Smoke: `RustLib.init()` + home flow with real generated API | Conditional UI with fakes |
| Multi-widget flows that need real plugins/bindings | Fast iteration on one screen |
| Regression that a widget fake cannot express | Pure Dart helpers |

Keep the suite **small**. Most product UI scenarios belong in
[widget-tests.md](widget-tests.md) with fakes → [frb-boundary.md](frb-boundary.md).

## Binding

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:my_translator_app/main.dart';
import 'package:my_translator_app/src/rust/frb_generated.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  setUpAll(() async {
    await RustLib.init();
  });

  testWidgets('home can call rust and show result', (tester) async {
    await tester.pumpWidget(const MyApp());
    await tester.pumpAndSettle();

    await tester.enterText(find.byKey(const Key('input')), 'hello');
    await tester.tap(find.byKey(const Key('call-rust')));
    await tester.pumpAndSettle();

    expect(find.textContaining('hello'.toUpperCase()), findsOneWidget);
  });
}
```

- Call `IntegrationTestWidgetsFlutterBinding.ensureInitialized()` once at the
  top of `main`.
- Init FRB in `setUpAll` (or once before first pump) when the test uses
  generated APIs — same rule as production `main()` (`await RustLib.init()`
  before UI that calls Rust).
- Optional driver entry: `test_driver/integration_test.dart` →
  `integrationDriver()` for older device-lab flows; day-to-day agents use
  `flutter test integration_test`.

## Scaffold debt (read `{app}/AGENTS.md`)

`integration_test/simple_test.dart` currently expects copy like
`Result: Hello, Tom!`, which **does not match** the current `lib/main.dart`
UI (`Результат появится здесь` / `Rust вернул: …`).

When this skill applies to home/demo or integration work:

1. Update or replace `simple_test.dart` in the **same change** so assertions
   match real labels and interactions (type text → tap «Вызвать Rust» → assert
   result), **or**
2. Delete/skip it only if an OpenSpec task explicitly retires that smoke.

Do not leave a knowingly stale expectation as “documentation”.

## What to assert

- End-to-end user-visible outcomes after real (or test-env) bridge calls.
- Keys / stable text added for the flow under test.
- Success path at minimum for smoke; add error path when the product surfaces
  bridge failures in UI.

## What not to assert

- Pixel-perfect layout (use [golden.md](golden.md) only when chosen).
- Internal Rust algorithm correctness — that belongs in `cargo test` under
  `rust/`.
- Duplicate coverage of every conditional already locked by widget tests.

## Checklist

- File under `integration_test/`, `*_test.dart`.
- Binding initialized; `RustLib.init()` when calling generated API.
- Assertions match **current** `main.dart` / feature screens.
- Prefer one smoke per critical flow, not a second widget-test suite.
- Ran `flutter test integration_test` from the app root; failures fixed.

## Anti-Patterns

```dart
// Stale copy from an older scaffold
expect(find.textContaining('Result: `Hello, Tom!`'), findsOneWidget);

// Using integration tests for every Key toggle
// → move to test/ with fakes

// Hand-editing lib/src/rust/** so integration passes
```
