## Why

При старте сессии cold init офлайн-модели OPUS-MT всё ещё делает UI недоступным, а тихий warm-up без splash не даёт честного сигнала. Fullscreen gate уже честен, но **анимация лоадера на native всё равно замирает**: ~200+ МБ модели идут через Dart AssetBundle → FRB bytes и блокируют UI isolate. Reader split: zero-sync и mid-chapter persist уже в scope (D14–D16), но на **Android** после Show split **оба** панели (верх и низ), бывшие mid-chapter, всё ещё уезжают к **началу главы** — слабый remount/`jumpTo` restore; нужно добить надёжный preserve без смены продуктовых правил.

## What Changes

- **BREAKING (vs тихий warm-up в этом же change):** после открытия storage — **полноэкранный loading**, пока offline translate ensure/warm-up не завершится (успех или fail); Library/Vocab недоступны до снятия loader. Fail → снять loader и пустить в shell (ошибка потом на Translate). **Оставляем gate** (решение explore: не убирать лоадер).
- Process-wide shared ensure / warm-up + lazy Translate со spinner на кнопке остаются.
- **Native cold path (delta):** убрать «намертво» зависание UI isolate при gate — **не** тащить полный ONNX через Dart→FRB `Vec<u8>`; извлечь/скопировать model files в app-local dir (с yield) и инициализировать ORT по **OS-path / dir** (пересмотр прежнего запрета OS-path). Web остаётся Transformers.js worker (без FRB bytes).
- **BREAKING (`reader/split`):** убрать **весь** one-shot sync при Show / Load / open; preserve/persist низ; linked top chapter pager; text-only bottom; Delete у split в AppBar (уже реализовано / в tasks §8–9).
- **`library/documents` + `reader/split` (delta):** persist/restore **mid-chapter** позиции **оригинала** EPUB так же, как у низа (абзац + alignment рядом с chapter index); на open — сразу на месте, без rewind/sync.
- **`reader/chapters` (delta):** убрать закреплённый chapter title chrome у панели оригинала; оставить текст абзацев и pager (стрелки + `n / N`).
- **Bugfix (`reader/split` + `reader/chapters`):** Show/Hide/open/reopen EPUB **MUST NOT** сбрасывать панель(и) к началу главы, если есть session/Drift якорь; pager **MUST** всегда открывать новую главу с paragraph 0 (не PageStorage mid-chapter); seed обновлять и при non-drag scroll (wheel), чтобы верх тоже переживал exit.
- **Bugfix Android remount (D17):** mid верх → Show → верх остаётся mid; mid низ → Hide→Show → низ остаётся mid; скролл верха не двигает низ; автоскролл **только** при pager (к началу главы). Harden restore после remount (retry/layout/tokenize), особенно Android.

## Scope

- **Capabilities:** `translate/offline`, `reader/split`, `reader/chapters`, `library/documents`
- Warm-up UX: fullscreen gate после Drift open; platforms Web (Chrome), Windows, Android.
- Native: responsive gate spinner — path/file init (+ FRB контракт под path/dir).
- Reader split/book: desync scroll; linked top chapter pager; bottom text-only; translation **и original** mid-chapter reading anchor; original без fixed title chrome; **надёжный** preserve на Show/open (в т.ч. Android) + pager rewind к 0.
- Слои: Flutter shell / Reader / Drift; native+Web facade; **Rust FRB translate init** (native path).

## Out of scope

- Смена модели / пары языков / облачный AI translate.
- Убрать fullscreen gate / вернуть тихий warm-up как product default (отклонено explore: «оставь лоадер»).
- Persist split/divider.
- Realtime scroll mirroring; video-frame sync; dictionary/parse changes.
- Убирать pager / стрелки главы у оригинала.
- Обязательный remote download модели (только встроенные assets).

## Capabilities

### New Capabilities

- (нет)

### Modified Capabilities

- `translate/offline`: fullscreen gate; shared ensure; Translate wait + spinner; **native init без hard-freeze UI** (path/dir вместо обязательного полного bytes-through-Dart); TO-11 ужесточён.
- `reader/split`: без one-shot sync; preserve/restore нижней **и** mid-chapter верхней (EPUB) на Show/Hide/open/reopen **без rewind к началу главы** (Android remount harden); bottom text-only; delete у split в AppBar.
- `reader/chapters`: chapter pager только сверху в split; top arrows двигают низ; смена главы → **всегда** начало главы (не mid из PageStorage); **без** fixed chapter title chrome на оригинале.
- `library/documents`: persist якоря чтения **перевода**; **EPUB оригинал** — chapter + paragraph (+ alignment).

## Impact

- Flutter: `AppShell` gate; native facade extract→path; Reader book pane chrome + original EPUB anchor persist/restore; Drift schema/repo; тесты TO/RS/RC/LD.
- Rust/FRB: новый или расширенный init по path/dir (codegen); ORT load from files; bytes-init MAY остаться для тестов/compat.
- Web: без смены worker-контракта (уже вне FRB).
- Skills: `work-with-translate` / `warmup.md` (D8 path), shell gate; при необходимости `work-with-drift` / `work-with-reader` (EPUB anchor).

## References

- Explore (2026-11-13): native freeze; оставить loader; вариант B — path init; D8 bytes-only снят для native.
- Explore (2026-11-13, Reader): reopen — низ помнит скролл, верх к началу главы; убрать fixed title; стрелки оставить; «скролить никуда не надо — только запоминать».
- Explore (2026-11-16, Reader): Show/open панели всё ещё к началу главы; reopen — только низ; pager → запомненный mid вместо 0 (PageStorage / restore miss).
- Explore (2026-11-16, Android): оба mid → Show → оба к началу главы; ожидания пользователя = канон (keep mid; no sync; pager-only autorescroll).
- Main specs: `translate/offline`, `reader/split`, `reader/chapters`, `library/documents`.
- Карта путей: `docs/projects-map.md`.
