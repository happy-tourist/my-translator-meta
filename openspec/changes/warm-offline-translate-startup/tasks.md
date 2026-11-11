## 1. Контекст и facade API

- [x] 1.1 Прочитать `proposal.md`, `design.md`, delta `specs/translate/offline/spec.md` (TO-09…TO-12) и skills `work-with-translate` + `work-with-pages` (`shell.md`); сверить текущие `lib/data/translate/*`, `web/translate_*.js`, точки Translate в Reader/Vocab — verify: краткая карта файлов совпадает с design D5
- [x] 1.2 Добавить в facade (`offline_translate.dart` + platform + native/web) публичный `warmUpOfflineTranslate` / `ensureOfflineTranslateReady` с **process-wide** in-flight Future, общим с Translate (design D2) — verify: два вызова `createOfflineTranslate()` + параллельный warm-up не стартуют второй `_loadAndInit` / второй JS pipeline; start hook и Translate path зовут один ensure

## 2. Native cold path + warm-up

- [x] 2.1 В `offline_translate_native.dart` вынести module-level ensure для warm-up+translate; при уже `isOfflineTranslateReady` — no-op — verify: повторный warm-up не зовёт повторный `initOfflineTranslate` зря
- [x] 2.2 Смягчить UI-block на cold load (design D3: yield кадра → sequential asset `await` → `Isolate.run`/`compute` для крупных copy; FRB init остаётся async) — verify: widget/facade тест или pump-тест TO-11 показывает кадры/interaction во время fake slow init
- [x] 2.3 **Rust verify-only (design D8):** не менять публичный FRB `init_offline_translate` / `translate_text` / `is_offline_translate_ready` и не класть ORT в `init_app`; из `{app}/rust` прогнать `cargo test` (модуль translate / релевантные) — verify: тесты зелёные; API surface без нового bytes/OS-path контракта

## 3. Web worker warm-up

- [x] 3.1 В `translate_worker.js` + `translate_bridge.js` + `offline_translate_web.dart` добавить `warmup` (общий `translatorPromise` с translate; Dart/JS ensure process-wide) — verify: warmup до translate не ломает последующий gloss; гонка warmup∥translate не создаёт два pipeline

## 4. Start hook (тихий фон)

- [x] 4.1 В `AppShell` после **первого post-frame** (design D1) один раз вызвать platform warm-up без splash/статуса; **не** из DB-loading `TranslationReaderApp` — verify: хук срабатывает post-frame shell; ошибка warm-up не показывает SnackBar сама (D6)
- [x] 4.2 Добавить escape hatch на `AppShell` (`enableOfflineTranslateWarmUp` и/или skip при injectable translate) и обновить `test/features/shell_test.dart` (+ другие mounts), чтобы auto warm-up не грузил модель в тестах — verify: `shell_test` проходит без AssetBundle/FRB warm-up side effects

## 5. Translate button wait path

- [x] 5.1 Убедиться, что Reader/Vocab Translate ждут shared ensure/warm-up и показывают spinner на кнопке на время ожидания (TO-10); при необходимости подправить sheets — verify: widget-тест mid-init → loading → gloss/err

## 6. Тесты и анализ

- [x] 6.1 Flutter-тесты по Traceability: TO-09 (warm-up после AppShell start seam), TO-10 (wait + loading), TO-11 (responsive cold init с fake slow), in-process ready после warm-up (часть TO-12) — verify: `flutter test` (релевантные файлы) проходит
- [x] 6.2 Из корня sibling app: `flutter analyze` без новых ошибок по затронутым путям — verify: analyze clean для change
