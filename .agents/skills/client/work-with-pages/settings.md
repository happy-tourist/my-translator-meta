# Settings

Read with the [core pages skill](SKILL.md) for shell/nav. Folder roles:
`client-work-with-structure` → `lib/features/settings/`.

## Target screen

| Screen | Suggested path | Role |
|--------|----------------|------|
| Settings | `lib/features/settings/settings_screen.dart` | Languages and app preferences |

Primary shell destination ([shell.md](shell.md)).

## UX canon (target)

**Near-term**

- Source / learning language preferences (what the user reads vs studies).
- Theme / display prefs if product wants them in-app (otherwise system theme
  from shell is enough).
- Persist via Rust/SQLite when settings API exists; until then, keep UI
  structure ready without inventing a second persistence stack in Dart.

**Later (explicitly optional — do not build until asked)**

- Optional AI provider selection / keys — AGENTS: not required at start; no
  mandatory cloud APIs.
- Never commit secrets; if keys are stored, follow a future OpenSpec for secure
  storage.

## Layering

```text
SettingsScreen
  → orchestration: load/save settings
      → FRB settings / prefs API (Rust) when available
```

- Shell owns theme application; Settings may **change** theme preference and
  notify shell — do not fork two theme engines.
- No account/auth or staff-tools screens.

## Do not

- Add monetization or cloud sync toggles (out of scope).
- Require an API key to use core offline Reader / dictionary flows.
- Expand i18n catalogs unless the user asks (hard restriction in SKILL.md).
