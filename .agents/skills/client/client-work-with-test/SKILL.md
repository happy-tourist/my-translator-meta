---
name: client-work-with-test
description: >-
  Use when planning or writing Flutter tests for my_translator_app (Translation
  Reader): widget tests (`flutter test`), integration tests
  (`flutter test integration_test`), FRB boundary fakes (prefer UI + fake
  orchestration; never hand-edit generated `lib/src/rust/**`), loading/error/
  empty UX, Key-based finders, optional goldens. Harness: `flutter_test`,
  `integration_test`, `{app}/test/`, `{app}/integration_test/`. Core in
  SKILL.md; details in topics: widget-tests.md, integration-tests.md,
  frb-boundary.md, golden.md.
---

# Work With Test

Use this skill to plan and write Flutter tests for **my_translator_app**
(product: **Translation Reader**). Start with a test plan, then implement
tests from that plan.

**Paths:** this skill lives in **my_translator_meta** at
`.agents/skills/client/client-work-with-test/`. Runtime paths (`lib/`, `test/`,
`integration_test/`) are relative to the **my_translator_app** sibling
(`../my_translator_app` from meta; see `docs/projects-map.md`). Rust unit
tests stay in `rust/` via `cargo test` — do **not** mix stacks or invent
Node / browser runners (Jest, Cypress, Playwright, etc.) for this package.

Stack: **Flutter** (`sdk: ^3.13.4`), **`flutter_test`**, **`integration_test`**,
**flutter_rust_bridge** `2.13.0` (generated Dart under `lib/src/rust/` — never
hand-edit). No backend; UI ↔ core is local FRB. State-management package is
not mandated until `{app}/AGENTS.md` adopts one.

Core principle: test user-visible behavior and screen/widget contracts, not
implementation details. Do not test unconditional rendering, static Theme
attrs, or direct constructor pass-through unless there is conditional logic,
transformation, user-visible behavior, or a callback depending on it.

## Specialized Topics

Read the matching file in this folder when the unit under test involves that
area (do not load every file at once):

| Topic | File |
|-------|------|
| Widget / unit tests under `test/` | [widget-tests.md](widget-tests.md) |
| Device/integration tests under `integration_test/` | [integration-tests.md](integration-tests.md) |
| FRB / RustLib boundary fakes and what not to edit | [frb-boundary.md](frb-boundary.md) |
| Optional golden / screenshot tests | [golden.md](golden.md) |

## Bootstrap (harness already in app)

Test harness lives in **my_translator_app**:

- Dev deps: `flutter_test` (SDK), `integration_test` (SDK), `flutter_lints`
- Scripts / commands (from app root):
  - Unit/widget: `flutter test`
  - Integration: `flutter test integration_test`
  - Analyze: `flutter analyze`
- Layout: `test/` (widget/unit), `integration_test/` (full-app / device),
  optional `test_driver/` for older driver entrypoints
- Scaffold note (`{app}/AGENTS.md`): `integration_test/simple_test.dart` is
  **out of sync** with current `lib/main.dart` (expects old
  `Result: Hello, Tom!` copy). Fix or rewrite that test when touching
  integration coverage for the home/demo UI — do not treat a red stale test
  as proof the product is broken.

Do **not** re-bootstrap with Node or web-only runners — use `flutter test`
and `flutter test integration_test`. Do **not** add mockito/mocktail unless
an OpenSpec change / AGENTS adopts them; prefer hand-written fakes and
constructor injection → [frb-boundary.md](frb-boundary.md).

Agent **runs** `flutter test` (and `flutter test integration_test` when
integration files changed) from the **app** root after writing or changing
tests; fix failures before claiming done. Also run `flutter analyze` when the
change touches production Dart. Rust logic: `cargo test` in `rust/` — separate
skill/stack.

OpenSpec: new Flutter UI / orchestration scenarios **must** include
`flutter test` (and integration when the scenario needs a real binding/bridge)
task items mapped to `SC-…` IDs in Traceability — not `flutter analyze` alone.

## Workflow

1. Read the screen / widget / orchestration under test: constructor params,
   callbacks, FRB call sites, conditional UI, async loading/error/empty.
2. Produce a test plan with two sections: **What needs to be faked/mocked**
   and **What to verify**. Pure Dart helpers: call directly (no fake).
