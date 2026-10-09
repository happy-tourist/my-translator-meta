---
name: verify-mock
description: >-
  Compares Flutter UI implementation (my_translator_app/lib) against an attached
  or referenced design mock: spacing, hex/rgba colors light+dark, typography/copy,
  icons, dividers, alignment, button height, vertical fill. Strictly enforces
  Visual Spec sizes and colors as Violations. Uses design Visual Spec from
  prepare-mock when present. Reports Violations / Warnings / Recommendations.
  Use when the user asks to verify-mock, check UI vs mock/макет, or when
  implement-change runs the mock phase after align-code.
---

# Verify Mock — Flutter UI vs design mock

Сверить **реализованный Flutter UI** (`lib/`) с **макетом**. Канон чисел/цветов — Visual Spec из [`prepare-mock`](../prepare-mock/SKILL.md).

Только анализ (standalone). В `implement-change` после отчёта — override «Добавь и поправь…».

## Когда применять

- `verify-mock` / сверка UI с макетом
- Фаза mock в `implement-change` после align-code

## Пути

| Ключ | Default | Роль |
|------|---------|------|
| `app` | `../my_translator_app` | runtime UI `lib/` |
| meta | `.` | OpenSpec / Visual Spec |

Перед чтением app — `{app}/AGENTS.md` если есть.

## Источник макета

Порядок как в prepare-mock. `Using mock: <path>`. Нет макета: standalone → стоп; в implement-change → `SKIPPED: no mock`.

Anti-miss: бледные dividers, сетка, vertical fill, каждый size/color токен Visual Spec, hover enlarge tension.

## Scope

1. Resolve OpenSpec change.
2. Surfaces из Scope / Visual Spec.
3. Найти widgets/theme/i18n в `lib/` (grep / design Decision).
4. Сверить код с макетом (+ Visual Spec exact/in-range). Опционально device/screenshot — не блокер.

Не раздувать на rust-only без UI на макете.

## Tiers

- **Violations** — размер/цвет/copy/иконка вне Spec или грубо мимо макета
- **Warnings** — tension Spec vs Material defaults; hover enlarge; soft chrome
- **Recommendations** — polish

Flutter specifics: `ThemeData`/`TextTheme` vs hardcoded; `EdgeInsets` vs Spec pads; `Color(0x…)` / theme scheme vs Spec hex.

## Report (Russian)

```markdown
## Verify Mock
**Mock:** …
**Change:** …
**Surfaces:** …

### Violations
### Warnings
### Recommendations

## Evidence
- `lib/...` L…
```

## Не делать

- Не править код в standalone без просьбы.
- Не сверять rust-ядро как UI.
- Не подменять align-code.
