## Why

После добавления слов в словарь из Reader сохранённые единицы не видны в оригинале транскрипта, а повторный тап не открывает карточку для правки. Нужна подсветка по морфологической лемме и edit-sheet по тапу на уже сохранённое — иначе словарь оторван от чтения.

## What Changes

- Морфологические lemma-keys (EN WordNet) на токенах оригинала; stub-нормализация как identity для словаря выпиливается.
- Подсветка single-token вхождений сохранённых лемм на панели оригинала (theme accent light+dark).
- Тап по уже сохранённому токену → сразу edit-sheet с содержимым; Add остаётся только для несохранённых single-token.
- Drift: morph `lemma` + `surface_key`; одноразовая schema migration с merge коллизий.

## Scope

- **Capability ID:** `reader/highlight` (новый) — paint по `is_in_vocab`, refresh после save/delete/migrate, edit по тапу на подсвеченный токен.
- **Capability ID:** `reader/tap` (изменяемый) — Add только single-token; multi-token selection/phrase save убирается из текущего поведения; in_vocab-тап минует CTA Add.
- **Capability ID:** `dictionary/entries` (изменяемый) — identity = morph lemma; `surface_key`; migration merge; lookup/upsert по morph-ключу.
- **Слои:** Rust (lemmatize + membership flags через FRB) и Flutter Reader UX + Drift persist + theme tokens.
- **Язык:** всегда EN WordNet на токене; не-EN / morph-fail → ключ = casefold(surface); RU morph и settings language — вне scope.
- **Highlight:** только single-token; phrase spans — later.

## Out of scope

- Contiguous phrase highlight и phrase save из Reader (later вместе).
- Русская морфология / отдельный RU lemmatizer.
- Settings language gate для выбора бэкенда лемматизации.
- Offline / auto translate prefilling перевода.
- Drag / long-press select; тап на панели translation.
- Ручной UI разбора merge-коллизий при миграции.
- Облачный sync словаря.

## Capabilities

### New Capabilities

- `reader/highlight`: подсветка сохранённых single-token лемм на оригинале; refresh marks; тап → edit-sheet (translation/context, upsert/delete).

### Modified Capabilities

- `reader/tap`: сужение Add до single-token; убрать multi-token contiguous Add/phrase surface из текущего MVP; in_vocab bypass CTA в пользу highlight/edit.
- `dictionary/entries`: morph lemma как unique identity; `surface_key` для legacy/lookup; одноразовая migration с merge по `updated_at`; stub-only normalize больше не является ключом identity.

## Impact

- Sibling app: зависимость lemmatizer (EN); расширение tokenize/lemma FRB контракта (`lemma_key` / `is_in_vocab` / `mark_vocab_tokens` / `lemma_keys_for_surfaces`); Reader paint + gesture ветвление; Drift schema + migration; theme highlight tokens.
- Meta: delta specs + skills reader/tokenize/dictionary/styles (канон morph highlight).
- Совместимость: существующие stub-записи мигрируют один раз; фразовые записи остаются в словаре, но не подсвечиваются до later.

## References

- Explore-решения: D1=B morph сразу; D2=`wordnet-lemmatizer`; D3=EN-only; D4=Rust `is_in_vocab`; D5=single-token; D6=dual-key + migrate on schema migration; Q1–Q8 (merge по max `updated_at`; phrases не highlight; tap in_vocab → edit; Add single-token; WordNet всегда; theme green tint).
- `docs/projects-map.md`, `openspec/config.yaml` (домен `reader/highlight`).
- Sibling `../my_translator_app/AGENTS.md`.
- Skills (не дублировать): `work-with-reader` / highlight, `work-with-tokenizematize` / lemmatize, `work-with-dictionary`, `work-with-styles`, `work-with-drift`, FRB skills.