3. Place tests:
   - Widget/unit → `test/…` mirroring `lib/` when helpful
     (`test/features/reader/reader_screen_test.dart`).
   - Integration → `integration_test/<flow>_test.dart`.
4. Write tests following the conventions below and specialized topics.
5. Cover planned success, error, loading, interaction, and conditional
   scenarios.
6. **Required params check (widgets):** after writing the file, re-read the
   widget constructor and confirm every required positional/named param
   without a default is supplied in the pump helper.

## Test Plan

### What Needs To Be Faked / Mocked

List every external dependency that must be controlled:

- **FRB / generated API** — do not edit `lib/src/rust/**`. Prefer injecting a
  thin orchestration/repository interface (or callbacks) into the widget under
  test and use a fake → [frb-boundary.md](frb-boundary.md).
- **File pickers / platform channels** — fake at the call site the screen owns.
- **Timers / animation** — `tester.pump` / `pumpAndSettle` / fake async as
  needed → [widget-tests.md](widget-tests.md).
- Pure Dart SUT: do **not** fake it; call it directly.

Do **not** call real `RustLib.init()` inside lightweight widget tests unless
the test is intentionally an integration/bridge smoke →
[integration-tests.md](integration-tests.md), [frb-boundary.md](frb-boundary.md).

### What To Verify

Use these categories only when the SUT has relevant behavior:

- Conditional UI: via `testWidgets` table/`for` loops; widget by `Key` /
  `find.byKey` exists or not.
- Text / labels: visible copy by condition (scaffold may be RU/EN — assert
  what the SUT actually shows).
- Child constructor args: only when conditional or transformed — never static
  pass-through.
- Orchestration / FRB-facing calls: when called, with what args, success vs
  error → [frb-boundary.md](frb-boundary.md).
- Buttons / enabled state: conditional `onPressed == null` / busy affordance
  by `Key`.
- Callbacks: verify fake/spy was invoked (not private `State` fields).
- Errors: error surface visible on failure; success path clears / does not
  leave stale error; loading cleared in `finally`-equivalent.
- Loading: busy flag/spinner shown then cleared after both resolve and reject.

Formulation rules:

- Use “via each” for multiple scenarios.
- Always specify a `Key` (or stable semantics) for interactive HTML-equivalent
  targets; prefer `Key('…')` / `ValueKey` added in the SUT when asserting.
- Always name the condition under test.
- For every FRB-facing call the SUT owns, cover success and error when the UI
  branches.
- Do not plan assertions for static, unconditional decoration.

## Do Not Test Static Child Props / Attributes

**Do not assert constructor args or style on children when those values are
fixed in `build` and do not depend on state, props, computed logic, or user
actions.**

Typical unnecessary tests:

- Hardcoded `EdgeInsets`, `Colors.blue`, `AppBar` title that never changes by
  scenario.
- Static icon / label pass-through.
- `expect(find.byType(Padding), …)` used only to assert layout chrome.

Write child assertions only when the value **changes by condition** —
loading, disabled while busy, transformed result text, shown/hidden branch.

```dart
// Bad: static theme chrome never changes by scenario
expect(tester.widget<AppBar>(find.byType(AppBar)).backgroundColor, Colors.blue);

// OK: result text depends on fake FRB success
expect(find.textContaining('Rust вернул'), findsOneWidget);
```

Examples:

```text
Verify orchestration.processText call on tap Key('call-rust') with field text

Verify via each success and error scenarios:
- On success: result text updated; error banner absent; loading idle
- On error: error surface visible; success text NOT shown; loading idle

Verify via each enabled/busy of button by Key('call-rust'):
- disabled / non-interactive while loading
- enabled when input non-empty and not loading
```

## Test File Structure

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:my_translator_app/features/demo/home_screen.dart';

class _FakeTranslator {
  String? lastInput;
  Object? throwError;
  String result = 'OK';

  Future<String> processText({required String input}) async {
    lastInput = input;
    if (throwError != null) throw throwError!;
    return result;
  }
}

