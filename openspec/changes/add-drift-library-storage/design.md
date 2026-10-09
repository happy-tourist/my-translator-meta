## Context

См. `proposal.md` — Why / Scope. Runtime сейчас scaffold (`process_text` / `add_numbers`); SQLite/rusqlite в Rust нет. Канон skills/AGENTS до этого change утверждал «пользовательские данные → SQLite в Rust»; для Web это несовместимо с `rusqlite`. Решение explore: **Drift в Flutter для всего persistence**, Rust — compute-only (parse/lemma/translate вне или после этого change).

Слои в этом change: **Flutter `lib/` + meta канон**; Rust API/FRB для хранения **не** вводятся. Mock / Visual Spec **нет** — UI минимальный, не pixel-faithful.

## Goals / Non-Goals

**Goals:**

- Единая Drift-БД для папок, документов библиотеки и словаря на Android / iOS / Web.
- Репозитории/DAO с CRUD, соответствующим specs; минимальный product shell (Library + Vocab) для проверки.
- Обновить sibling `AGENTS.md` и затронутые meta skills/docs, чтобы persistence-канон = Drift и не конфликтовал с apply/verify.
- Хранение байтов субтитров: native — sandbox FS + путь в БД; Web — содержимое в Drift (BLOB/text), без зависимости от classic file path.

**Non-Goals:**

- Реализация parse/Reader/highlight/translate в Rust в этом change.
- Введение `rusqlite` / Diesel / слоя `rust/src/db` для пользовательских данных.
- Любые правки `../my_translator_app/rust/**`, `Cargo.toml` (deps), FRB codegen ради storage.
- Облачный sync, произвольные файлы, видеоплеер.
- Settings tab, Reader screen, Visual Spec / pixel mock.

## Decisions

### D0: Граница Rust в этом change (no-op)

- **Выбор:** `rust/` **не меняется**. Нет новых FRB API, нет `rusqlite`/`diesel`, нет `rust/src/db`. Scaffold `process_text` / `add_numbers` остаётся как есть (вызовы можно убрать из UI при замене home — код API в Rust не трогать).
- **Почему:** user persistence целиком на Drift; compute (parse/lemma/translate) — out of scope этого change.
- **Проверка apply:** `git diff` / `rg` по `rust/` пуст относительно storage; `Cargo.toml` без rusqlite.
- **Не путать** с устаревшим каноном skills «словарь → SQLite в Rust»: задачи §1 обновляют skills/AGENTS; до этого apply **следует design D0–D1**, не старому skill.

### D1: Persistence = Drift (Dart), не Rust SQLite

- **Выбор:** пакеты Drift (+ platform sqlite / web) в `my_translator_app`; единственный source of truth для folders / documents / dictionary.
- **Почему:** один код для Web + native; отказ от `rusqlite` на WASM.
- **Альтернативы:** dual store (rusqlite + IndexedDB) — сложнее; только native — отвергнуто (проверка на Web).
- **Зависимости (apply — зафиксировать совместимые с sdk ^3.13 при `flutter pub add`):**
  - runtime: `drift`, `sqlite3`, `path`, `path_provider`, `file_picker`;
  - native sqlite: `sqlite3_flutter_libs` (или актуальный аналог из docs Drift);
  - Web: следовать актуальному Drift Web setup (`drift` + `sqlite3` wasm / `drift_flutter` — что рекомендует Drift на момент apply); не полагаться на filesystem path для содержимого файла;
  - dev: `drift_dev`, `build_runner`.
- **Web storage note:** БД Drift на Web живёт в browser storage (IndexedDB/OPFS через sqlite3 wasm — как настроит выбранный Drift Web path). После hard refresh / того же origin данные MUST переживать reopen (см. LD-06, DE-08). Clear site data = wipe — ожидаемо для greenfield.

### D2: Слои Flutter + home shell

```
lib/
  main.dart                 # RustLib.init + open Drift + runApp (тонкий)
  app/                      # MaterialApp + shell (Library | Vocab)
  data/                     # или features/*/data — Drift database, tables, DAOs
  features/upload/          # library UI: дерево папок, import
  features/vocab/           # словарь list/add/delete
```

- **Home (locked):** заменить scaffold `MyHomePage` demo на product shell: две primary destinations — **Library** (default) и **Vocabulary**. Settings / Reader в этом change не обязательны.
- UI тонкий; правила «документ только в папке», cascade delete, upsert леммы — в data/domain слое Dart рядом с Drift, не в виджетах.
- Generated `lib/src/rust/**` не трогать; FRB для storage не нужен.
- Строки UI: допустим hardcoded RU/EN в виджетах для минимального smoke; полный ARB/l10n — не блокер этого change (отдельный polish).

### D3: Модель данных (логическая)

- **folders:** `id`, `parent_id` (null = корень), `name`, timestamps. Запрет циклов при move/create (v1: create с явным parent; move out of scope). Имя после `trim` MUST быть непустым (LF-03); иначе validation error, без insert.
- **documents:** `id`, `folder_id` (NOT NULL), `title`, `format` (`srt` | `vtt` | `ass`), `storage_key` / path или inline `content` (Web), timestamps.
  - Расширения файлов: `.srt`, `.vtt`, `.ass`, `.ssa` → format `ass` для обоих ASS/SSA.
