## Context

См. `proposal.md` — Why / Scope. Уже в коде (apply v1): process-wide `warmUpOfflineTranslate` / `ensureOfflineTranslateReady`, quiet post-frame hook в `AppShell`, native chunked asset copy, Web `warmup`, Translate spinner. Пользователь уточнил: UI при init всё равно мёртв → нужен **fullscreen gate**; плюс Reader: убрать one-shot sync, persist низа, linked top chapter arrows, text-only bottom, Delete у split.

Платформы: Web (Chrome), Windows, Android.

## Goals / Non-Goals

**Goals:**

- Fullscreen loading после Drift open до ready/fail ensure; затем shell.
- Shared process-wide ensure (native+Web); Translate wait + spinner.
- Reader: zero scroll/time/paragraph sync; restore bottom from own anchor; top-only chapter pager drives bottom chapter; chapter change → both lists at top; bottom = text only; Delete next to split.

**Non-Goals:**

- OS-path / disk-cache модели; ORT в `init_app`.
- Persist split/divider; mid-chapter persist для **оригинала** EPUB сверх LD-28.
- Realtime scroll mirror; video sync.

## Decisions

### D1 — Fullscreen loading gate (revises quiet warm-up)

**Выбор:** после успешного открытия Drift (конец bootstrap `TranslationReaderApp`) показать **полноэкранный** loading и ждать `ensureOfflineTranslateReady` / `warmUpOfflineTranslate`. Product shell (Library/Vocab) не интерактивен, пока gate не снят. Успех → shell. Fail → снять gate, показать shell; без обязательного SnackBar от warm-up (ошибка на Translate).

**Почему:** cold init всё равно блокирует UI; честный fullscreen лучше тихого «мёртвого» shell.

**Escape hatch:** тесты — `enableOfflineTranslateWarmUp: false` / skip gate (как сейчас для quiet hook).

**Было отклонено ранее, теперь принято:** блокирующий splash на старте сессии (после storage).

### D2 — Shared init future (без изменений контракта)

Process-wide module-level Future native+Web; Translate и warm-up/gate делят один ensure. Уже реализовано.

### D3 — Cold path yields (без изменений цели)

Native: yield → sequential asset load → chunked own-copy + yields → FRB init async. Нужен и под gate (живой spinner), и если Translate после fail-dismiss.

### D4 — Translate button UX

Без изменений: spinner на кнопке при wait после shell.

### D5 — Точки врезки (обновлено)

| Зона | Файлы / роль |
|------|----------------|
| Gate | `lib/app/app.dart` / `shell.dart` — fullscreen overlay until ensure; escape hatch |
| Facade | `lib/data/translate/*` — уже ensure/warmUp |
| Reader sync removal | `reader_screen.dart` — убрать `_syncTranslationToTimeMs` / `_syncBookPanes` / `_alignTranslationChapterToOriginal` с Show/Load/open |
| Chapter link | `_goToChapter(fromOriginal: true)` — также clamp translation chapter; rewind both lists to 0; убрать pager/title с translation pane |
| Bottom chrome | убрать `_buildBottomDeleteChrome` из bottom; Delete IconButton рядом со split в AppBar |
| Persist | `DocumentRepository` + Drift schema — translation reading anchor; Reader save on bottom settle / pop / Hide |
| Tests | shell gate; split no-sync; bottom restore; chapter link; AppBar delete; repo anchor |
| Rust | D8 без API change |

### D6 — Warm-up / gate errors

Fail ensure → dismiss fullscreen; no SnackBar from gate. Translate показывает kind.

### D7 — Тесты

- TO-09/13 gate; TO-10/11; RS-05/18/19/20/23/24; RC-10/11/12; LD-31/32.

### D8 — Rust / FRB (без изменений)

Verify-only; no new FRB.

### D9 — Zero one-shot sync (NEW)

Удалить/не вызывать one-shot на Show, Load translation, open. Не подменять restore низа sync с верха. Top preserve (RS-10) и bottom preserve (RS-11/24) — независимо.

### D10 — Linked chapter from top arrows only (NEW)

В split EPUB: pager только на original pane. Top prev/next → `_translationChapterIndex = clamp(original)`; оба paragraph list → index 0. Нет независимого pager на низу (RC-11). Scroll абзацев не зеркалится.

### D11 — Translation reading anchor persist (NEW)

Расширить Drift/repo: якорь низа (subtitle: startMs+alignment; EPUB translation: paragraph index+alignment; chapter согласуется с original при reopen через D10). Хранить на original companion fields **или** на hidden translation row — выбрать минимальный schema path в apply (`work-with-drift`). Main LD-17 original не ломать. EPUB original mid-chapter persist не расширяем (LD-28).

### D12 — Bottom text-only + Delete by split (NEW)

Bottom: `Stack`/list без Column delete bar; list padding top ≈ 0 к divider; без chapter title/pager на translation pane. AppBar: split + delete (visible when translation present/linked).

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Fullscreen gate удлиняет time-to-Library | Честнее мёртвого UI; fail быстро dismiss |
| Schema migration для translation anchor | Minimal columns; tests; fail-open on missing |
| Users relied on one-shot align | Product decision — explicit remove |
| Unequal EPUB chapter counts | Clamp + existing mismatch message |

## Migration Plan

- Warm-up code уже в app: сменить quiet hook → gate overlay.
- Drift migration для translation anchor; старые installs без якоря низа → низ с начала.
- Rollback sync: вернуть one-shot helpers (хуже UX по новому решению).

## Open Questions

Нет — зафиксировано explore (D5–D7 пользователя: persist низа; rewind on chapter; Delete у split).
