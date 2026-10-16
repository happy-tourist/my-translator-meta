# Индекс агента — my_translator_meta

Канонический каталог документации и skills для AI-агентов в метарепозитории **My Translator** (продукт: **Translation Reader**).

Краткие always-on инструкции (Cursor): [`AGENTS.md`](../AGENTS.md) в корне репозитория.

Runtime-код: один sibling [`../my_translator_app`](../my_translator_app) — Flutter `lib/` + Rust `rust/` + FRB (in-process).  
Контекст пакета:

| Пакет | AGENTS.md |
|-------|-----------|
| App (Flutter + Rust) | [`../my_translator_app/AGENTS.md`](../my_translator_app/AGENTS.md) |

---

## Документация

| Документ | Назначение |
|----------|------------|
| [`docs/projects-map.md`](../docs/projects-map.md) | Ключ `app` → sibling-путь; local YAML; Child `project-map.md` (ключ `my_translator_meta`) |
| [`docs/README.md`](../docs/README.md) | Оглавление документации |

Локальные оверрайды путей: [`projects-map.local.yaml`](../projects-map.local.yaml) / шаблон [`projects-map.local.yaml.example`](../projects-map.local.yaml.example) в корне meta.

OpenSpec: [`openspec/config.yaml`](../openspec/config.yaml); артефакты в `openspec/changes/` и `openspec/specs/`.

---

## Skills

### OpenSpec / workspace (`.agents/skills/`)

