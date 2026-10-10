## Why

В Reader сейчас нельзя выделить слово или смежную фразу в оригинале и сохранить в личный словарь из контекста субтитров — словарь заполняется только вручную на экране Vocabulary. Нужен MVP-путь «тап → выбор → форма перевода → Drift», чтобы изучение слов шло прямо во время чтения.

## What Changes

- Токенизация текста оригинальных реплик (Rust compute) с позициями для hit-test в UI.
- В Reader на панели оригинала: тап по словам, contiguous selection, bottom CTA и bottom sheet для перевода/контекста.
- Сохранение выбранного слова или фразы в словарь через существующий Drift-persist (upsert по нормализованному ключу).
- Пакетная токенизация документа при открытии Reader (не per-line на каждый build).

## Scope

- **Capability ID:** `reader/tap` (новый) — тап/выделение на оригинале, CTA, sheet, batch tokenize для cues оригинала.
- **Capability ID:** `dictionary/entries` (изменяемый) — добавление записи из Reader с той же политикой upsert/нормализации, что и Vocab UI.
- **Слои:** Rust (tokenize compute + FRB surface) и Flutter Reader UI + Drift persist.
- **Жесты:** только tap (без drag-select); selection только внутри одной реплики; только contiguous слова; только панель original.
- **Ключ словаря в MVP:** stub-нормализация surface/фразы (`trim` + case-fold); настоящая морфологическая лемматизация — вне scope.

## Out of scope

- Морфологическая лемматизация / отдельный lemma crate.
- Offline / auto translate prefilling перевода.
- Подсветка lemma в Reader после save (и refresh marks).
- Drag / long-press phrase select; non-contiguous «составная фраза»; выделение across cues.
- Тап/выделение на панели translation.
- Кастомная «карточка слова» с полировкой (audio, drag-to-dismiss) — MVP chrome = bottom sheet.
- Изменение Vocab list UI (кроме того, что новые записи появляются через общий persist).

## Capabilities

### New Capabilities

- `reader/tap`: тап по токенам оригинала, contiguous multi-tap selection, bottom CTA, bottom sheet добавления в словарь; batch tokenize cues оригинала при открытии Reader.

### Modified Capabilities

- `dictionary/entries`: разрешить / описать добавление записи из Reader (не только Vocab UI), с той же upsert-политикой по нормализованному ключу.

## Impact

- Sibling app: новый FRB batch tokenize cues оригинала (char spans); панель оригинала в Reader получает token hit-test / selection; Rust tokenizer (tokenize-only); сохранение в словарь из Reader через общий Drift-persist (не Rust SQLite).
- Flutter: словарьный repository доходит до Reader через shell → Library → Reader (сейчас только Vocab); seam для подмены tokenize в widget-тестах.
- Meta: delta `reader/tap` + `dictionary/entries`; skills reader/tokenize — канон поведения (MVP chrome = bottom sheet, не target word-card).
- Совместимость: Vocab CRUD не ломается; parse/split без изменения контракта парсинга.

## References

- Explore-решения сессии: panko tokenize-only; lemma/translate/highlight позже; tap + bottom CTA + sheet; contiguous / one cue / original-only; batch on open; context auto-fill editable; upsert как в Vocab.
- `docs/projects-map.md`, `openspec/config.yaml` (пример capability `reader/tap`).
- Sibling `../my_translator_app/AGENTS.md`.
- Skills (не дублировать): `work-with-reader`, `work-with-tokenizematize`, `work-with-dictionary`, `work-with-frb` / `work-with-frb-api`.
