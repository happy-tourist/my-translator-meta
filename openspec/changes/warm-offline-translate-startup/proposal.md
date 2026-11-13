## Why

При старте сессии cold init офлайн-модели OPUS-MT всё ещё делает UI недоступным, а тихий warm-up без splash не даёт честного сигнала. Параллельно в Reader split one-shot sync (время / глава+абзац) и chrome нижней панели мешают: пользователь хочет независимый скролл низа с памятью позиции, связанные только стрелки глав сверху, и нижнюю панель из одного текста.

## What Changes

- **BREAKING (vs тихий warm-up в этом же change):** после открытия storage — **полноэкранный loading**, пока offline translate ensure/warm-up не завершится (успех или fail); Library/Vocab недоступны до снятия loader. Fail → снять loader и пустить в shell (ошибка потом на Translate).
- Process-wide shared ensure / warm-up + lazy Translate со spinner на кнопке (уже заложено) остаются; cold-path yields на native сохраняются.
- **BREAKING (`reader/split`):** убрать **весь** one-shot sync скролла/абзаца/времени при Show split, Load translation и open (субтитры и EPUB одинаково). Никогда не подгонять низ к верху.
- Сохранять позицию **нижней** панели при скролле / Hide / pop; **persist** якоря перевода (Drift); при Show / reopen — восстановить низ из сохранённого, не из верха.
- EPUB: стрелки глав **только на верхней** панели; переключают главу и внизу; при смене главы обе панели с **начала главы**. Нижний pager/title убрать.
- Нижняя панель: **только текст** вплотную к divider (без delete-bar, title, pager). Delete translation — рядом с кнопкой split в AppBar.

## Scope

- **Capabilities:** `translate/offline`, `reader/split`, `reader/chapters`, `library/documents`
- Warm-up UX: fullscreen gate после Drift open; platforms Web (Chrome), Windows, Android.
- Reader split/book: desync scroll, linked top chapter pager, bottom text-only chrome, translation reading anchor persist.
- Слои: Flutter shell / Reader / Drift repos; native+Web translate facade (без смены публичного FRB bytes-контракта).

## Out of scope

- Смена модели / пары языков / облачный AI translate; OS-path к модели в FRB.
- Persist split/divider; disk-cache модели между процессами.
- Resume mid-chapter для **оригинала** EPUB сверх текущего LD-28 (chapter index) — не расширяем в этом change, кроме якоря **перевода**.
- Realtime scroll mirroring; video-frame sync; dictionary/parse changes.

## Capabilities

### New Capabilities

- (нет)

### Modified Capabilities

- `translate/offline`: fullscreen loading gate на старте сессии вместо тихого warm-up; shared ensure; Translate wait + spinner.
- `reader/split`: без one-shot sync; preserve/restore нижней позиции; bottom pane text-only; delete у split в AppBar.
- `reader/chapters`: chapter pager только сверху в split; top arrows двигают низ; смена главы → scroll к началу главы.
- `library/documents`: persist якоря чтения **перевода** (скрытый translation document / эквивалент на original).

## Impact

- Flutter: `AppShell` / app bootstrap loading overlay; Reader `_toggleSplit` / sync helpers удалить или no-op; AppBar delete; Drift schema/repo для translation anchor; тесты TO/RS/RC/LD.
- Web/native translate facade: без смены FRB API; warm-up уже есть — меняется UX gate.
- Skills: `work-with-translate` / `warmup.md`, `work-with-reader` split-scroll / chapters, `work-with-drift`.

## References

- Explore decisions (fullscreen loader; no sync; top-linked chapters; bottom persist; delete by split; text-only bottom).
- Main specs: `translate/offline`, `reader/split`, `reader/chapters`, `library/documents`.
- Карта путей: `docs/projects-map.md`.
