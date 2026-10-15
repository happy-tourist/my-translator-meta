## Why

Translation Reader умеет изучать слова только в субтитрах. Нужна та же петля обучения (чтение → тап/словарь → перевод → split) для EPUB-книг: импорт `.epub`, чтение по главам со стрелками, синхронный переводной EPUB в нижней панели — на native и Web.

## What Changes

- Library принимает `.epub` наряду с субтитрами; open парсит книгу в главы/абзацы.
- Reader в book-режиме: внутри главы — скролл абзацев (как список cues); между главами — pager `[<] n/N [>]` снизу.
- Learning-поверхность как у субтитров: tap / highlight / vocab / offline translate на тексте абзацев.
- Split: для EPUB-оригинала перевод — только `.epub`; синхронизация панелей по **индексу главы** (готовые переводы с одинаковой структурой глав).
- Картинки, сноски и video-frames для EPUB — не входят.

## Capabilities

### New Capabilities

- `parse/epub`: разбор EPUB (bytes) в упорядоченные главы и plain-text абзацы для Reader (compute в Rust через FRB).
- `reader/chapters`: book-режим Reader — список абзацев главы, нижний chapter pager, learning-жесты как у cues; без timed cue list и без video.

### Modified Capabilities

- `library/documents`: import/open `.epub`; persist bytes; open-flow через EPUB-parse; linked translation только EPUB↔EPUB.
- `reader/split`: для book-режима — load/delete перевода как EPUB; sync панелей по индексу главы (не по `startMs`).

## Scope

- **Capability ID:** `parse/epub`, `reader/chapters`, `library/documents`, `reader/split`.
- **Слои:** Flutter `lib/` (Library + Reader book-mode + Drift) и Rust `rust/` (EPUB parse + thin FRB); FRB-контракт `parse_epub(bytes) → ParsedEpub` / `EpubParseErrorKind` (design D2).
- Поведение TOC: вложенные пункты с контентом = страницы пейджера; без TOC → одна глава на всю книгу.
- Платформы: native и Web (один bytes-контракт).

## Out of scope

- Картинки, сноски, rich HTML/CSS в теле главы.
- Video attachment / `reader/video-frames` для EPUB-документов.
- Смешанный split EPUB↔SRT/VTT/ASS.
- Облачный sync библиотек; отдельный «магазин» книг.
- Редактирование/запись EPUB; выбор альтернативного EPUB-парсера после фиксации в design.

## Impact

- Library allowlist форматов и open-flow (subtitle vs book); Web EPUB persist binary-safe (base64 / Blob — design D6), не UTF-8 text roundtrip.
- Новый FRB surface EPUB parse; Reader book-mode session рядом с cue/video путями (chapter payload, без Load video).
- Reading anchor format-gated: chapter index в `lastReadStartMs` для epub.
- Split sync ветка: time (субтитры) vs chapter index (EPUB); translation load через `parse_epub`.
- Main specs после sync: `parse/epub`, `reader/chapters`, обновления `library/documents`, `reader/split`.

## References

- Sibling app: `../my_translator_app/AGENTS.md`.
- Meta: `docs/projects-map.md`; skills `work-with-parse`, `work-with-reader`, `work-with-pages` / upload, `work-with-drift`, `work-with-frb` / `work-with-frb-api`.
- Explore (2026-10-15): rbook; главы + абзацы; split EPUB↔EPUB; без картинок/сносок.
