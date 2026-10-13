# Widget Tests

Use with the [core test skill](SKILL.md) for tests under `{app}/test/` run by
`flutter test`. Prefer these for screens, widgets, and orchestration with
**faked** FRB boundaries → [frb-boundary.md](frb-boundary.md).

## When to use

| Use widget/unit tests | Prefer integration instead |
|-----------------------|----------------------------|
| Conditional UI, taps, text fields, loading/error/empty | Real `RustLib.init()` + generated API smoke |
| Fake orchestration / repository injection | Full app boot + platform channels you cannot fake cleanly |
| Pure Dart helpers (mappers, formatters) | Multi-route flows that need a real binding |

## File layout

```text
my_translator_app/test/
  features/reader/reader_screen_test.dart
  features/reader/video_frame_strip_test.dart
  features/reader/video_frame_page_math_test.dart
  features/vocab/vocab_entry_tile_test.dart
  widgets/empty_state_test.dart
```

Mirror `lib/` when it helps discovery. Name `*_test.dart`. One primary SUT per
file unless a tiny helper group shares the same fake.

## Binding and host

```dart
void main() {
  TestWidgetsFlutterBinding.ensureInitialized(); // usually implicit via flutter_test

  Future<void> pumpSut(WidgetTester tester) async {
    await tester.pumpWidget(
      MaterialApp(
        home: ReaderScreen(
          loadCues: fake.loadCues,
          onAddToVocab: fake.addToVocab,
        ),
      ),
    );
  }
}
```

- Wrap screens in `MaterialApp` (or `WidgetsApp`) so `Theme` / `MediaQuery` /
  `Directionality` exist.
- For leaf widgets that only need directionality, a smaller host is fine.
- Do **not** call `RustLib.init()` here unless the widget under test cannot be
  decoupled yet — then document the debt and prefer extracting an injectable
  call site → [frb-boundary.md](frb-boundary.md).

## Driving the UI

| Action | API |
|--------|-----|
| Tap | `tester.tap(find.byKey(…))` then `pump` / `pumpAndSettle` |
| Text | `tester.enterText(find.byKey(…), '…')` |
| Scroll | `tester.drag` / `fling` on the scrollable finder |
| Frame | `pump()` one frame; `pump(Duration)` for timers; `pumpAndSettle()` when idle is expected |

After every user action that schedules async work, pump until the assertion
surface is stable. If `pumpAndSettle` loops forever, an animation/timer is
still running — pump fixed frames or fake the ticker source.

## Loading / error / empty

Assert what the user sees:

```dart
testWidgets('shows error text when load fails', (tester) async {
  fake.throwError = Exception('parse failed');
  await pumpSut(tester);
  await tester.pumpAndSettle();

  expect(find.byKey(const Key('reader-error')), findsOneWidget);
  expect(find.byKey(const Key('cue-list')), findsNothing);
});

testWidgets('clears loading after success', (tester) async {
  await pumpSut(tester);
  expect(find.byKey(const Key('reader-loading')), findsOneWidget);
  await tester.pumpAndSettle();
  expect(find.byKey(const Key('reader-loading')), findsNothing);
  expect(find.byKey(const Key('cue-list')), findsOneWidget);
});
```

Add `Key`s on loading / error / empty / list hosts in the SUT when asserting.
Do not invent a parallel toast pipeline — match whatever the screen actually
shows (`Text`, banner, `SnackBar` if production code uses it).

## Pure Dart helpers

Call directly — no `pumpWidget`:

```dart
test('maps lemma highlight spans', () {
  expect(mapHighlight('running', lemma: 'run'), isNotEmpty);
});
```

Keep parse / lemma / dictionary **rules** out of Dart; if the helper only
formats UI models from Rust DTOs, testing the mapper is fine.

## Checklist

- File under `test/`, name `*_test.dart`.
- Pump helper with no scenario args; fake fields configure scenarios.
- Keys for interactive and conditional surfaces.
- Success + error (+ loading reset) when the SUT branches on async.
- No `RustLib.init()` / generated API unless intentionally coupled.
- Ran `flutter test` (or a focused path) from the app root.

## Anti-Patterns

```dart
// Integration binding inside a unit/widget file without need
IntegrationTestWidgetsFlutterBinding.ensureInitialized();

// Asserting private State fields
expect(tester.state<_ReaderScreenState>(…)._loading, isTrue);

// Skipping pump after tap
await tester.tap(find.byKey(const Key('call-rust')));
expect(find.text('…'), findsOneWidget); // may still be pending

// Testing static AppBar title that never branches
expect(find.text('Flutter + Rust Demo'), findsOneWidget);
```
