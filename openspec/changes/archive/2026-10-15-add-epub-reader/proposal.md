## Why

Translation Reader умеет изучать слова только в субтитрах. Нужна та же петля обучения (чтение → тап/словарь → перевод → split) для EPUB-книг: импорт `.epub`, чтение по главам со стрелками, синхронный переводной EPUB в нижней панели — на native и Web.

После первой реализации book-режима на **Web** скролл абзацев и включение split заметно лагают: на каждое слово строится отдельный виджет hit-target, а tokenize/mark идут батчем на всю главу. Нужна post-load оптимизация без смены контракта `parse_epub` и без ломки learning UX.

## What Changes

- Library принимает `.epub` наряду с субтитрами; open парсит книгу в главы/абзацы.
- Reader в book-режиме: внутри главы — скролл абзацев (как список cues); между главами — pager `[<] n/N [>]` снизу.
- Learning-поверхность как у субтитров: tap / highlight / vocab / offline translate на тексте абзацев.
- Split: для EPUB-оригинала перевод — только `.epub`; синхронизация панелей по **индексу главы** (готовые переводы с одинаковой структурой глав).
- Картинки, сноски и video-frames для EPUB — не входят.
- **Perf (Web book-mode):** лёгкий hit-test абзаца вместо WidgetSpan-на-слово; progressive tokenize + cache токенов по главе; highlight MAY отставать от появления текста/тапа; divider split без полного rebuild списков абзацев. Subtitle CueRow path **не** менять в этом follow-up.

## Capabilities

### New Capabilities

- `parse/epub`: разбор EPUB (bytes) в упорядоченные главы и plain-text абзацы для Reader (compute в Rust через FRB).
- `reader/chapters`: book-режим Reader — список абзацев главы, нижний chapter pager, learning-жесты как у cues; без timed cue list и без video.

### Modified Capabilities

- `library/documents`: import/open `.epub`; persist bytes; open-flow через EPUB-parse; linked translation только EPUB↔EPUB.
- `reader/split`: для book-режима — load/delete перевода как EPUB; sync панелей по индексу главы (не по `startMs`); плавный scroll/split chrome после perf follow-up.
- `reader/chapters` (delta follow-up): book-mode hit-test / tokenize progressive+cache; RC learning UX сохранён.

## Scope

- **Capability ID:** `parse/epub`, `reader/chapters`, `library/documents`, `reader/split`.
- **Слои:** Flutter `lib/` (Library + Reader book-mode + Drift) и Rust `rust/` (EPUB parse + thin FRB); FRB-контракт `parse_epub(bytes) → ParsedEpub` / `EpubParseErrorKind` (design D2).
- Поведение TOC: вложенные пункты с контентом = страницы пейджера; без TOC → одна глава на всю книгу.
- Платформы: native и Web (один bytes-контракт); perf follow-up ориентирован на Web jank, native MUST не регрессировать.
- Perf follow-up: только Flutter book-mode UI/tokenize orchestration; **не** lazy parse тел глав / не новый FRB tokenize API / не смена `parse_epub` payload.

## Out of scope

- Картинки, сноски, rich HTML/CSS в теле главы.
- Video attachment / `reader/video-frames` для EPUB-документов.
- Смешанный split EPUB↔SRT/VTT/ASS.
- Облачный sync библиотек; отдельный «магазин» книг.
- Редактирование/запись EPUB; выбор альтернативного EPUB-парсера после фиксации в design.
- Lazy chapter-body parse (TOC titles first) — отдельное решение, не этот follow-up.
- Переписывание subtitle `CueRow` WidgetSpan path (кроме неизбежного shared helper, если book выносит hit-test в отдельный виджет).

## Impact

- Library allowlist форматов и open-flow (subtitle vs book); Web EPUB persist binary-safe (base64 / Blob — design D6), не UTF-8 text roundtrip.
- Новый FRB surface EPUB parse; Reader book-mode session рядом с cue/video путями (chapter payload, без Load video).
- Reading anchor format-gated: chapter index в `lastReadStartMs` для epub.
- Split sync ветка: time (субтитры) vs chapter index (EPUB); translation load через `parse_epub`.
- Book-mode paragraph rows: TextSpan / paragraph-level hit-test; widget-тесты Keys `readerCueToken_*` для book обновляются (Semantics / finder по индексу).
- Progressive tokenize + in-memory chapter token cache; mark/highlight MAY догонять после первого tap-ready.
- Main specs после sync: `parse/epub`, `reader/chapters`, обновления `library/documents`, `reader/split`.

## References

- Sibling app: `../my_translator_app/AGENTS.md`.
- Meta: `docs/projects-map.md`; skills `work-with-parse`, `work-with-reader` (+ `chapters.md`), `work-with-pages` / upload, `work-with-drift`, `work-with-frb` / `work-with-frb-api`.
- Explore (2026-10-15): rbook; главы + абзацы; split EPUB↔EPUB; без картинок/сносок.
- Explore perf (2026-10-15): Web scroll/split jank → lightweight hit-test (A); chunked tokenize + chapter cache; highlight MAY lag; subtitles unchanged.
