# ThemeMode / ColorScheme / reader tokens

Read with the [core styles skill](SKILL.md).

## Runtime Light / Dark

Preference: `ThemeMode.light` | `ThemeMode.dark` | `ThemeMode.system`
(follows OS when system). Wire on `MaterialApp`:

```dart
MaterialApp(
  theme: AppTheme.light,       // ThemeData with ColorScheme + ReaderColors
  darkTheme: AppTheme.dark,
  themeMode: ThemeMode.system, // present; Settings toggle later
  // …
);
```

| Concern | Behavior | Where |
|---------|----------|-------|
| Present wiring | `AppTheme.light` / `.dark` + `ThemeMode.system` | `lib/app/app.dart` + `lib/app/theme.dart` |
| Persistence / toggle | Later Settings — keep one control when added | `lib/features/settings/` |
| Reader highlights | Same lemma tokens in both modes (tuned pair) | `ReaderColors` on both themes |

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

Present extension (`lib/app/theme.dart`):

```dart
@immutable
class ReaderColors extends ThemeExtension<ReaderColors> {
  const ReaderColors({
    required this.lemmaHighlight,
    required this.lemmaHighlightOn,
  });

  final Color lemmaHighlight;   // fill behind is_in_vocab tokens
  final Color lemmaHighlightOn; // ink on highlight if needed

  // light: 0xFFC8E6C9 / 0xFF1B5E20; dark: 0xFF1B3D24 / 0xFFA5D6A7
}

// Usage in cue_row
final rc = Theme.of(context).extension<ReaderColors>()!;
// WidgetSpan / TextStyle background → rc.lemmaHighlight
```

Rules:

- Define **light and dark** pairs; verify contrast on reader surface.
- Selection chrome (Add CTA / selected token) stays separate from vocab
  highlight; do not overload `lemmaHighlight` for phrase selection unless a
  change adds a dedicated token.
- Vocab list badges / chips reuse the same extension — do not invent a second
  green.
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