- **dictionary_entries:** `id`, `lemma` (UNIQUE, normalized), `translation`, `context` nullable, timestamps.

Документы **не** живут в корне: только при `folder_id`. В UI выбранной папки список = дочерние папки (`library/folders`) + документы (`library/documents`).

### D4: Политика дубликата леммы = upsert + normalize

- Перед insert/lookup: `trim` + Unicode case-fold / lowercase → **normalized lemma** как UNIQUE key.
- Пустая лемма или пустой перевод после trim → validation error (не писать в БД).
- При повторном add той же (нормализованной) леммы — обновить `translation` / `context`, сохранить один ряд.
- Альтернатива reject — хуже для «проверки словаря» из Reader later.

### D5: Каскадное удаление папки

Удаление папки удаляет поддерево folders + documents (+ файлы sandbox по `storage_key`). Реализовать транзакцией Drift; на native — удалить файлы после успешного commit (или best-effort cleanup).

**UX:** перед cascade delete UI SHALL запросить подтверждение (destructive dialog); для delete одного document — confirm желателен, MAY быть простым dismissible confirm.

### D6: Импорт файла

- `file_picker` → байты/путь → проверка расширения (D3) → запись document + сохранение байтов (native: app sandbox via `path_provider` + `storage_key`; Web: inline `content` в Drift).
- Парсинг cues в Reader — out of scope; храним сырой файл/текст для будущего parse.
- **Platform notes:**
  - Android / iOS: при необходимости добавить минимальные permission / usage strings, которых требует выбранная версия `file_picker` (не запрашивать лишние broad storage permissions).
  - Web: picker отдаёт bytes in-memory; не сохранять path как source of truth.
- Import affordance активен **только внутри выбранной папки** (не в корне списка папок) — иначе LD-02.

### D7: Обновление канона (обязательно в том же apply)

Чтобы «не конфликтовало»:

| Место | Что сделать |
|-------|-------------|
| `../my_translator_app/AGENTS.md` | Core Stack / Data: persistence = Drift; Rust без user SQLite |
| `openspec/config.yaml` context + rules (meta) | User persist = Drift; домены `library/*`; Rust layers compute-only; §1.2 — `rg` на остатки «SQLite user store» / `…/ db` |
| Client skills (`client-work-with-structure`, `upload.md`, `vocab.md`, verify/locate и т.п.) | «Rust + SQLite» → Drift для user data; NLP всё ещё Rust; vocab empty/add в этом change — in-UI add (Reader later) |
| Server skills (`work-with-dictionary`, `work-with-config`, structure) | Dictionary *domain rules* могут остаться target в Rust later для NLP; **persist** entries — Drift; не требовать `rust/src/db` для user data |

Точный список файлов — в `tasks.md`.

### D8: UI states (минимальный набор)

Без Visual Spec; достаточно явных состояний, чтобы FE не изобретал продукт:

| Surface | States |
|---------|--------|
| App start | Пока Drift open: простой loading (splash/centered progress); при ошибке open — error + retry |
| Library root / folder | loading list; empty (LF-05) с CTA «создать папку»; error snackbar/banner на failure create/delete/import |
| Folder with content | список дочерних папок + документов; actions: create folder, import, delete |
| Document row | показать title/format; **tap не открывает Reader** в этом change (no-op или snackbar «Reader later») |
| Vocab | empty с формой/CTA add (не «только из Reader»); list; validation errors на пустых полях; delete |

### D9: Тесты

- Unit-тесты DAO/репозиториев (in-memory Drift где возможно).
- Widget/smoke: создать папку → import (mock picker или fixture bytes) → add vocab → close/reopen DB (SC LD-06 / DE-08 intent в автотестах — `tasks.md` §5.1).
- Полный browser restart same-origin на Web — **не** пункт `tasks.md` (запрет manual smoke в checklist); остаётся verify/ops после apply (`flutter run -d chrome` + hard reload).

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Канон skills всё ещё говорит «SQLite в Rust» → агенты ломают слой | Задачи на обновление meta/AGENTS в том же change; verify против нового канона; apply следует D0–D1 |
| Web FS отличается от native | Documents: content в Drift на Web; один репозиторийный API с platform branch (D1/D6) |
| Большие subtitle в BLOB на Web | Для v1 ок; позже — квота / предупреждение |
| Словарь в Dart, а highlight/lemma позже в Rust | Lookup API на Dart; в будущем Flutter передаёт set лемм в Rust или матчит на UI — отдельный change |
| Scaffold demo vs library | **Locked (D2):** demo home удаляется; shell Library + Vocab |
| Устаревший vocab skill «add only from Reader» | В этом change in-UI add обязателен; §1.3 правит `vocab.md` |

## Migration Plan

- Greenfield: миграции Drift с version 1 (folders, documents, dictionary_entries).
- Откат: удалить app data / clear browser storage; schema ещё не в проде.
- После merge: `/opsx-sync` перенесёт specs в `openspec/specs/`.

## Open Questions

- Нет блокеров для Flutter implementability.
- Версии пакетов Drift / sqlite / file_picker — зафиксировать при `flutter pub add` актуальные совместимые с sdk ^3.13 (не пинить устаревшие номера в meta заранее).
- Детали Material chrome (иконки, точные labels) — на apply по `work-with-pages` / `work-with-styles`; не требуют Visual Spec.
