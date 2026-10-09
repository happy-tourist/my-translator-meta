# ThemeMode / ColorScheme / reader tokens

Read with the [core styles skill](SKILL.md).

## Runtime Light / Dark

Preference: `ThemeMode.light` | `ThemeMode.dark` | `ThemeMode.system`
(follows OS when system). Wire on `MaterialApp`:

```dart
MaterialApp(
  theme: AppTheme.light,       // ThemeData with ColorScheme + TextTheme
  darkTheme: AppTheme.dark,
  themeMode: controller.themeMode, // from local settings
  // …
);
```

| Concern | Behavior | Where (target) |
|---------|----------|----------------|
| Early apply | Restore saved mode before first frame if possible; else `system` | `main` / app bootstrap after `RustLib.init()` |
| Persistence | Local only — `SharedPreferences` / in-memory until Rust settings API exists | settings orchestration or small theme controller |
| Toggle UI | One control in app shell or Settings — light ↔ dark (and optional “System”) | `lib/features/settings/` or `lib/app/` shell |
| Reader highlights | Same lemma tokens in both modes (tuned pair, not light-only) | `ThemeExtension` / highlight tokens on both themes |

Do **not** duplicate the toggle on every screen. Do **not** invent a second
non-Material dark engine.

## ColorScheme

Build both themes from explicit schemes (seed or hand-tuned). Prefer Material 3
(`useMaterial3: true`).

```dart
// Sketch — put real product hex in lib/app/theme.dart
ThemeData light = ThemeData(
  useMaterial3: true,
  colorScheme: ColorScheme.fromSeed(
    seedColor: const Color(0xFF1565C0),
    brightness: Brightness.light,
  ),
  textTheme: /* chrome + reader roles */,
  extensions: const [ReaderColors.light],
);

ThemeData dark = ThemeData(
  useMaterial3: true,
  colorScheme: ColorScheme.fromSeed(
    seedColor: const Color(0xFF1565C0),
    brightness: Brightness.dark,
  ),
  textTheme: /* paired dark type */,
  extensions: const [ReaderColors.dark],
);
```

Map Visual Spec brand/chrome hex into `ColorScheme` roles (`primary`,
`surface`, `onSurface`, `error`, `outline`, …). If Spec invents a role Material
lacks (lemma highlight), use a **`ThemeExtension`** — do not overload
`secondary` for unrelated meanings without documenting it.

### Consumption

```dart
final scheme = Theme.of(context).colorScheme;
final text = Theme.of(context).textTheme;
// Backgrounds / ink
Container(color: scheme.surface);
Text('Cue', style: text.bodyLarge?.copyWith(color: scheme.onSurface));
```

## TextTheme (chrome vs reader)

| Role | Typical use |
|------|-------------|
| `titleLarge` / `titleMedium` | App bar, screen titles, vocab entry head |
| `bodyLarge` | Default chrome body; **or** promote to reader cue style if Spec says so |
| `bodyMedium` / `bodySmall` | Secondary labels, translation pane, meta |
| `labelLarge` | Buttons, chips |

If reader cues need a size/height distinct from chrome, add a clear named style
(extension or documented `textTheme` role) used by cue widgets only — keep one
definition, light+dark.

Line height matters for subtitle reading: set `height` on the reader style;
do not rely on Material defaults if Spec gives a number.

## Lemma / Vocab Highlight Tokens

Rust decides **which** token spans match dictionary lemmas. Flutter paints
them with **theme** colors so light/dark and vocab chips stay consistent.

Suggested extension shape:

```dart
@immutable
class ReaderColors extends ThemeExtension<ReaderColors> {
  const ReaderColors({
    required this.lemmaHighlight,
    required this.lemmaHighlightOn,
    required this.phraseSelection,
  });

  final Color lemmaHighlight;   // fill behind known lemma
  final Color lemmaHighlightOn; // ink on highlight if needed
  final Color phraseSelection;  // active tap / phrase select

  // light + dark const instances; lerp implementation…
}

// Usage
final rc = Theme.of(context).extension<ReaderColors>()!;
// Text.rich / CustomPainter / WidgetSpan backgrounds use rc.lemmaHighlight
```

Rules:

- Define **light and dark** pairs; verify contrast on reader surface.
- Vocab list badges / chips reuse the same extension — do not invent a second
  yellow.
- Do not bake highlight hex into Rust FFI models.
- Mock Spec highlight colors → these extension fields (both modes).

## Spacing And Shape

Keep next to theme (e.g. `AppSpacing`, `AppRadii`) or on `ThemeData`
component themes:

| Token | Use |
|-------|-----|
| `xs/sm/md/lg` (4/8/16/24…) | Default pads/gaps |
| Spec exact pads | Named constants (`readerCuePadding`) when mock-driven |
| `BorderRadius` | `CardTheme` / `ButtonTheme` / shared radii |
| Dividers | `DividerTheme` + `colorScheme.outlineVariant` |

Widgets: `padding: EdgeInsets.all(AppSpacing.md)` or Spec constant — not
unrelated magic numbers.

## Persistence Notes

| Stage | Approach |
|-------|----------|
| Scaffold / early UI | `ThemeMode.system` or in-memory notifier |
| Local prefs | Persist enum/`string` (`light`/`dark`/`system`) on device |
| Rust settings (target) | When settings API exists, store with other user settings via FRB; UI still only applies `ThemeMode` |

Failures loading prefs → fall back to `ThemeMode.system`.

## Wiring Checklist

1. `AppTheme.light` / `AppTheme.dark` with `ColorScheme` + `TextTheme` +
   `ReaderColors` (or equivalent).
2. `MaterialApp(theme:, darkTheme:, themeMode:)`.
3. Single toggle/settings entry updates mode + persists.
4. Reader / vocab widgets read `ColorScheme` + `ReaderColors` only.
5. Visual Spec hex (if any) live in theme files, both brightnesses.
6. Smoke both modes on reader highlight + chrome.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Only `theme:` without `darkTheme:` | Always pair |
| `ThemeMode` not persisted / reset every launch | Local prefs or Rust settings |
| Highlight colors only for light | Dark pair + contrast check |
| `primarySwatch` scaffold left as product theme | Replace with M3 `ColorScheme` |
| Extension forgotten → widgets use `Colors.amber` | Add extension; migrate call sites |
| Changing type size in one cue widget only | Change reader `TextTheme` / extension once |
