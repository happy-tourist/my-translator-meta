## Why

При первом нажатии Translate в сессии приложения UI «замирает»: офлайн-модель OPUS-MT поднимается лениво только в этот момент (сотни MB assets + init движка). Повторные переводы быстрые, но после полного выхода и нового запуска зависание повторяется. Нужно прогревать движок тихо при старте и сохранить отзывчивый loading на кнопке, если tap случился раньше готовности — на Web, Windows и Android.

## What Changes

- Тихий фоновый warm-up offline translate после первого кадра **product shell** (Library/Vocab после открытия storage; native + Web).
- Lazy init на Translate остаётся fallback: пока движок не ready — ждём на кнопке со spinner.
- Cold path MUST NOT «замораживать» UI (кадры / жесты) на время загрузки и init.
- Без splash и без отдельного статуса прогрева вне кнопки Translate.

## Scope

- **Capability ID:** `translate/offline`
- Продуктовое поведение прогрева и отзывчивости Translate (Add/Edit Vocab sheets и эквивалентные точки вызова через тот же facade).
- Слои: Flutter facade / shell orchestration; Web Transformers.js worker bridge; native путь через существующий FRB init (без обязательной смены публичного bytes-контракта).
- Платформы приёмки: **Web (Chrome), Windows, Android**.

## Out of scope

- Смена модели / пары языков / облачный или AI translate.
- Блокирующий splash «Готовим модель…» или постоянный индикатор в shell/settings.
- Persist «уже прогретого» движка между процессами; OS-path к модели в публичном FRB API.
- Предзагрузка при входе только в Reader (warm-up при старте app, не отложенный до Reader).
- Изменения dictionary persist, reader gestures, parse/tokenize.

## Capabilities

### New Capabilities

- (нет)

### Modified Capabilities

- `translate/offline`: фоновый warm-up при старте сессии; Translate при незавершённом init ждёт с живым loading; UI остаётся отзывчивым на cold path (Web + native).

## Impact

- Flutter: тихий warm-up после первого кадра **product shell** (не bootstrap открытия storage); общий process-wide ensure facade (native + Web); loading на Translate в Vocab/Reader; escape hatch warm-up для widget-тестов.
- Web: явный warmup до первого translate с общим in-flight pipeline.
- Native: без смены публичного FRB bytes→init; тяжёлый init не в app bootstrap; verify-only регрессия translate-тестов (без обязательного codegen).
- Платформы приёмки и автопокрытие: Web (Chrome), Windows, Android; widget/facade на warm-up / wait / responsive cold path.

## References

- Main spec: `openspec/specs/translate/offline/spec.md`
- Archive: `openspec/changes/archive/2026-10-13-add-offline-opus-translate/`
- Skills: `work-with-translate`, `work-with-pages` (shell), `work-with-env-deploy`; sibling app `AGENTS.md` (offline translate dual runtime)
- Карта путей: `docs/projects-map.md`
