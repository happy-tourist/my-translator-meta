---
name: end-implement-change
description: >-
  Closes an active OpenSpec change end-to-end: openspec-update-change →
  openspec-sync-specs → openspec-archive-change → commit. Use when the user asks to
  end-implement-change, finalize/close a change after implementation, or run
  update → sync → archive → commit without pausing for questions.
---

# End Implement Change — update → sync → archive → commit

Оркестратор закрытия **активного** OpenSpec change после реализации. Запускает четыре скилла **подряд**, без пауз и без вопросов пользователю.

## Когда применять

- Пользователь запускает `end-implement-change` / просит закрыть change после implement
- Нужен end-to-end: update артефактов → sync delta в main specs → archive → commit (+ push)

Не подменять одиночный `openspec-update-change` / `openspec-sync-specs` / `openspec-archive-change` / `commit`, если пользователь явно хочет только один шаг.

## Жёсткое правило: без уточнений

**Ничего не спрашивать.** На любой prompt / confirmation / choice из дочерних скиллов — решить самому и продолжить.

Defaults:

| Вопрос дочернего скилла | Решение |
|-------------------------|---------|
| Какой change? | Имя от пользователя → иначе из контекста → иначе единственный active → иначе самый недавно изменённый из `openspec list --json` |
| Coherence / что обновлять в artifacts? | Полный coherence-проход: согласовать существующие артефакты с реализованным состоянием |
| Неполные artifacts при archive? | Продолжить |
| Неполные tasks при archive? | Продолжить |
| Sync delta перед archive? | Уже сделан в фазе 2 → **Archive now**. Если после фазы 2 ещё есть drift — **Sync now**, затем archive |
| Sync anyway / Cancel? | **Archive now** |
| Commit messages / подтверждение commit? | Составить сообщения по diff (как в `commit`); не ждать OK. Запуск `end-implement-change` = разрешение на stage → commit → push |

Override «ask the user» из дочерних скиллов **не действует** в этом оркестраторе.

Останавливаться только при жёстком blocker: CLI/IO error, archive target уже существует, sync validation failed и повтор не помог, нет ни одного active change, git/auth blocker у `commit`.

## Делегируемые скиллы

| Фаза | Skill |
|------|--------|
| 1. Update | [`.agents/skills/openspec-update-change/SKILL.md`](../openspec-update-change/SKILL.md) |
| 2. Sync | [`.agents/skills/openspec-sync-specs/SKILL.md`](../openspec-sync-specs/SKILL.md) |
| 3. Archive | [`.agents/skills/openspec-archive-change/SKILL.md`](../openspec-archive-change/SKILL.md) |
| 4. Commit | [`.agents/skills/commit/SKILL.md`](../commit/SKILL.md) |

## Вход / выбор change

1. Имя от пользователя, иначе из контекста диалога.
2. Иначе `openspec list --json` — если ровно один active → его.
3. Иначе → самый недавно изменённый (`lastModified`), без вопроса.

Объявить: `Using change: <name>`.

## Workflow

```
End-Implement-Change Progress:
- [ ] 1. Select active change
- [ ] 2. openspec-update-change
- [ ] 3. openspec-sync-specs
- [ ] 4. openspec-archive-change
- [ ] 5. commit (stage → commit → push) — meta + app
- [ ] 6. Short final report
```

### 1. Update

Выполнить `openspec-update-change` для `<name>`: coherence существующих planning-артефактов; писать сразу; не править runtime.

### 2. Sync

Выполнить `openspec-sync-specs` для того же `<name>`: все delta → main specs; validate. Нет delta — «no delta» и дальше.

### 3. Archive

Выполнить `openspec-archive-change`: incomplete → proceed; после sync → **Archive now**; `mv` в `archive/YYYY-MM-DD-<name>`.

### 4. Commit

Выполнить [`commit`](../commit/SKILL.md) целиком (stage → commit → push по dirty-репо **meta + app**).

### 5. Final report

```markdown
# End Implement Change — итог

**Change:** <name>
**Update:** OK | fail
**Sync:** OK | no-delta | fail
**Archive:** OK | fail
**Commit:** meta … / app …
**Stopped:** нет | <blocker>
```

## Не делать

- Не спрашивать подтверждений.
- Не править runtime вне update (update — только planning artifacts).
- Не пропускать sync перед archive, если есть delta.
