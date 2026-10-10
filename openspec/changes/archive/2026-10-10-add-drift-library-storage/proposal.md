## Why

Пользователю нужна локальная библиотека субтитров с деревом папок и личным словарём, работающая на Android, iOS и Web. Сейчас приложение — scaffold без persistence; канон «SQLite в Rust» не закрывает Web без отдельного backend’а.

## What Changes

- Локальное хранение пользовательских данных через Drift (Dart) на всех целевых платформах, включая Web.
- Библиотека: вложенные папки и импорт файлов субтитров внутрь папок.
- Личный словарь (CRUD) для проверки persistence и дальнейшего reader/highlight.
- **BREAKING** (канон): persistence переносится с «SQLite в Rust» на Drift; Rust остаётся compute-only (parse / lemmatize / translate — вне этого change или позже).
- Обновление канона в meta skills и sibling `AGENTS.md`, чтобы слои не конфликтовали.

## Scope

Capability ID:

- `library/folders` — дерево папок (создание, вложенность, удаление).
- `library/documents` — документы субтитров в папках (импорт с устройства, список, удаление).
- `dictionary/entries` — записи личного словаря (добавление, список, удаление, lookup по лемме).

Слои:

- Flutter `lib/`: Drift schema/DAO, инициализация БД, shell **Library (home) + Vocab** (замена scaffold demo), file picker import, минимальные UI states (loading/empty/error).
- Meta docs/skills + sibling `AGENTS.md`: канон persistence = Drift; Rust без владения пользовательским SQLite.
- **Rust `rust/` в этом change — no-op:** не добавлять FRB storage API, не трогать `Cargo.toml` ради persistence, не вводить `rusqlite` / `rust/src/db`. FRB-контракт для хранения не нужен.
- Mock / Visual Spec: нет (не pixel-faithful).

## Out of scope

- Парсинг SRT/VTT/ASS в Reader, токенизация, лемматизация, offline/AI translate.
- Экран Reader / split view / открытие документа в Reader (tap = no-op / later).
- Подсветка лемм в Reader и Settings tab (shell только Library + Vocab).
- Синхронизация между устройствами / облако.
- Видеоплеер и sync субтитров с видео.
- Произвольные (не-subtitle) файлы в библиотеке.
- Visual Spec / pixel-faithful mock.
- Любые изменения sibling `rust/` (включая FRB storage surface и `rusqlite`).

## Capabilities

### New Capabilities

- `library/folders`: дерево папок для группировки документов библиотеки.
- `library/documents`: импорт и хранение метаданных/содержимого subtitle-документов внутри папок.
- `dictionary/entries`: локальный личный словарь (lemma-keyed) для проверки хранения и будущего highlight.

### Modified Capabilities

- (нет — main specs пока пусты)

## Impact

- Sibling `my_translator_app` **Flutter**: Drift (+ sqlite3/web), `file_picker` / `path_provider`, слой данных в `lib/`, shell Library+Vocab вместо demo home, Web content в Drift.
- Sibling `my_translator_app` **Rust**: без изменений в этом change (compute later в других changes).
- Meta: правки `AGENTS.md` app и затронутых client/server skills (persistence target = Drift).
- Пользовательские данные живут on-device / in-browser через Drift; тяжёлая NLP-логика по-прежнему планируется в Rust, но без локальной SQLite-истины там.

## References

- Sibling [`../my_translator_app/AGENTS.md`](../../../../my_translator_app/AGENTS.md) (продукт Translation Reader, target UX library / vocab).
- [`docs/projects-map.md`](../../../docs/projects-map.md), [`openspec/config.yaml`](../../config.yaml).
- Delta specs: [`library/folders`](specs/library/folders/spec.md), [`library/documents`](specs/library/documents/spec.md), [`dictionary/entries`](specs/dictionary/entries/spec.md).
- Client skills: `work-with-pages` (`upload.md`, `vocab.md`); server skills: persistence-канон подлежит обновлению в этом change (`tasks.md` §1).
