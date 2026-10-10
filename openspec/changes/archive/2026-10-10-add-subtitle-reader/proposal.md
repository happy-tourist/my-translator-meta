## Why

Библиотека уже позволяет импортировать субтитры в папку, но tap по документу не открывает чтение («Reader later»). Без парсинга в cues и экрана Reader продукт не выполняет основной сценарий Translation Reader — просмотр субтитров с таймкодами.

## What Changes

- Парсинг содержимого импортированного файла (SRT, WebVTT, ASS/SSA) в упорядоченные реплики с временем начала/конца и текстом.
- Открытие документа из папки: при успехе — экран Reader со скроллом реплик (время + текст); при ошибке парсинга, нераспознанном формате, проблеме кодировки или нуле реплик — Reader не открывается, ошибка сразу видна в Library.
- Снятие ограничения «tap не открывает Reader» у документов библиотеки.

## Scope

- **Capability ID:** `parse/subtitles` (новое) — compute-парсинг субтитров через FRB; lossy-разбор пользовательских файлов; hard-fail при нераспознанном формате / encoding / 0 cues.
- **Capability ID:** `reader/cues` (новое) — экран чтения: список реплик со временем start→end и текстом, полный скролл документа.
- **Capability ID:** `library/documents` (изменяемое) — tap по документу инициирует open/parse flow вместо «Reader later».
- **Слои:** Rust (parse compute + тонкий FRB) и Flutter (Library orchestration → Reader UI); байты документа по-прежнему из Drift / sandbox.
- **Форматы в scope:** SRT, WebVTT, ASS/SSA (как уже принимает import).

## Out of scope

- Тап по слову / выделение фразы, карточка перевода, словарь из Reader.
- Подсветка лемм, split view, sync scroll оригинал↔перевод.
- Постраничная / Stream-загрузка cues (полный список + scroll).
- Валидация содержимого при import (ошибка только при открытии).
- Видеоплеер / синхронизация с медиа.
- Офлайн-перевод и опциональный AI.

## Capabilities

### New Capabilities

- `parse/subtitles`: парсинг байтов/текста субтитров в cues (время + текст); ошибки формата/encoding/пустоты.
- `reader/cues`: отображение cues в Reader со скроллом и таймкодами; навигация из Library при успешном parse.

### Modified Capabilities

- `library/documents`: требование «Document row does not open Reader yet» заменяется на open-flow с ошибкой в Library без перехода в Reader при fail.

## Impact

- Rust: первый продуктовый compute (parse) + тонкий FRB API (`api/parse.rs`; крейт/toolchain — в `design.md`).
- Flutter: Library open-flow → Reader; load bytes → FRB → UI (без смены Drift schema); placeholder «Reader later» снят.
- Main specs (после sync этого change): новые `parse/subtitles` и `reader/cues`; обновлённый `library/documents`.

## References

- Sibling app: `../my_translator_app/AGENTS.md` (Current vs target: Reader / parse).
- Meta: `docs/projects-map.md`; skills `work-with-parse`, `work-with-reader`, `work-with-pages` (upload.md / reader.md).
- Explore decisions: fasrt; SRT+VTT+ASS; Reader с временем; lossy; fail → не открывать; empty = error.
