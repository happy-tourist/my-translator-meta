# Vocabulary (личный словарь)

Read with the [core pages skill](SKILL.md) for shell/nav. Folder roles:
`client-work-with-structure` → `lib/features/vocab/`.

## Target screens

| Screen | Suggested path | Role |
|--------|----------------|------|
| Vocabulary list | `lib/features/vocab/vocabulary_screen.dart` | Entries: lemma, translation, optional context |
| Entry detail (optional) | same feature folder | Edit notes / see contexts / delete |

Primary shell destination alongside Library and Settings ([shell.md](shell.md)).

## UX canon (target)

- List dictionary entries stored **locally** (Rust + SQLite target).
- Show lemma + translation; context (source cue/phrase) when present.
- Remove entry; edits only when Rust API supports them.
- Opening an entry may deep-link back to a document/cue **later** — do not
  invent cross-navigation until OpenSpec says so.
- Empty state: explain that words are added from Reader word card
  ([reader.md](reader.md)).

## Layering

```text
VocabularyScreen
  → orchestration: list / delete / (optional) update
      → FRB dictionary API (Rust authority)
```

- Flutter: list UI, confirm delete, search filter **on already-fetched** rows
  or via Rust query — prefer Rust for large lists when API exists.
- Do not keep a second source of truth in `SharedPreferences` for the
  dictionary if SQLite/Rust owns it.

## Relationship to Reader

- **Add** happens on the word card in Reader.
- **Highlight** in Reader reads membership from the same dictionary.
- Vocab screen is management/review, not the primary add surface.

## Out of scope

- Multi-device sync (AGENTS).
- Social / shared decks.
- Spaced-repetition drills unless a change explicitly adds them.
