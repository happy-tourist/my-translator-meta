## Why

В EPUB book-режиме при Show split список абзацев оригинала пересоздаётся и сбрасывается к началу главы, а низ открывается с абзаца 0 той же главы — хотя пользователь уже промотал середину и ждёт одноразового выравнивания обеих панелей к этой позиции (как one-shot по времени у субтитров).

## What Changes

- При Show split (и при load EPUB-перевода при уже включённом split) one-shot выравнивает **главу и абзац** нижней панели к якорю верхней (середина главы → середина).
- При Show / Hide split позиция прокрутки абзацев **оригинала** внутри главы сохраняется (не сброс к началу главы из‑за remount).
- После one-shot панели по-прежнему независимы: дальнейший скролл абзацев и pager не двигают другую панель.

## Scope

- **Capability ID:** `reader/split` — EPUB one-shot: глава + индекс абзаца (+ alignment) при Show / load перевода; preserve scroll оригинала на Show/Hide в book-режиме; без realtime mirror абзацев.
- **Слои:** Flutter UI (Reader book split); Rust/FRB **без** изменений.
- Субтитры: существующий one-shot по `startMs` и preserve cue-scroll **не ломать** (регрессия вне цели, но поведение сохранить).

## Out of scope

- Realtime sync скролла абзацев или pager между панелями.
- Persist позиции абзаца внутри главы между сессиями (resume по-прежнему chapter index — `library/documents` / `reader/chapters`).
- Persist split on/off / divider.
- Изменения parse EPUB, FRB, vocabulary / tap / highlight / video-frames / offline translate.
- Новая семантика pairing глав/абзацев в Rust.

## Capabilities

### New Capabilities

_(нет)_

### Modified Capabilities

- `reader/split`: EPUB one-shot при Show / load перевода включает выравнивание абзаца внутри главы; Show/Hide MUST сохранять mid-chapter scroll оригинала; после one-shot абзацы независимы (как уже для cue scroll / chapter pager).

## Impact

- Flutter: Reader book-режим — capture/restore скролла оригинала при toggle split; one-shot jump перевода к тому же (clamp) индексу абзаца; widget-тесты на новые сценарии.
- Rust/FRB: нет — достаточно существующих `EpubChapterDto.paragraphs` (индексный clamp в UI).
- Main specs (после sync): `reader/split` (Purpose; body требования «Synchronize EPUB panes by chapter index» — one-shot = глава+абзац; preserve mid-chapter на Show/Hide; добавить RS-21/RS-22; уточнить RS-10/11/19/20).

## References

- Sibling app: `../my_translator_app/AGENTS.md`.
- Meta: `docs/projects-map.md`; skills `work-with-reader` (split-scroll, chapters), `work-with-pages`.
- Explore (эта сессия): D1 = one-shot глава + абзац (B); continuous sync не возвращать; unequal paragraph counts → clamp.
