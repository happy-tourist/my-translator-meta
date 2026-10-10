## 0. Rust boundary (no-op)

- [x] 0.1 Подтвердить design **D0**: в этом change **нет** правок `../my_translator_app/rust/**`, нет `rusqlite`/`diesel`, нет FRB storage API — apply не создаёт `rust/src/db` и не меняет `Cargo.toml` ради persistence; scaffold `process_text` / `add_numbers` в Rust без изменений (UI demo MAY быть удалён)

## 1. Canon: Drift persistence (meta + app AGENTS)

- [x] 1.1 Прочитать `proposal.md`, `design.md` (D0/D1/D7) и текущий `../my_translator_app/AGENTS.md` (Core Stack / Data); обновить AGENTS: user persistence = Drift, Rust без user SQLite; при необходимости добавить домены `library/folders` / `library/documents` рядом с dictionary — проверить, что в файле больше нет утверждения «SQLite через Rust» как source of truth
- [x] 1.2 Проверить `openspec/config.yaml` (context + rules): Drift / `library/*` / Rust compute-only уже выровнены в align; `rg -n "SQLite|rusqlite|Drift|library/" openspec/config.yaml` — при регрессии поправить
- [x] 1.3 Обновить client skills с user-data persistence (`client-work-with-structure`, `work-with-pages/upload.md`, `work-with-pages/vocab.md`, и места в locate/verify/align, где «Rust + SQLite» для словаря/субтитров): указать Drift; в `vocab.md` — empty/add через Vocab UI в этом change (Reader add — later); в `upload.md` — delete/import через Drift, не ждать Rust persistence API — `rg -n "Rust \\+ SQLite|SQLite в Rust|through Rust \\(target SQLite\\)" .agents/skills/client` не оставляет устаревший канон для user data
- [x] 1.4 Обновить server skills (`work-with-dictionary`, `work-with-config`, `server-work-with-structure`): persist user data не требует `rust/src/db` / rusqlite; NLP по-прежнему Rust — проверить формулировки в затронутых SKILL.md / content.md; явно: dictionary CRUD persistence этого change = Drift, не FRB

## 2. Drift setup (Flutter app)

- [x] 2.1 В sibling app добавить зависимости по design **D1**: `drift`, `sqlite3`, `path`, `path_provider`, `file_picker`, native sqlite libs (`sqlite3_flutter_libs` или актуальный аналог), Web path по docs Drift (`drift_flutter` и/или wasm sqlite3); dev: `drift_dev`, `build_runner` — `flutter pub get` в `../my_translator_app` без ошибок
- [x] 2.2 Создать Drift database + таблицы `folders`, `documents`, `dictionary_entries` (design D3); codegen Drift успешен, типы таблиц доступны из `lib/`
- [x] 2.3 Инициализировать БД при старте (`main`: после `RustLib.init`, до или вместе с `runApp`); cold start: loading → open OK; failure → error+retry (D8); `flutter analyze` чист по затронутым файлам
- [x] 2.4 Проверить Web entry (wasm/worker/assets по выбранному Drift Web setup) — `flutter run -d chrome` открывает приложение и БД без uncaught exception при старте

## 3. Library folders + documents

- [x] 3.1 Реализовать репозиторий/DAO папок: create (root/nested), list children, cascade delete (design D3/D5) — unit-тесты покрывают SC LF-01…LF-06 из `specs/library/folders/spec.md`
- [x] 3.2 Реализовать репозиторий/DAO документов: import только в папку, reject unsupported type (`.srt`/`.vtt`/`.ass`/`.ssa` → format), list, delete; native sandbox path + Web content в Drift (design D6) — unit-тесты покрывают SC LD-01…LD-05, LD-07 (и LD-09 где тестируемо) из `specs/library/documents/spec.md`
- [x] 3.3 Минимальный UI библиотеки (`features/upload` + shell): навигация папок, create, import (`file_picker` только внутри папки), delete с confirm на cascade; states: loading / empty / error (D8); document tap **не** открывает Reader — widget-тесты LF-03, LF-05, LF-07, LF-08, LD-02…LD-04, LD-08

## 4. Dictionary entries

- [x] 4.1 Реализовать репозиторий словаря: normalize lemma (trim + case-fold), add (upsert), list, remove, lookup (design D4) — unit-тесты покрывают SC DE-01…DE-07, DE-09, DE-10 из `specs/dictionary/entries/spec.md`
- [x] 4.2 Минимальный UI словаря (`features/vocab`): empty с формой add (lemma + translation + optional context), list, delete; validation на пустые поля — widget-тесты DE-04, DE-05, DE-09, DE-11; empty state не требует Reader

## 5. Shell home + persistence + quality gates

- [x] 5.1 Тест(ы) на переживание данных после close/reopen той же Drift DB (или эквивалент) для document + dictionary — покрывают intent SC LD-06 и DE-08 (полный browser restart same-origin — вне tasks, см. design D9 / verify-ops)
- [x] 5.2 **Заменить** scaffold `MyHomePage` demo на shell Library (default) + Vocab (design D2); `MaterialApp.title` → Translation Reader; demo TextField/«Вызвать Rust» не остаётся home — `flutter analyze` в app без issues по изменённым файлам
- [x] 5.3 Прогнать `flutter test` в sibling app — все новые/затронутые тесты зелёные
