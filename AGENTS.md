# AGENTS.md — my_translator_meta

Метарепозиторий экосистемы **My Translator** (продукт: **Translation Reader**): каноническая документация и skills для AI-агентов.  
**Runtime-код здесь не живёт** — исходники в sibling [`../my_translator_app`](../my_translator_app) (Flutter `lib/` + Rust `rust/` + FRB).

Подробный индекс docs и skills: [`.agents/AGENTS.md`](.agents/AGENTS.md).

## Обязательно перед работой

1. Из корня meta: [`docs/projects-map.md`](docs/projects-map.md) — таблица «Пути для OpenSpec / агентов»; при наличии — смержить [`projects-map.local.yaml`](projects-map.local.yaml) (local перекрывает канон). Шаблон: [`projects-map.local.yaml.example`](projects-map.local.yaml.example).
2. Если работаешь в sibling `my_translator_app` — сначала локальный [`project-map.md`](../my_translator_app/project-map.md) (ключ `my_translator_meta` → `..`), затем `my_translator_meta/docs/projects-map.md` (+ local YAML в meta).
3. Путь к коду — sibling через projects-map (+ local):
   - App: `../my_translator_app` (Flutter UI + Rust core + FRB)
4. Канон документации: [`docs/README.md`](docs/README.md). В child markdown — ссылки вида `my_translator_meta/docs/...` (резолв через `project-map.md`).
5. **Контекст пакета:** перед правками runtime читай [`../my_translator_app/AGENTS.md`](../my_translator_app/AGENTS.md).
6. **Перед выбором скилла:** skills из `my_translator_meta/.agents/skills/` — [`client/`](.agents/skills/client/) для Flutter `lib/`, [`server/`](.agents/skills/server/) для Rust `rust/`, плюс OpenSpec/оркестраторы в [`.agents/skills/`](.agents/skills/). Локальных канонических `.agents/skills/` в app нет.

## OpenSpec

- Конфиг: [`openspec/config.yaml`](openspec/config.yaml).
- Capability ID — продуктовый путь (`reader/cues`, `dictionary/entries`, `parse/subtitles`, …), не номер тикета и не имя change.
- **Запрещено** класть в `openspec/specs/` номер задачи или имя change как capability.
- Артефакты OpenSpec создаются и архивируются **в my_translator_meta**, не в `my_translator_app`.
- Пути к runtime при apply — через [`docs/projects-map.md`](docs/projects-map.md) (+ local YAML).

## Работа с OpenSpec

- **task** — schema по умолчанию `spec-driven` (proposal → specs → design → tasks).
- Типичный Cursor chat workflow: `/opsx-explore` → `prepare-changes` (если scope широкий) → `/opsx-propose` → `align-change` (опционально) → review → `/opsx-apply` / `implement-change` → `/opsx-sync` → `/opsx-archive`.

## Работа с Git

- Runtime-код коммитить в `my_translator_app`; docs/OpenSpec/skills meta — в `my_translator_meta`.
- Если начинаем новую задачу, но есть локальные изменения — спроси, что с ними делать.
- Не пушить и не создавать PR/MR без явной просьбы пользователя.

## Skills (discovery)

- OpenSpec / workspace (этот репозиторий): `.agents/skills/` (`openspec-*`, [`align-code`](.agents/skills/align-code/SKILL.md), [`align-implementability-back`](.agents/skills/align-implementability-back/SKILL.md), [`align-implementability-front`](.agents/skills/align-implementability-front/SKILL.md), [`align-meta`](.agents/skills/align-meta/SKILL.md), [`prepare-mock`](.agents/skills/prepare-mock/SKILL.md), [`verify-mock`](.agents/skills/verify-mock/SKILL.md), [`check-changes`](.agents/skills/check-changes/SKILL.md), [`prepare-changes`](.agents/skills/prepare-changes/SKILL.md), [`align-change`](.agents/skills/align-change/SKILL.md), [`commit`](.agents/skills/commit/SKILL.md), [`implement-change`](.agents/skills/implement-change/SKILL.md), [`end-implement-change`](.agents/skills/end-implement-change/SKILL.md))
- Client-wide (meta): [`.agents/skills/client/`](.agents/skills/client/) — Flutter UI: `../my_translator_app/lib/`
- Server-wide (meta): [`.agents/skills/server/`](.agents/skills/server/) — Rust core: `../my_translator_app/rust/`

Детали и перечень: [`.agents/AGENTS.md`](.agents/AGENTS.md).

## Сборка и команды

Команды `flutter` / `cargo` / FRB codegen запускает **агент** из каталога sibling `my_translator_app` (для Rust — из `rust/` или с `-C rust`). Не ждать подтверждения пользователя; при падении — починить до завершения задачи.

Типичные команды (из корня app, если не указано иное):

- Dart/Flutter: `flutter pub get` / `flutter analyze` / `flutter test` / `flutter run` / `flutter build apk|ipa|web`
- Rust: `cargo check` / `cargo test` в `rust/`
- FRB: `flutter_rust_bridge_codegen generate` (CLI ≈ 2.13)

Скилл [`commit`](.agents/skills/commit/SKILL.md): незакоммиченные изменения в meta + app → stage → commit на `main`/`master` → **всегда push** (запуск скилла = разрешение на push).
