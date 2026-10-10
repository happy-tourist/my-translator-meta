---
name: implement-change
description: >-
  Orchestrates full implementation of an active OpenSpec change: apply task
  blocks via subagents, then align (постановка), verify-code (compliance),
  optional verify-mock (only if a mock is attached and the change is mock-driven),
  check-changes, commit, then Web smoke with COOP/COEP. Use when the user asks
  to implement-change, fully implement an active change, or run
  apply → align → verify → [mock] → check → commit → web-smoke end-to-end
  without pausing for review.
---

# Implement Change — apply → align → verify → [mock] → check → commit → web-smoke

Оркестратор полного цикла по **активному** OpenSpec change. Parent **не** пишет runtime-код сам: делегирует субагентам.

Runtime: **один** app `../my_translator_app` (Flutter `lib/` + Rust `rust/`) + meta.

## Когда применять

- `implement-change` / полная реализация active change
- end-to-end без промежуточного review

Не подменять одиночный `openspec-apply-change`. Resume — `.implement-change-state.yaml`.

Standalone [`align-code`](../align-code/SKILL.md) (align+verify в одном прогоне) **не** меняется — в этом оркестраторе оси **разведены** на две фазы.

## Жёсткое правило прерывания

Прерывать **только** при необходимости решения человека. verify-mock без макета → `SKIPPED`, пайплайн продолжается. Web smoke после commit при сбое Chrome/headers — `HUMAN_BLOCKER` только если без человека нельзя продолжить/починить; иначе починить и повторить.

## State-файл

`openspec/changes/<name>/.implement-change-state.yaml`:

```yaml
change: <name>
phase: apply | align | verify | mock | check | commit | web-smoke | done
apply_block: "<N-or-null>"
blocker: null | "<short reason>"
updated: "<ISO-8601>"
```

- После всего apply → `phase: align`
- После align → `phase: verify`
- После verify → если gate mock → `phase: mock`, иначе `phase: check`
- После mock → `phase: check` → `commit` → `web-smoke` → `done`

## Делегируемые скиллы

| Фаза | Skill |
|------|--------|
| Apply | [`openspec-apply-change`](../openspec-apply-change/SKILL.md) |
| Align | [`client-align-code`](../client/client-align-code/SKILL.md) + [`server-align-code`](../server/server-align-code/SKILL.md) (только постановка; **не** полный `align-code`) |
| Verify | [`client-verify-code`](../client/client-verify-code/SKILL.md) + [`server-verify-code`](../server/server-verify-code/SKILL.md) (compliance / tooling) |
| Mock | [`verify-mock`](../verify-mock/SKILL.md) — если gate |
| Check | [`check-changes`](../check-changes/SKILL.md) |
| Commit | [`commit`](../commit/SKILL.md) |
| Web-smoke | [`work-with-env-deploy`](../client/work-with-env-deploy/SKILL.md) — Chrome + COOP/COEP |

Пути — [`docs/projects-map.md`](../../../docs/projects-map.md).

## Выбор change

Имя → контекст → единственный active → иначе HUMAN_BLOCKER. `Using change: <name>`.

## Gate verify-mock

G1: макет прикреплён (файл/вложение). G2: change про реализацию макета (Visual Spec / UI по скрину). Оба → mock; иначе SKIPPED → check.

## Workflow

```
Implement-Change Progress:
- [ ] 1. Select change + read apply skill
- [ ] 2. Status + apply instructions + contextFiles
- [ ] 3. Apply: каждый ##-блок tasks.md → отдельный субагент
- [ ] 4. Align → субагент (только *-align-code) (+ «Добавь и поправь…»)
- [ ] 5. Verify → субагент (только *-verify-code) (+ «Добавь и поправь…»)
- [ ] 6. Verify-mock → если gate; иначе SKIPPED
- [ ] 7. Check-changes → субагент (+ правка)
- [ ] 8. Commit → субагент (meta + app)
- [ ] 9. Web-smoke → Chrome с COOP/COEP (после commit)
- [ ] 10. Short final report
```

### Apply blocks

Блок = `## N. …` в tasks.md. Субагент: следовать apply; только этот блок; `- [x]` сразу; не спрашивать; `HUMAN_BLOCKER` или `OK`. Последовательно (часто rust → flutter → bridge). Parent не пишет продукт-код.

**Web mid-apply:** если блок/design требует проверки Web (Drift Web, FRB WASM, task вроде «flutter run -d chrome») — субагент **сам** запускает Chrome с COOP/COEP из `work-with-env-deploy` (не голый `flutter run -d chrome`).

### Align (фаза 4)

Отдельный субагент:

1. Выполнить **только** `client-align-code` + `server-align-code` (постановка / FRB races / overlay) — **не** вызывать полный `align-code` и **не** гонять `*-verify-code` здесь.
2. Затем дословно: **«Добавь и поправь что считаешь нужным»**.
3. Вернуть findings + что поправлено, или `HUMAN_BLOCKER`.

### Verify (фаза 5)

Отдельный субагент **после** align:

1. Выполнить **только** `client-verify-code` + `server-verify-code` (`flutter analyze` / `flutter test`, `cargo check` / `cargo test` / clippy, conventions).
2. Затем дословно: **«Добавь и поправь что считаешь нужным»**.
3. При необходимости Web-проверки tooling — Chrome только с COOP/COEP.
4. Вернуть violations/warnings + что поправлено, или `HUMAN_BLOCKER`.

### Mock / Check

После отчёта — дословно **«Добавь и поправь что считаешь нужным»**. Mock: абсолютные пути + `file_attachments`; правки UI в `lib/`.

### Commit

[`commit`](../commit/SKILL.md): stage → commit → push **meta + app**.

### Web-smoke (фаза 9, после commit)

Parent или субагент из корня app:

```bash
flutter run -d chrome \
  --web-header=Cross-Origin-Opener-Policy=same-origin \
  --web-header=Cross-Origin-Embedder-Policy=require-corp
```

Критерий OK: приложение стартует, `RustLib.init` + Drift open без uncaught exception (как smoke task 2.4). Оставить процессrunning для ручной проверки пользователя, если это финал сессии; иначе зафиксировать успех старта в отчёте.

Не запускать web-smoke без COOP/COEP.

### Final report

```markdown
# Implement Change — итог
**Change:** <name>
**Apply:** N/M …
**Align:** OK | HUMAN_BLOCKER | skipped
**Verify:** OK | HUMAN_BLOCKER | skipped
**Verify-mock:** OK | SKIPPED (…) | HUMAN_BLOCKER
**Check-changes:** …
**Commit:** …
**Web-smoke:** OK | HUMAN_BLOCKER | skipped
**Stopped:** нет | <причина>
**Resume:** снова `implement-change`
```

## Субагенты

Новый субагент на фазу; `run_in_background: false` (кроме финального web-smoke, который MAY оставить `flutter run` в фоне для пользователя); контекст: change, пути meta/app (`lib/`, `rust/`).

## Не делать

- Не сливать align и verify в один субагент внутри этого оркестратора.
- Не пропускать align / verify / check / commit / web-smoke.
- Не verify-mock без gate.
- Не archive/PR в этом скилле.
- Не ослаблять safety commit.
- Не `flutter run -d chrome` без COOP/COEP headers.