| Skill | Когда |
|-------|--------|
| [`align-code`](skills/align-code/SKILL.md) | Flutter + Rust: `client-*` + `server-*` align/verify → сводный отчёт (FRB races) |
| [`align-implementability-back`](skills/align-implementability-back/SKILL.md) | Meta Rust/API claims vs `my_translator_app/rust` (реализуемость); read-only |
| [`align-implementability-front`](skills/align-implementability-front/SKILL.md) | Meta Flutter/UI claims vs `lib/` (+ полнота Visual Spec при mock-driven); read-only |
| [`align-meta`](skills/align-meta/SKILL.md) | Качество созданных/изменённых meta-артефактов (OpenSpec/docs/skills/indexes); read-only |
| [`verify-mock`](skills/verify-mock/SKILL.md) | Flutter UI vs макет + **строгие размеры и цвета (hex/rgba)** Visual Spec → Violations / Warnings / Recommendations |
| [`prepare-mock`](skills/prepare-mock/SKILL.md) | Снять с макета размеры/отступы/**hex·rgba цвета**/copy/dividers → Visual Spec в design (+ specs); до propose/apply |
| [`commit`](skills/commit/SKILL.md) | Status → stage → commit → **push** по meta + app (main/master) |
| [`check-changes`](skills/check-changes/SKILL.md) | Unstaged app (`lib/` / `rust/`) → add/change/delete/**split** skills + maps/docs/AGENTS/`openspec/config.yaml` |
| [`prepare-changes`](skills/prepare-changes/SKILL.md) | До propose: чаще всего **один** change (норма); дробить только по нужде; пункты работ + проверяемый результат |
| [`align-change`](skills/align-change/SKILL.md) | Active change: align-back → align-front → align-meta (субагенты); resume по state; без propose/apply |
| [`implement-change`](skills/implement-change/SKILL.md) | Активный change: apply → **align** → **verify** → verify-mock (**если** макет) → check-changes → commit → **web-smoke** (Chrome `--web-port=5555` + COOP/COEP); resume — state |
| [`end-implement-change`](skills/end-implement-change/SKILL.md) | Закрытие change: update → sync-specs → archive → commit (без вопросов) |
| [`openspec-propose`](skills/openspec-propose/SKILL.md) | Propose: proposal → specs → design → tasks; при ровно одном active change — всегда дописывать в него (новый каталог только по явной просьбе) |
| [`openspec-new-change`](skills/openspec-new-change/SKILL.md) | Создать change и начать артефакты по схеме |
| [`openspec-continue-change`](skills/openspec-continue-change/SKILL.md) | Продолжить создание недостающих артефактов change |
| [`openspec-update-change`](skills/openspec-update-change/SKILL.md) | Правка существующих артефактов change без кода |
| [`openspec-apply-change`](skills/openspec-apply-change/SKILL.md) | Реализация по `tasks.md` |
| [`openspec-verify-change`](skills/openspec-verify-change/SKILL.md) | Проверка реализации против артефактов |
| [`openspec-archive-change`](skills/openspec-archive-change/SKILL.md) | Архивация закрытого change |
| [`openspec-explore`](skills/openspec-explore/SKILL.md) | Explore mode до/во время change; явно флагает «⛔ нужно решение разработчика» если фича ещё не implementable |
| [`openspec-sync-specs`](skills/openspec-sync-specs/SKILL.md) | Delta → main specs без archive |

### Discovery (другие корни)

| Корень | Назначение |
|--------|------------|
| [`.agents/skills/client/`](skills/client/) | Client-wide skills (meta); runtime Flutter: `../my_translator_app/lib/` |
| [`.agents/skills/server/`](skills/server/) | Server-wide skills (meta); runtime Rust: `../my_translator_app/rust/` |

#### Client / Flutter (`.agents/skills/client/`)

Стек: Flutter / Dart (`lib/`), Material, FRB 2.13 → generated `lib/src/rust/`, тяжёлая логика в Rust. Runtime: `../my_translator_app/`.

| Skill | Когда |
|-------|--------|
| [`client-align-code`](skills/client/client-align-code/SKILL.md) | Audit диффа / UX vs OpenSpec; FRB races; overlay chrome |
| [`client-locate-change-points`](skills/client/client-locate-change-points/SKILL.md) | Где править в `lib/` (и когда fix в `rust/src/api/`) |
| [`client-verify-code`](skills/client/client-verify-code/SKILL.md) | Compliance + DRY/KISS; `flutter analyze` / conventions |
| [`client-work-with-structure`](skills/client/client-work-with-structure/SKILL.md) | Экраны / виджеты / слои / feature folders |
| [`client-work-with-errors`](skills/client/client-work-with-errors/SKILL.md) | Ошибки UI / вызовы Rust / `RustLib.init` |
| [`work-with-pages`](skills/client/work-with-pages/SKILL.md) | Reader (cues / EPUB book-mode) / словарь / Library (root + folders) / настройки (+ shell) |
| [`work-with-drift`](skills/client/work-with-drift/SKILL.md) | Drift user persist: schema 7+ (nullable `folderId` = root), repos (`listVisible`), content store (Web UTF-8 / EPUB base64), format-gated ReadingAnchor |
| [`work-with-styles`](skills/client/work-with-styles/SKILL.md) | Тема, Material, lemma highlight tokens |
| [`work-with-localization`](skills/client/work-with-localization/SKILL.md) | Локаль UI (ARB) vs learning language pair |
| [`work-with-env-deploy`](skills/client/work-with-env-deploy/SKILL.md) | `flutter build` apk/ipa/web; run/devices; Web `--web-port=5555` + COOP/COEP |
| [`client-work-with-test`](skills/client/client-work-with-test/SKILL.md) | `flutter test` / `integration_test` / Drift tests |
| [`work-with-frb`](skills/client/work-with-frb/SKILL.md) | Generated API, `RustLib.init`, codegen |
| [`work-with-reader`](skills/client/work-with-reader/SKILL.md) | Contiguous тап→CTA/sheet, lemma/phrase highlight/edit, split one-shot (time / EPUB chapter; no settle realtime), book-mode ([chapters.md](skills/client/work-with-reader/chapters.md)), paged video frames ([video-frames.md](skills/client/work-with-reader/video-frames.md)) |

#### Server / Rust (`.agents/skills/server/`)

Стек: Rust edition 2021, crate `rust_lib_my_translator_app` (`rust/`), FRB 2.13 (`rust/src/api/` → generated). Runtime: тот же `../my_translator_app/` (имя «server» в skills = Rust core, не отдельный репозиторий).

| Skill | Когда |
|-------|--------|
| [`server-align-code`](skills/server/server-align-code/SKILL.md) | Audit диффа `rust/` vs OpenSpec; FRB races (init / stream / blocking FFI) |
| [`server-locate-change-points`](skills/server/server-locate-change-points/SKILL.md) | Где править в `rust/` (`api/` vs parse/nlp/dict/translate/db) |
| [`server-verify-code`](skills/server/server-verify-code/SKILL.md) | Compliance + DRY/KISS; `cargo check` / `cargo test` (+ [built-in.md](skills/server/server-verify-code/built-in.md)) |
| [`server-work-with-structure`](skills/server/server-work-with-structure/SKILL.md) | Тонкий `api/` vs внутренние модули (+ [content.md](skills/server/server-work-with-structure/content.md)) |
| [`server-work-with-errors`](skills/server/server-work-with-errors/SKILL.md) | `Result` / ошибки через FRB (не UI) |
| [`server-work-with-test`](skills/server/server-work-with-test/SKILL.md) | `cargo test`; topics [unit.md](skills/server/server-work-with-test/unit.md), [fixtures.md](skills/server/server-work-with-test/fixtures.md) |
| [`work-with-config`](skills/server/work-with-config/SKILL.md) | `Cargo.toml`, `flutter_rust_bridge.yaml`, pin FRB ≈ 2.13 |
| [`work-with-frb-api`](skills/server/work-with-frb-api/SKILL.md) | Публичный API в `rust/src/api/` + codegen (Rust-сторона) |
| [`work-with-parse`](skills/server/work-with-parse/SKILL.md) | SRT / VTT / ASS(SSA); EPUB ([epub.md](skills/server/work-with-parse/epub.md)) |
| [`work-with-tokenizematize`](skills/server/work-with-tokenizematize/SKILL.md) | Tokenize (`panko`) / WordNet lemmatize / morph+phrase mark / spans |
| [`work-with-dictionary`](skills/server/work-with-dictionary/SKILL.md) | Lemma keys / NLP handoff; persist CRUD = Drift (Flutter) |
| [`work-with-translate`](skills/server/work-with-translate/SKILL.md) | Офлайн-перевод: native ORT + Web Transformers.js facade |

Runtime-пути в client/server skills (`lib/…`, `rust/…`, `test/…`) — относительно корня `my_translator_app`.

---

## OpenSpec / workflow

- Конфиг: [`openspec/config.yaml`](../openspec/config.yaml) (`projectsMap`, `context`, capability rules).
- Capability ID = продуктовый путь (`reader/cues`, `dictionary/entries`, `parse/subtitles`, `translate/offline`, …), не имя change.
- Delta: `openspec/changes/<slug>/specs/<capability-id>/spec.md`
- Main: `openspec/specs/<capability-id>/spec.md`
- Типичный Cursor chat workflow: `/opsx-explore` → [`prepare-changes`](skills/prepare-changes/SKILL.md) (если scope широкий) → `/opsx-propose` → [`align-change`](skills/align-change/SKILL.md) (опционально) → review → `/opsx-apply` / [`implement-change`](skills/implement-change/SKILL.md) → `/opsx-sync` → `/opsx-archive`.
- Артефакты живут в **my_translator_meta**, не в app. Пути к runtime при apply — через [`docs/projects-map.md`](../docs/projects-map.md) (+ local YAML).
- Always-on: [`AGENTS.md`](../AGENTS.md).
