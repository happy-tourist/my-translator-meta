## 0. Rust / FRB — no work

- [x] 0.1 Подтвердить по `design.md` § «Rust / FRB boundary»: новый API / правки `rust/` / FRB codegen / pairing **не** входят в change; парс перевода = повторный `parseSubtitles` как Library open. (Чеклист для apply-агента — не писать Rust-задачи.)

## 1. Drift: hidden translation link

- [x] 1.1 Прочитать `design.md` (D2), `specs/library/documents/spec.md` (LD-12…LD-16) и skill `.agents/skills/client/work-with-drift/SKILL.md` (+ schema.md при наличии)
- [x] 1.2 В sibling app расширить `Documents`: nullable `translationDocumentId` (FK), bump `schemaVersion` + migration; `flutter pub get` / codegen Drift при необходимости без ошибок
- [x] 1.3 Обновить `DocumentRepository`: list/watch только visible docs; `attachTranslation`; `detachAndDeleteTranslation`; delete original cascade translation; `loadLinkedTranslation` / loadBytes для связанного — unit-тесты LD-12…LD-15 зелёные

## 2. Sync helpers + deps

- [x] 2.1 Прочитать `design.md` (D4) и `specs/reader/split/spec.md` (RS-05, RS-06); skill `work-with-reader` / `split-scroll.md` (MVP time-first)
- [x] 2.2 Добавить в `pubspec.yaml` зависимость для scroll-to-index (например `scrollable_positioned_list`); `flutter pub get` успешен; не добавлять pixel-only sync package как единственный sync
- [x] 2.3 Реализовать pure helper `nearestCueIndex(List<CueDto> cues, BigInt startMs)` (+ unit-тест на nearest и tie-break) для RS-05; сравнивать `CueDto.startMs` как `BigInt`, без `toInt()`

## 3. Reader split UI + orchestration

- [x] 3.1 Прочитать `proposal.md` Scope, `specs/reader/split/spec.md` (RS-01…RS-09), design D1/D3/D5 и skills `work-with-pages` (`reader.md`), `work-with-reader`, `client-work-with-structure`, `client-work-with-test`
- [x] 3.2 `ReaderScreen` → Stateful: toggle split, resizable divider (default ratio 0.5), single-pane default (RS-01…RS-03); hide split сохраняет translation cues в сессии (RS-02); ctor: `originalDocumentId`, optional translation cues/id, inject `DocumentRepository` + optional `ParseSubtitlesFn`
- [x] 3.3 Нижняя панель: empty + load (`file_picker` → injected `parseSubtitles`); success → `attachTranslation` + show cues (RS-04, LD-13); fail → заметная ошибка, cues не показывать (RS-08); delete → `detachAndDeleteTranslation` + empty (RS-07, LD-14)
- [x] 3.4 Sync: на settle и после load / включения split с переводом — time (nearest `startMs`) затем pixel settle; guard `_syncing` (RS-05, RS-06)
- [x] 3.5 Library `_onDocumentTap`: при link — best-effort `loadBytes`+`parseSubtitles` перевода; push Reader с original id + optional translation; сбой перевода не блокирует оригинал (LD-16, RS-09); `listInFolder`/UI списка не показывает hidden translation (LD-12)

## 4. Tests + analyze

- [x] 4.1 Widget-тесты Reader/Library по SC-ID: RS-01…RS-09 и LD-12/LD-14/LD-16 (где применимо с fake repos/parse) — `flutter test` зелёный
- [x] 4.2 `flutter analyze` в корне app без ошибок по затронутому коду

## 5. UX polish: scroll preserve + delete chrome

- [x] 5.1 Прочитать `design.md` (D7, D8) и `specs/reader/split/spec.md` (RS-07 обновлённый, RS-10)
- [x] 5.2 При включении split: сохранить top-visible якорь оригинала и после layout `jumpTo` восстановить (RS-10); не сбрасывать верх в начало; sync низа после restore при наличии translation cues
- [x] 5.3 Перенести Delete из AppBar в верхнюю зону нижней панели (при cues или stuck link); AppBar оставить только toggle split; обновить/добавить widget-тесты RS-07 / RS-10 — `flutter test` + `flutter analyze` зелёные