void main() {
  late _FakeTranslator fake;

  setUp(() {
    fake = _FakeTranslator();
  });

  Future<void> pumpHome(WidgetTester tester) async {
    await tester.pumpWidget(
      MaterialApp(
        home: HomeScreen(processText: fake.processText),
      ),
    );
  }

  testWidgets('calls processText with field text on tap', (tester) async {
    await pumpHome(tester);
    await tester.enterText(find.byKey(const Key('input')), 'hello');
    await tester.tap(find.byKey(const Key('call-rust')));
    await tester.pumpAndSettle();

    expect(fake.lastInput, 'hello');
    expect(find.textContaining('OK'), findsOneWidget);
  });
}
```

Adapt names to the real SUT; while scaffold still uses `MyHomePage` + direct
`processText` imports, either extract an injectable call site for the test or
cover behavior via integration with a real bridge →
[frb-boundary.md](frb-boundary.md), [integration-tests.md](integration-tests.md).

## Pumping Widgets

- Prefer `pumpWidget` + `MaterialApp`/`Directionality` host for screens.
- Always create a **pump helper** with **no scenario parameters**. Never call
  `pumpWidget` with ad-hoc trees duplicated in every test. Put host options
  inside the helper; override via mutable fake / `tester` interactions.
- After actions and async work: `await tester.pump()` /
  `await tester.pumpAndSettle()` as appropriate → [widget-tests.md](widget-tests.md).
- Prefer `test/` colocated by feature. Integration files stay under
  `integration_test/`.
- Do not reach into `State` private fields; drive UI and assert finders /
  fake call logs.

## Required Constructor Params

Required params belong in the pump helper’s widget construction; scenario
overrides use the fake’s fields or a second helper only when the widget tree
shape must differ.

Before finishing any widget test file:

1. List every constructor param without a default.
2. Confirm each appears in the pump helper.
3. Use no-op / fake callbacks for required callbacks; smallest valid value for
   data params.

## Finding Elements

Priority order:

1. `Key` / `find.byKey` for interactive or conditional targets (add in SUT
   when asserting behavior).
2. `find.text` / `find.textContaining` for user-visible copy under test.
3. `find.byType` when a single instance is unambiguous.
4. Do not find by incidental layout/`Padding` nesting or palette classes.

When several instances of the same type exist, give each a `Key`; do not rely
on `find.byType(X).at(index)` for behavior assertions.

## Callback / Event Assertions

- Verify via fake/spy call logs or resulting UI, not private `State` methods.
- Prefer `expect(fake.lastInput, …)` + visible result over
  `expect(tester.state<…>, …)`.

## Checklist Before Finishing

- Harness exists — `flutter test` runs from app root.
- Required constructor-param audit passed for widgets.
- All trees via pump helper; `setUp` resets fakes.
- Prefer widget tests with FRB fakes; real `RustLib.init()` only in
  integration / intentional bridge smoke → [frb-boundary.md](frb-boundary.md).
- Never hand-edit `lib/src/rust/**` or `rust/src/frb_generated.rs` for tests.
- `pump` / `pumpAndSettle` after actions and async.
- Interactive targets by `Key`; similar scenarios use table/`for` loops.
- Success and error paths for every FRB-facing call the SUT owns; loading
  resets after both.
- No static unconditional decoration assertions.
- If touching integration for home/demo: reconcile
  `integration_test/simple_test.dart` with current `main.dart` copy.
- Ran `flutter test` from the app root; failures fixed. Ran integration
  suite when those files changed.

## Anti-Patterns

```dart
// Pump helper taking scenario props from each test
await pumpHome(tester, result: 'x');

// Required callback only set after first pump via hacks
await tester.pumpWidget(MaterialApp(home: HomeScreen()));

// Duplicated pumpWidget trees instead of one helper
await tester.pumpWidget(MaterialApp(home: HomeScreen(processText: fake.processText)));

// Hand-editing generated FRB Dart to “make tests pass”
// lib/src/rust/api/simple.dart  ← FORBIDDEN

// Asserting private State
final state = tester.state<_MyHomePageState>(find.byType(MyHomePage));
expect(state._result, isNotEmpty);

// Finding only by nested layout types
expect(find.byType(Padding), findsWidgets);

// Unconditional static chrome
expect(find.text('Flutter + Rust Demo'), findsOneWidget); // AppBar title never branches

// Wrong runner / stack
import 'package:jest/...'; // does not exist — use flutter_test
```

Topic-specific anti-patterns live in the specialized topic files above.
