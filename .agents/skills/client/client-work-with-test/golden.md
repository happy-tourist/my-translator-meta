# Golden Tests (Optional)

Use with the [core test skill](SKILL.md) only when visual regression for a
**stable** widget is explicitly in scope (OpenSpec / user request). Goldens are
**optional** — default coverage is behavior via
[widget-tests.md](widget-tests.md), not screenshots.

## When to use

| Consider goldens | Skip goldens |
|------------------|--------------|
| Shared chrome that must not drift (token highlight paint, vocab tile) | One-off scaffold demo |
| Mock-driven Visual Spec already captured (`prepare-mock` / `verify-mock`) | Loading spinners / time-varying text |
| Cross-platform pixel intent agreed | Layout still in flux |

`verify-mock` (meta skill) compares implementation to a design mock; goldens
freeze a Flutter render. Prefer mock verify for “matches макет”; use goldens
for “does not regress this widget’s pixels”.

## Minimal pattern

```dart
testWidgets('vocab entry tile golden', (tester) async {
  await tester.pumpWidget(
    const MaterialApp(
      home: Scaffold(
        body: Center(child: VocabEntryTile(lemma: 'run', translation: 'бежать')),
      ),
    ),
  );
  await tester.pumpAndSettle();

  await expectLater(
    find.byType(VocabEntryTile),
    matchesGoldenFile('goldens/vocab_entry_tile.png'),
  );
});
```

- Store PNGs under `test/…/goldens/` (or project-agreed folder).
- Update with `flutter test --update-goldens` only when the visual change is
  intentional; include PNGs in the same change.
- Pin surface size (`tester.binding.setSurfaceSize`) when needed for stability.
- Prefer local widget goldens over full-screen app goldens.

## Rules

1. Do not golden the entire scaffold `MyHomePage` unless asked.
2. Do not replace behavior assertions with goldens alone — keep Key/text checks
   for interaction and error/loading branches.
3. Platform font/CI differences can flake — document known local-only goldens
   or use a consistent test font if the project adds one later.
4. Never hand-edit generated FRB Dart to stabilize pixels →
   [frb-boundary.md](frb-boundary.md).

## Checklist

- Explicitly in scope (not default for every UI change).
- Golden file committed with the intentional visual change.
- Behavior tests still cover interaction / async branches.
- Ran `flutter test` for the golden file path; flakes addressed.

## Anti-Patterns

```dart
// Golden-only coverage for a tap → FRB → result flow
await expectLater(find.byType(MyApp), matchesGoldenFile('home.png'));
// missing: enterText, tap, assert result / fake call

// Updating goldens to hide a layout bug without product intent
flutter test --update-goldens  // “to make CI green”
```
