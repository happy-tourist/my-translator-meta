## Why

После morph-highlight shipped: одиночные леммы подсвечиваются, но **фразовые** записи словаря (основной контент пользователя) не красятся, а contiguous multi-token Add из Reader убран. Нужно вернуть сохранение фраз и подсветку phrase spans рядом с morph singles — иначе словарь снова оторван от чтения.

## What Changes

- **Вернуть** contiguous tap selection + phrase save из Reader (machine как в `add-reader-tap-vocab`: expand / shrink-to-single / gap-flash).
- **Phrase highlight**: ключ = casefold(join token surfaces); матч по consecutive токенам (не cue substring); paint **всех** токенов span’а; overlaps — красить обе.
- Morph highlight / WordNet для **single-token** остаётся; фразы **без** WordNet.
- Тап по подсвеченному (слово или фраза) → edit-sheet; при overlaps приоритет длиннее фраза > короче > single.
- Уже shipped (без отката): EN lemmatize, Drift dual-key schema 4, `mark_vocab_tokens` / `lemma_keys_for_surfaces`, `ReaderColors`, edit для singles.

## Scope

- **Capability ID:** `reader/highlight` — phrase-span membership + paint + edit priority; сохранить single-token morph highlight.
- **Capability ID:** `reader/tap` — восстановить contiguous multi-token Add; phrase surface = casefold(join tokens); single save = morph lemma.
- **Capability ID:** `dictionary/entries` — phrase keys (whitespace / join) как identity рядом с morph singles; upsert по тому же `lemma` ключу.
- **Слои:** Rust mark (phrase + morph) через FRB; Flutter selection machine + paint/edit; Drift без новой schema bump (schema 4 уже есть).
- **Язык:** morph только single-token EN WordNet; фразы — casefold(join).

## Out of scope

- Morph внутри фразы (лемматизация каждого слова фразы → join) — не в этом delta.
- Cue-substring identity (punct/whitespace как в сыром тексте) — матч только join токенов.
- Русская морфология / settings language gate.
- Offline / auto translate prefilling.
- Drag / long-press select; highlight на translation pane.
- Ручной UI merge-коллизий; облачный sync.
- Legacy stub cleanup UI (пользователь чистит старые ряды сам).

## Capabilities

### New Capabilities

- (нет) — `reader/highlight` уже введён этим change; delta расширяет поведение.

### Modified Capabilities

- `reader/highlight`: phrase spans + overlap paint/edit priority; отмена «phrases never highlight».
- `reader/tap`: contiguous Add restored; phrase save key = casefold(join); single = morph.
- `dictionary/entries`: Reader MAY добавлять multi-word keys; phrase identity = casefold(join).

## Impact

- Sibling app: расширить `mark_vocab_tokens` (или связанный mark) под phrase windows; вернуть selection state machine; sheet save branch single vs phrase; tests RH/RT/DE.
- Meta: delta specs/design/tasks + skills reader tap/highlight после apply.
- Совместимость: schema 4 / morph pipeline уже в коде; этот delta — поведение поверх shipped foundation.

## References

- Explore (phrase restore): D1=A phrase key casefold(join) + token match; morph singles only; paint all tokens; edit on tap; overlaps paint both; tap priority longest>shorter>single; selection = tap-vocab machine; legacy rows user deletes.
- Prior explore/ship: morph WordNet; Drift dual-key; `ReaderColors`; edit singles.
- Archived `add-reader-tap-vocab` (contiguous selection reference).
- `docs/projects-map.md`, sibling `../my_translator_app/AGENTS.md`.
- Skills: `work-with-reader` / tap+highlight, `work-with-tokenizematize`, `work-with-dictionary`, `work-with-styles`, `work-with-drift`, FRB.
