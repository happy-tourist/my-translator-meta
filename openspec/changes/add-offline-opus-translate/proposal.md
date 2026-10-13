## Why

При добавлении слова или фразы в словарь перевод сейчас вводится только вручную. Нужен офлайн EN→RU gloss по кнопке в формах добавления/редактирования (Reader sheets и Vocab), чтобы ускорить набор словаря без сети и без API-ключей — на Web (где идёт основная проверка) и на нативных сборках.

## What Changes

- Офлайн-перевод EN→RU по явной кнопке рядом с полем слова/леммы: loading на кнопке → вставка результата в поле Translation.
- Одна bilingual-модель OPUS-MT en→ru (ONNX) в assets приложения; атрибуция и предупреждение о возможных ошибках перевода в UI.
- Dual runtime: Web — Transformers.js; native — Rust ONNX Runtime через существующий FRB; общая продуктовая поверхность для UI.
- Обновление канона docs/skills/AGENTS: на Web допустим JS offline-engine при том же UX-контракте (исключение из «весь translate только в Rust»).

## Scope

- **Capability ID:** `translate/offline` (новый) — офлайн gloss слова/фразы EN→RU, готовность движка, ошибки unavailable/empty/fail.
- **Capability ID:** `dictionary/entries` (изменяемый) — кнопка Translate на форме Vocab (добавление).
- **Capability ID:** `reader/tap` (изменяемый) — кнопка Translate на Add-sheet и Edit-sheet Vocabulary из Reader.
- **Слои:** Flutter UI (Vocab + Reader sheets) + оркестрация; Rust translate compute + FRB на native; Web JS engine за тем же фасадом; assets модели при apply.
- **Пара:** только EN→RU в этом change.
- **Текст запроса:** surface как на экране (не отдельный morph-пересчёт для translate).
- **UX:** кнопка у поля слова/леммы; loading на кнопке; пустой ввод → кнопка disabled (Vocab) / sheet не открывается без selection (Reader, как сейчас); после успеха Translation перезаписывается; на Add **и** Edit sheet — Translate disabled до закрытия после старта (RT-19/RT-24); на Vocab — disabled на время loading, после снова enabled; ошибки → SnackBar (не inline validation); facade `lib/data/translate/` + Keys — design D4.
- **Копирайт/доверие:** locked copy `OPUS-MT (Helsinki-NLP), CC-BY 4.0` + `Translation may be inaccurate` (design D4.6).

## Out of scope

- Другие языковые пары и «докачка» дополнительных пакетов.
- NLLB / `trad` / облачный или AI-перевод.
- Автоперевод без нажатия кнопки; auto-save в словарь после translate.
- Batch-перевод субтитров / целых документов.
- Экран Settings для выбора языков.
- Отдельный полноэкранный «скачиваем пакет» (модель в assets; loading только на кнопке).
- Фильтр по длине перевода (3× и т.п.) — только disclaimer.
- Смена политики identity/upsert словаря и lemma highlight.

## Capabilities

### New Capabilities

- `translate/offline`: офлайн EN→RU перевод слова или фразы по запросу UI; состояния готовности/loading/ошибки; без обязательной сети в runtime после установки сборки с assets.

### Modified Capabilities

- `dictionary/entries`: форма добавления на Vocab — кнопка Translate у lemma, fill Translation, атрибуция/disclaimer.
- `reader/tap`: Add-sheet и Edit-sheet — кнопка Translate у Word, fill Translation, те же кредиты/disclaimer; disable Translate до закрытия sheet после старта (Add и Edit; RT-19/RT-24). Новые SC-ID: **RT-18…RT-25** (не пересекаются с main RT-14…RT-17).

## Impact

- Sibling app: native FRB `init_offline_translate` (asset bytes) + `translate_text` + Web JS путь; assets ONNX/tokenizer; UI Vocab и Reader sheets; codegen FRB при смене `api/`.
- Meta: delta specs; правка skills/AGENTS под dual-runtime исключение на Web; `work-with-translate` — OPUS-MT en→ru + locked FRB/error kinds.
- Размер поставки: ONNX assets порядка ~100 MB+ в сборке.
- Совместимость: ручной ввод перевода и Drift persist без изменения контракта записей.

## References

- Explore-решения: dual A (Transformers.js web + Rust ORT native); OPUS-MT `onnx-community/opus-mt-en-ru` в assets; EN→RU; Vocab + Reader Add/Edit; surface as displayed; loading на кнопке; атрибуция + disclaimer; без length-filter.
- `docs/projects-map.md`, `openspec/config.yaml` (домен `translate/offline`).
- Sibling `../my_translator_app/AGENTS.md`.
- Skills (не дублировать): `work-with-translate`, `work-with-frb` / `work-with-frb-api`, `work-with-pages`, `work-with-reader`, `client-work-with-errors`.
