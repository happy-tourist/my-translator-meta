---
name: work-with-styles
description: >-
  Use when adding, changing, reviewing, or debugging Flutter Material styles in
  my_translator_app (Translation Reader): ThemeData light/dark, ColorScheme,
  TextTheme, Theme.of(context), lemma/vocab highlight colors, reader typography,
  spacing/EdgeInsets conventions, hardcoded Color/TextStyle vs theme tokens, or
  mapping prepare-mock / verify-mock Visual Spec hex/sizes into theme. Topic
  details in theme.md (ThemeMode + ColorScheme + persistence). Flutter
  Material only.
---

# Work With Styles

## Overview

Use this skill when working with **visual styles** in **my_translator_app**
(product: **Translation Reader**).

Stack (from `{app}/AGENTS.md`): Flutter / Dart Material (`lib/`), Rust core
(`rust/`), FRB generated Dart (`lib/src/rust/` — do not edit). Styles are
**Flutter `ThemeData` + Material 3** — not a second CSS/Sass design-token
package.

Path package: `package:my_translator_app/...`. App path via
`docs/projects-map.md` → `{app}`.

Skills for this client live under `.agents/skills/client/` in
`my_translator_meta`.

## Specialized Topics

Read the matching file in this folder when the change involves that area
(do not load every file at once):

| Topic | File |
|-------|------|
| ThemeMode light/dark/system, ColorScheme, TextTheme, persistence, lemma highlight tokens | [theme.md](theme.md) |

No board-game / pack-card CSS topics — those are not part of this app.

## Project Style Model

| Layer | Role | Location (target) |
|-------|------|-------------------|
| App themes | `ThemeData` light + dark, `ThemeMode` | `lib/app/theme.dart` (or `lib/app/`) wired from `MaterialApp` |
| Color roles | Material `ColorScheme` (primary, surface, onSurface, error, …) | Same theme file(s) |
| Type roles | `TextTheme` / `primaryTextTheme` for chrome + reader body | Same theme file(s) |
| Product accents | Lemma/vocab highlight, selection ink (not plain Material roles) | `ThemeExtension` or named tokens next to theme — see [theme.md](theme.md) |
| Widget chrome | Prefer `Theme.of(context)` / `ColorScheme` / `TextTheme` | Screens & widgets under `lib/features/…`, `lib/widgets/` |
| Mock-driven pixels | Exact hex / sizes from Visual Spec | OpenSpec `design.md` → fold into theme tokens once; then widgets read theme |

Do **not** add a second theme engine (CSS variables, brand-profile packages,
or non-Material dark toggles). One Material theme pair + optional
`ThemeExtension`s.

Present: `lib/app/theme.dart` (`AppTheme.light` / `.dark` + `ReaderColors`)
wired from `TranslationReaderApp` with `ThemeMode.system`. Do not reintroduce
`primarySwatch` demo theme.

## Default Choices

- Prefer **`Theme.of(context)`** (or `Theme.of(context).colorScheme` /
  `.textTheme`) over literal `Color(0x…)`, `Colors.grey`, or ad-hoc
  `TextStyle(fontSize: …)` in widgets.
- Prefer Material widgets and theme roles (`ElevatedButton` / `FilledButton`,
  `AppBar`, `ListTile`, `Divider`) before custom paint — customize via theme
  first.
- Reader body type → dedicated `TextTheme` role or a named reader style on the
  theme / extension (comfortable line height for subtitle lines). Chrome
  (app bar, buttons, vocab list) uses standard Material type roles.
- Spacing → named constants or theme-adjacent spacing helpers (`AppSpacing`)
  shared across features; prefer multiples of 4/8. Do not scatter magic
  `EdgeInsets.all(13)` unless a Visual Spec requires that exact value.
- Icons → Material Icons (`Icons.*`) unless an OpenSpec/mock specifies custom
  assets.
- Lemma highlight fill/ink → **theme tokens** (extension or scheme extensions),
  not one-off `Colors.yellow` in the painter. Highlight **which** spans come
  from Rust token/lemma data; Flutter only paints with theme colors.
- Mock-driven work → read Visual Spec from `prepare-mock`; after implement,
  `verify-mock` checks sizes/colors. Map Spec hex into **theme** (light+dark),
  then widgets consume theme — do not leave Spec hex only in leaf widgets.

## Theme vs Hardcode

| Prefer | Avoid |
|--------|-------|
| `Theme.of(context).colorScheme.primary` | `Color(0xFF1976D2)` in a screen |
| `textTheme.bodyLarge` / reader role | `TextStyle(fontSize: 16)` copied per line |
| `colorScheme.surfaceContainerHighest` (or extension) for highlight | `Colors.amber.withOpacity(0.3)` inline |
| Shared `AppSpacing.md` / Spec-backed pad constant | Unrelated `EdgeInsets` values per widget |
| Spec hex → once in `theme.dart` (+ dark pair) | Same hex duplicated in 5 widgets |

**Exception:** while folding a Visual Spec, temporary literals in `theme.dart`
(or a tokens file) are OK. Widgets still go through `Theme` /
`ThemeExtension`. Do not bypass theme “because Spec said `#AABBCC`”.

