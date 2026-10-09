---
name: work-with-localization
description: >-
  Use when adding or changing Flutter UI locale strings in my_translator_app
  (Translation Reader): ARB / gen-l10n, AppLocalizations, MaterialApp
  localizationsDelegates, or distinguishing UI chrome i18n from learning
  language pair (subtitle / translate content languages). Not for Rust
  NLP/translate catalogs or FRB message codes.
---

# Work With Localization

Use this skill for **Flutter UI chrome** strings in **my_translator_app**
(product: **Translation Reader**).

Stack (target): Flutter **ARB + `gen-l10n`** → generated `AppLocalizations`,
wired through `MaterialApp` (`localizationsDelegates` / `supportedLocales`).
App canon: `{app}/AGENTS.md`. Folder roles: `client-work-with-structure`.

There is **no** TypeScript i18n catalog or web-framework boot file in this
app. Do not invent unrelated chrome namespaces — UI keys stay product
chrome for Translation Reader.

## UI locale vs learning languages (critical)

| Concern | What it is | Where it lives |
|---------|------------|----------------|
| **UI locale** | Chrome copy: buttons, labels, empty states, settings labels, a11y | Flutter ARB / `AppLocalizations` |
| **Learning language pair** | Source / target of subtitles and word/phrase translation (e.g. EN↔RU) | Settings + Rust (persist / NLP / translate) — **not** ARB |

Never store subtitle content, dictionary lemmas, or translate-provider prompts
as UI l10n keys. Never use UI locale alone to decide tokenization or offline
translate direction.

## Reality check

As of scaffold (`pubspec.yaml` without `flutter_localizations`, no `l10n.yaml`,
no `*.arb`), **gen-l10n is not wired**. Demo copy in `lib/main.dart` may be
hardcoded.

This skill describes the **target** approach. When first introducing UI i18n:

1. Add the SDK dependency and enable code gen (below).
2. Create the ARB template + wire `MaterialApp`.
3. Put **new** user-facing chrome on keys; do **not** mass-migrate hardcoded
   strings unless the user asks.

Do not invent a parallel DIY JSON/Map i18n layer when gen-l10n is available.

## Rules

| Do | Don't |
|----|--------|
| Put new UI chrome in ARB and use `AppLocalizations` / `.of(context)` | Hardcode new shared chrome when adding fresh product strings |
| Keep one generated localizations API (Flutter `gen-l10n`) | Add easy_localization or a second hand-rolled catalog unless asked |
| Separate UI locale from learning language pair in settings UX and persistence | Conflate `Locale` with subtitle source/target codes |
| Use ICU placeholders / plurals in ARB (`{name}`, plural forms) | Concatenate translated fragments in Dart when a single message can hold the params |
| Mirror key names across locale ARB files | Add country masks, phone locale maps, or a locale switcher unless asked |

## Target layout

| Path | Role |
|------|------|
| `pubspec.yaml` | `flutter_localizations` (sdk), `intl` as needed; `flutter: generate: true` |
| `l10n.yaml` | gen-l10n config (`arb-dir`, `template-arb-file`, `output-localization-file`) |
| `lib/l10n/app_en.arb` | Template / master schema (keys + English or agreed template copy) |
| `lib/l10n/app_<locale>.arb` | Additional UI locales (e.g. `app_ru.arb`) when added |
| Generated under `.dart_tool` / configured output | `AppLocalizations` — **do not hand-edit** |
| `lib/app/` (or `MaterialApp` host) | `localizationsDelegates`, `supportedLocales`, optional `locale` |

Suggested `l10n.yaml` (adjust only if the app already chose different names):

```yaml
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
```

`pubspec.yaml` (target snippets):

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter
  intl: any # or a pinned version consistent with Flutter SDK

flutter:
  generate: true
  uses-material-design: true
```

## Bootstrap (target)

```dart
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';
// Import path follows project gen-l10n output; prefer the generated
// AppLocalizations import Flutter creates for this package.

MaterialApp(
  localizationsDelegates: const [
    AppLocalizations.delegate,
    GlobalMaterialLocalizations.delegate,
    GlobalWidgetsLocalizations.delegate,
    GlobalCupertinoLocalizations.delegate,
  ],
  supportedLocales: AppLocalizations.supportedLocales,
  // locale: explicit override only if settings persist a UI locale choice
  // home / router …
);
```

Call sites:

```dart
final l10n = AppLocalizations.of(context)!;
Text(l10n.someKey);
// With placeholder: l10n.greeting(userName);
```

Prefer `AppLocalizations.of(context)!` (or a null-safe pattern already used nearby)
over global singletons. Do not call FRB for chrome strings.

## Adding / changing strings

1. If gen-l10n is not enabled yet and the task needs UI i18n, wire the layout
   above first (dependency + `l10n.yaml` + template ARB + `MaterialApp`).
2. Open the template ARB (`lib/l10n/app_en.arb`).
3. Add a clear key (prefer feature-ish prefixes when catalogs grow, e.g.
   `readerEmpty`, `vocabAdd`, `settingsUiLanguage` — **not** learning-pair
   codes as message keys).
4. Add the same key to other locale ARB files if they exist.
5. Run code gen (`flutter gen-l10n` or `flutter pub get` with `generate: true`).
6. Use the generated getter / method in the widget.

**ARB shape (illustrative — not a product catalog):**

```json
{
  "@@locale": "en",
  "appTitle": "Translation Reader",
  "readerEmpty": "No subtitles open",
  "vocabAdd": "Add to vocabulary",
  "settingsUiLanguage": "Interface language",
  "settingsLearningPair": "Learning languages",
  "greeting": "Hello, {name}",
  "@greeting": {
    "placeholders": {
      "name": { "type": "String" }
    }
  }
}
```

Learning-pair **labels** in settings (e.g. “English”, “Russian”) may be UI
strings; the **codes** (`en`, `ru`) and which side is source/target are app
settings / Rust — keep them out of ARB as the source of truth.

## Checklist

- [ ] New UI chrome lives in ARB (or wiring gen-l10n is in the same change)
- [ ] Widget uses `AppLocalizations` (not a fresh hardcoded string for new copy)
- [ ] UI locale ≠ learning language pair in design and persistence
- [ ] No hand-edit of generated l10n Dart; no second i18n framework
- [ ] No country / phone / brand localization scaffolding unless requested
- [ ] No unrelated chrome key dumps outside product UI strings
)
