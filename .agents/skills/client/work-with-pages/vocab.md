# Vocabulary (личный словарь)

Read with the [core pages skill](SKILL.md) for shell/nav. Folder roles:
`client-work-with-structure` → `lib/features/vocab/`. Persist:
`work-with-drift` → `lib/data/dictionary/`.

## Target screens

| Screen | Suggested path | Role |
|--------|----------------|------|
| Vocabulary list | `lib/features/vocab/vocabulary_screen.dart` | Entries: lemma, translation, optional context |
| Entry detail (optional) | same feature folder | Edit notes / see contexts / delete |

Primary shell destination alongside Library ([shell.md](shell.md)).

## UX canon (target)

- List dictionary entries stored **locally** via **Drift** (`dictionary/entries`).
- Show lemma + translation; context (source cue/phrase) when present.
- **Add** from Vocab UI (lemma + translation + optional context) — empty state has form/CTA add; do **not** require Reader for first entries.
- Remove entry; validation on empty lemma/translation after trim.
- Opening an entry may deep-link back to a document/cue **later** — do not invent cross-navigation until OpenSpec says so.
- **Reader add** (current: `reader/tap`) — bottom sheet after selection on
  original; same Drift `DictionaryRepository.add` / upsert. Highlight paint
  in Reader is still later.

## Layering

```text
VocabularyScreen
  → orchestration: list / add / delete / (optional) update
      → Drift dictionary repository   # persist authority
```

- Flutter: list UI, add form, confirm delete, search filter on fetched rows.
- Do not keep a second source of truth in `SharedPreferences`.
- Do **not** wait for FRB dictionary storage API — persist is Drift, not Rust SQLite.

## Relationship to Reader

- **Add from Vocab UI:** in-screen form.
- **Add from Reader:** CTA → sheet → same Drift store (`work-with-reader` / tap.md).
- **Highlight** in Reader (later) reads membership from the same dictionary.

## Out of scope

- Multi-device sync (AGENTS).
- Social / shared decks.
- Spaced-repetition drills unless a change explicitly adds them.
