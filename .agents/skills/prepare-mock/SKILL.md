---
name: prepare-mock
description: >-
  Extracts measurable UI requirements from an attached design mock (sizes,
  padding, button heights, radii, hex/rgba colors light+dark, dividers, grids,
  copy, gaps, icons, text alignment, hover/focus) for Flutter implementation.
  Produces a structured inventory and folds it into OpenSpec design/requirements.
  Use when the user runs prepare-mock, asks to measure a mock/макет/screenshot
  into specs, or to capture visual tokens before propose/apply.
---

# Prepare Mock — снять с макета требования (Flutter)

Из **прикреплённого макета** снять измеримые UI-требования и перенести в OpenSpec (`design.md` → **Visual Spec (from mock)**; copy — в specs при необходимости).

Не пишет runtime и не коммитит. Не подменяет [`verify-mock`](../verify-mock/SKILL.md) / [`prepare-changes`](../prepare-changes/SKILL.md).

Цепочка: **prepare-mock** → propose/update → apply → **verify-mock**.

## Когда применять

- `prepare-mock` / снять размеры с макета
- Перед propose, когда есть скрин без Visual Spec

## Пути

| Ключ | Default | Роль |
|------|---------|------|
| meta | `.` | OpenSpec change |
| `app` | `../my_translator_app` | опционально: существующие Theme/TextStyle/tokens в `lib/` |

## Источник макета

1. Вложения / `file_attachments` / `assets/…`
2. Макеты сессии
3. Ссылки в active change + файлы рядом
4. Явный путь

`Using mock: <path>`. Нет макета — стоп. Read (vision). Light/dark — оба набора.

Уточнение человека по макету — ground truth.

### Anti-miss

Бледные dividers; «колонки»; выравнивание текста; font-size per role; vertical fill; button height; размеры всех блоков; hover table; цвета hex/rgba light+dark; icon size+ink. Approx OK с confidence. Не invent.

Токены формулировать так, чтобы их можно было выразить во Flutter: `EdgeInsets`, `SizedBox`, `BorderRadius`, `TextStyle`, `Color(0xAARRGGBB)` / theme tokens — **числа и hex в Visual Spec**, не «как Material default» без чисел при pixel-faithful.

## Резолв change

User name → single active → dialog → `OpenSpec: none` (только extract-отчёт).

## Hard Boundary

Не писать `lib/` / `rust/`. Писать только meta artifacts (design/specs) когда change выбран.

## Workflow

```
Prepare-Mock Progress:
- [ ] 1. Resolve mock + change
- [ ] 2. Vision inventory (structure, copy, sizes, pads, type, colors, icons, hover)
- [ ] 3. Map to Flutter-expressible tokens
- [ ] 4. Write/update design.md ## Visual Spec (from mock)
- [ ] 5. Optionally sync copy into delta specs
- [ ] 6. Short report + gaps / Open Questions
```

Секция Visual Spec — тот же чеклист блоков, что ожидает `align-implementability-front` / `verify-mock`.

## Report

Кратко: mock path, change, что записано в Visual Spec, Open Questions, next: propose/update/apply.

## Не делать

- Не выдумывать пиксели без макета.
- Не сверять уже написанный UI (это verify-mock).
- Не ссылаться на web UI frameworks — только Flutter Material.
