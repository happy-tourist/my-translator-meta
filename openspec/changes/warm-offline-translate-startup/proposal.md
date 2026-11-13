## Why

При старте сессии cold init офлайн-модели OPUS-MT всё ещё делает UI недоступным, а тихий warm-up без splash не даёт честного сигнала. Fullscreen gate уже честен, но **анимация лоадера на native всё равно замирает**: ~200+ МБ модели идут через Dart AssetBundle → FRB bytes и блокируют UI isolate. Параллельно в Reader split one-shot sync и chrome нижней панели мешали независимому скроллу низа. После restore низа при reopen верх EPUB всё ещё открывается с **начала главы** (только LD-28 chapter index), а закреплённый title главы на оригинале лишний — нужен только текст + pager.

## What Changes

- **BREAKING (vs тихий warm-up в этом же change):** после открытия storage — **полноэкранный loading**, пока offline translate ensure/warm-up не завершится (успех или fail); Library/Vocab недоступны до снятия loader. Fail → снять loader и пустить в shell (ошибка потом на Translate). **Оставляем gate** (решение explore: не убирать лоадер).
- Process-wide shared ensure / warm-up + lazy Translate со spinner на кнопке остаются.
- **Native cold path (delta):** убрать «намертво» зависание UI isolate при gate — **не** тащить полный ONNX через Dart→FRB `Vec<u8>`; извлечь/скопировать model files в app-local dir (с yield) и инициализировать ORT по **OS-path / dir** (пересмотр прежнего запрета OS-path). Web остаётся Transformers.js worker (без FRB bytes).
- **BREAKING (`reader/split`):** убрать **весь** one-shot sync при Show / Load / open; preserve/persist низ; linked top chapter pager; text-only bottom; Delete у split в AppBar (уже реализовано / в tasks §8–9).
- **`library/documents` + `reader/split` (delta):** persist/restore **mid-chapter** позиции **оригинала** EPUB так же, как у низа (абзац + alignment рядом с chapter index); на open — сразу на месте, без rewind/sync.
- **`reader/chapters` (delta):** убрать закреплённый chapter title chrome у панели оригинала; оставить текст абзацев и pager (стрелки + `n / N`).

## Scope

- **Capabilities:** `translate/offline`, `reader/split`, `reader/chapters`, `library/documents`
- Warm-up UX: fullscreen gate после Drift open; platforms Web (Chrome), Windows, Android.
- Native: responsive gate spinner — path/file init (+ FRB контракт под path/dir).
- Reader split/book: desync scroll; linked top chapter pager; bottom text-only; translation **и original** mid-chapter reading anchor; original без fixed title chrome.
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
- `reader/split`: без one-shot sync; preserve/restore нижней **и** mid-chapter верхней (EPUB) позиции; bottom text-only; delete у split в AppBar.
- `reader/chapters`: chapter pager только сверху в split; top arrows двигают низ; смена главы → scroll к началу главы; **без** fixed chapter title chrome на оригинале.
- `library/documents`: persist якоря чтения **перевода**; **EPUB оригинал** — chapter + paragraph (+ alignment).

## Impact

- Flutter: `AppShell` gate; native facade extract→path; Reader book pane chrome + original EPUB anchor persist/restore; Drift schema/repo; тесты TO/RS/RC/LD.
- Rust/FRB: новый или расширенный init по path/dir (codegen); ORT load from files; bytes-init MAY остаться для тестов/compat.
- Web: без смены worker-контракта (уже вне FRB).
- Skills: `work-with-translate` / `warmup.md` (D8 path), shell gate; при необходимости `work-with-drift` / `work-with-reader` (EPUB anchor).

## References

- Explore (2026-11-13): native freeze; оставить loader; вариант B — path init; D8 bytes-only снят для native.
- Explore (2026-11-13, Reader): reopen — низ помнит скролл, верх к началу главы; убрать fixed title; стрелки оставить; «скролить никуда не надо — только запоминать».
- Main specs: `translate/offline`, `reader/split`, `reader/chapters`, `library/documents`.
- Карта путей: `docs/projects-map.md`.
