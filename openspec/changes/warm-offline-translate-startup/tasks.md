## 1. Контекст и facade API

- [x] 1.1 Прочитать `proposal.md`, `design.md`, delta `specs/translate/offline/spec.md` (TO-09…TO-12) и skills `work-with-translate` + `work-with-pages` (`shell.md`); сверить текущие `lib/data/translate/*`, `web/translate_*.js`, точки Translate в Reader/Vocab — verify: краткая карта файлов совпадает с design D5
- [x] 1.2 Добавить в facade (`offline_translate.dart` + platform + native/web) публичный `warmUpOfflineTranslate` / `ensureOfflineTranslateReady` с **process-wide** in-flight Future, общим с Translate (design D2) — verify: два вызова `createOfflineTranslate()` + параллельный warm-up не стартуют второй `_loadAndInit` / второй JS pipeline; start hook и Translate path зовут один ensure

## 2. Native cold path + warm-up

- [x] 2.1 В `offline_translate_native.dart` вынести module-level ensure для warm-up+translate; при уже `isOfflineTranslateReady` — no-op — verify: повторный warm-up не зовёт повторный `initOfflineTranslate` зря
- [x] 2.2 Смягчить UI-block на cold load (design D3: yield кадра → sequential asset `await` → chunked copy + yields; FRB init остаётся async) — verify: widget/facade тест или pump-тест TO-11 показывает кадры/interaction во время fake slow init
- [x] 2.3 **Rust verify-only (design D8):** не менять публичный FRB `init_offline_translate` / `translate_text` / `is_offline_translate_ready` и не класть ORT в `init_app`; из `{app}/rust` прогнать `cargo test` (модуль translate / релевантные) — verify: тесты зелёные; API surface без нового bytes/OS-path контракта

## 3. Web worker warm-up

- [x] 3.1 В `translate_worker.js` + `translate_bridge.js` + `offline_translate_web.dart` добавить `warmup` (общий `translatorPromise` с translate; Dart/JS ensure process-wide) — verify: warmup до translate не ломает последующий gloss; гонка warmup∥translate не создаёт два pipeline

## 4. Start hook (тихий фон) — частично superseded

- [x] 4.1 ~~В `AppShell` после первого post-frame тихий warm-up без splash~~ **INVALIDATED:** заменено fullscreen gate (design D1 / tasks §7) — verify: тихий-only path больше не product default
- [x] 4.2 Добавить escape hatch на `AppShell` (`enableOfflineTranslateWarmUp` и/или skip при injectable translate) и обновить `test/features/shell_test.dart` (+ другие mounts), чтобы auto warm-up не грузил модель в тестах — verify: `shell_test` проходит без AssetBundle/FRB warm-up side effects

## 5. Translate button wait path

- [x] 5.1 Убедиться, что Reader/Vocab Translate ждут shared ensure/warm-up и показывают spinner на кнопке на время ожидания (TO-10); при необходимости подправить sheets — verify: widget-тест mid-init → loading → gloss/err

## 6. Тесты и анализ (v1 warm-up)

- [x] 6.1 Flutter-тесты по Traceability TO-09…TO-12 (v1 quiet warm-up) — verify: релевантные `flutter test` проходили на момент v1
- [x] 6.2 Из корня sibling app: `flutter analyze` без новых ошибок по затронутым путям — verify: analyze clean для v1

## 7. Fullscreen loading gate (translate/offline)

- [x] 7.1 После Drift open показать fullscreen loading и ждать `ensureOfflineTranslateReady` / warm-up; успех → product shell; fail → снять loader без обязательного SnackBar (TO-09, TO-13; design D1/D6) — verify: widget-тест gate → shell; fail → shell
- [x] 7.2 Сохранить/расширить escape hatch (тесты не блокируются на модели); обновить shell/app tests под gate UX — verify: tests green without real ORT load

## 8. Reader: убрать sync + preserve bottom

- [x] 8.1 Убрать one-shot sync на Show / Load translation / open для subtitles и EPUB (`_syncTranslationToTimeMs`, `_syncBookPanes`, align chapter-from-top на Show/Load) — verify: RS-05/18/19/20 (нет align к верху)
- [x] 8.2 Запоминать позицию низа (cue/paragraph + alignment) на scroll settle, Hide, pop; восстанавливать на Show/reopen без sync к верху; сохранить preserve верха (RS-10/11) — verify: Hide→Show низ на своём месте
- [x] 8.3 Drift + `DocumentRepository`: persist/load translation reading anchor (LD-31/32; design D11); прокинуть в Reader open — verify: unit/repo + reopen restore

## 9. Chapter pager link + bottom chrome

- [x] 9.1 EPUB split: pager/title только наверху; top prev/next выставляет главу низа (clamp) и rewind обоих списков к началу главы (RC-10/11/12) — verify: top next → bottom same chapter at top
- [x] 9.2 Нижняя панель text-only вплотную к divider (без delete bar / title / pager / лишнего top padding) (RS-23) — verify: widget layout
- [x] 9.3 Delete translation в AppBar рядом со split; убрать delete chrome из bottom (RS-07) — verify: delete from AppBar; bottom без delete

## 10. Регрессия и анализ (delta)

- [x] 10.1 Flutter-тесты: TO-09/13 gate; RS no-sync + bottom restore; RC linked pager; LD translation anchor; обновить/удалить тесты, ожидавшие one-shot — verify: релевантный `flutter test` green
- [x] 10.2 `flutter analyze` (+ `cargo test` translate smoke если трогали rust) по затронутым путям — verify: clean
