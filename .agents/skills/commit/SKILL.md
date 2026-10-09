---
name: commit
description: >-
  Scans uncommitted changes across my_translator_meta and my_translator_app;
  always stages, commits on main/master, and pushes. Use when the user runs the
  commit skill or asks to commit across the ecosystem («закоммитить»); push is
  always included — no separate push request needed.
---

# Commit — meta + app

Проверить незакоммиченные изменения в **meta** и **app**, застейджить, закоммитить и **всегда запушить**. Запуск этого скилла = полный цикл `stage → commit → push`. Отдельное слово «пуш» в запросе не требуется.

Рабочие ветки: **main** или **master**.

## Когда применять

Только если пользователь **явно** запускает commit / просит закоммитить по экосистеме (этот скилл), **или** оркестратор `implement-change` / `end-implement-change` делегирует эту фазу. Не вызывать «на всякий случай» после правок кода.

## Репозитории

Разрешить пути от корня `my_translator_meta` через [`docs/projects-map.md`](../../../docs/projects-map.md) (+ `projects-map.local.yaml` если есть):

| Ключ | Репозиторий | Default path | Default branch |
|------|-------------|--------------|----------------|
| meta (этот репо) | `my_translator_meta` | `.` | `main` |
| `app` | `my_translator_app` | `../my_translator_app` | `main` |

Если путь не найден — спросить пользователя, не угадывать.

## Safety (жёстко)

- Не менять `git config`.
- Не `--force` / `--force-with-lease` на `main`/`master` без явной просьбы.
- Не `--amend`, rebase `-i`, `reset --hard`, `--no-verify` без явной просьбы.
- Не коммитить секреты: `.env*`, `credentials.json`, ключи, токены. Если пользователь просит — предупредить и исключить.
- **Push обязателен** после успешного commit в каждом dirty-репо (и для репо, где уже есть unpushed commits на `main`/`master`). Не спрашивать отдельно про push.
- Пустой коммит не создавать: репо без изменений и без unpushed commits пропускать.

## Workflow

Копируй чеклист и отмечай прогресс:

```
Commit Progress:
- [ ] 1. Resolve paths
- [ ] 2. Status scan (meta + app)
- [ ] 3. Branch check (main/master)
- [ ] 4. Draft messages + confirm if needed
- [ ] 5. Stage → commit (per dirty repo)
- [ ] 6. Push (always)
- [ ] 7. Final status report
```

### 1. Resolve paths

Из корня meta прочитать projects-map (+ local YAML). Убедиться, что каждый корень — git work tree (`git rev-parse --is-inside-work-tree`).

### 2. Status scan (параллельно по двум репо)

В **каждом** репо выполнить:

```bash
git status -sb
git status --porcelain
git diff
git diff --cached
git log -5 --oneline
```

Собрать таблицу:

| Repo | Branch | Dirty? | Ahead/behind | Summary of changes |
|------|--------|--------|--------------|--------------------|

Показать пользователю кратко **до** stage/commit. Если оба чистые — остановиться: «нечего коммитить».

### 3. Branch check

Для каждого dirty-репо:

1. `git branch --show-current` → должна быть `main` или `master` (или tracking `origin/main` / `origin/master`).
2. Если ветка другая — **не** коммитить молча: спросить.
3. Meta: допускается первый коммит на пустом `main`.

### 4. Commit messages

- Если пользователь дал одно общее сообщение — использовать его во всех dirty-репо (допустимо префикс `app:` / `meta:`).
- Если сообщения нет — для **каждого** dirty-репо своё по diff (1–2 предложения, **why**).
- Стиль: как в недавнем `git log` репо; по умолчанию короткое английское или русское описание.

Если запрос неоднозначен (какие файлы, чужие WIP, секреты) — спросить перед шагом 5. Иначе сразу 5 → 6.

### 5. Stage → commit (по каждому dirty-репо)

Последовательно:

```bash
git add -A
git commit -m "$(cat <<'EOF'
<message>

EOF
)"
```

- Hook reject → исправить и **новый** commit (не amend).
- Не стейджить ignored-секреты через `git add -f`.

### 6. Push (всегда)

```bash
git push -u origin HEAD
```

Также push для clean+ahead. Auth/rejected — показать вывод, не force.

### 7. Final report

```
| Repo | Result | Branch | Commit | Push |
|------|--------|--------|--------|------|
| meta | committed / clean / skipped | … | <sha or —> | ok / skipped / fail |
| app | … | … | … | … |
```

## Примеры триггеров

- «Закоммить» / commit skill
- «Проверь незакоммиченные в meta/app и закоммить»
- Фаза commit из `implement-change` / `end-implement-change`

## Не делать

- Не создавать PR/MR в этом скилле.
- Не мержить ветки и не переключать branch без согласия.
- Сканировать оба репо; коммитить только dirty.