## Reader Typography And Spacing

Target UX (AGENTS): cue lines, tap targets, lemma highlight, word/phrase card,
split panes, vocab list.

| Concern | Guidance |
|---------|----------|
| Cue / reader body | Slightly larger than chrome body; clear line height; same style for original pane; translation pane may use a secondary role (e.g. `bodyMedium` / muted onSurface) |
| Tap / selection | Selection and highlight colors from theme tokens; keep contrast in **both** light and dark |
| Word/phrase card | Use Material surface + `title`/`body` roles; pad from shared spacing or Spec |
| Split view | Pane chrome and dividers from `ColorScheme.outline` / `DividerTheme`; do not hardcode grey bars |
| Vocab list | Standard list density; lemma chip/badge uses the **same** highlight tokens as reader |
| Touch targets | Prefer ≥48dp interactive height unless Spec mandates otherwise |

Typography tokens that appear in a mock Visual Spec (font size, weight, line
height) become `TextStyle` entries on `TextTheme` or a `ReaderTypography`
extension — not anonymous styles in `build()`.

## Visual Spec Cross-Refs

| Skill | Role |
|-------|------|
| `prepare-mock` | Measures mock → OpenSpec **Visual Spec** (hex light+dark, pads, type, radii) |
| `verify-mock` | Compares `lib/` to Spec; flags hardcoded vs `ThemeData`/`TextTheme` tension |

When implementing mock-driven UI:

1. Copy Spec numbers/colors into theme / spacing tokens (light **and** dark).
2. Widgets read theme / shared spacing.
3. Leave Spec as the meta authority; do not invent a parallel Sass/CSS token
   layer beside Material theme.

## Where Style Files Live

| Area | Path (target) |
|------|----------------|
| `MaterialApp` / `theme` / `darkTheme` / `themeMode` | `lib/app/app.dart` (+ `lib/app/theme.dart`) |
| ColorScheme + TextTheme + extensions | `lib/app/theme.dart` (or `lib/app/theme/`) |
| Spacing / radius constants | `lib/app/theme.dart` or `lib/app/spacing.dart` |
| Feature UI | `lib/features/<area>/` — consume theme; no local palette files |
| Shared widgets | `lib/widgets/` — same rule |
| Generated FRB | `lib/src/rust/` — never for colors |

Align folder placement with `client-work-with-structure`.

## How To Add Or Change Styles

1. **Chrome vs reader accent?** Ordinary Material role → `ColorScheme` /
   `TextTheme`. Lemma highlight / selection → extension token ([theme.md](theme.md)).
2. **Light/dark?** Define both `theme` and `darkTheme`; drive with `ThemeMode`
   (see [theme.md](theme.md)). Never retune only light.
3. **Mock Spec present?** Map hex/sizes into theme tokens first; then widgets.
4. **Spacing/radius?** Shared constants or Spec-backed values; prefer theme
   `CardTheme` / `ButtonTheme` / `InputDecorationTheme` for control chrome.
5. **Do not** port web utility-class CSS, scoped CSS, or board/pack-card SCSS.
6. **Do not** put style truth in Rust/FRB — Rust supplies **which** lemmas to
   highlight; Flutter supplies **how** they look.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Web CSS / Sass patterns | Material `ThemeData` only |
| `Colors.blue` / random hex in every widget | Centralize in theme; use `Theme.of(context)` |
| Only light `ThemeData` | Pair `darkTheme` + test both |
| Lemma highlight as raw `Colors.yellow` | ThemeExtension / named highlight tokens, light+dark |
| Duplicating Spec hex in leaves | Spec → theme → widgets |
| Reader type as one-off `TextStyle` per cue | `TextTheme` / reader extension |
| Theme toggle per screen | One shell/settings control ([theme.md](theme.md)) |
| Remote theme sync | Local only (prefs or Rust settings when exist) |
| Styling which-token logic in Flutter | Rust lemma match; Flutter paints |
| Hand-editing generated FRB for UI colors | Never |
| Dumping all styles into `main.dart` scaffold | `lib/app/theme.dart` + feature widgets |

## Related Skills

- Structure / `lib/` placement → `client-work-with-structure`
- Mock measurement → `prepare-mock`
- UI vs mock check → `verify-mock`
- Flutter implementability → `align-implementability-front`
- App AGENTS → `{app}/AGENTS.md` via `docs/projects-map.md`

## Verification

- [ ] Styles go through `ThemeData` / `ColorScheme` / `TextTheme` / extensions
- [ ] Light and dark both defined when chrome is themed
- [ ] Widgets use `Theme.of(context)` (no scattered hardcoding)
- [ ] Lemma/vocab highlight uses shared theme tokens
- [ ] Reader typography is a named theme role, not anonymous styles
- [ ] Spacing/radii shared or Spec-backed consistently
- [ ] Mock-driven: Spec tokens folded into theme; `verify-mock` can check them
- [ ] No web CSS/Sass or board/pack-card leftovers
- [ ] No style logic in Rust/FRB beyond “which spans to paint”
