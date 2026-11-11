## Context

См. `proposal.md` — Why / Scope. Сейчас offline translate: native = AssetBundle bytes → lazy `init_offline_translate` → `translate_text` (`lib/data/translate/offline_translate_native.dart`); Web = Transformers.js worker, pipeline создаётся на первом `translate` (`web/translate_worker.js`). Тяжёлый init намеренно не в `#[frb(init)] init_app`. Модель уже в assets (~encoder 50MB + decoder 178MB native); «замирание» — cold load/copy/init на UI-критичном пути при первом Translate.

Платформы: Web (Chrome), Windows, Android.

## Goals / Non-Goals

**Goals:**

- Тихий warm-up после первого кадра UI; lazy на Translate как shared init/fallback.
- Живой loading на кнопке при ожидании; UI не freeze на cold path.
- Один facade API для warm-up + translate (native/web).

**Non-Goals:**

- Смена FRB на OS-path к модели; disk-cache оптимизация следующего запуска.
- Splash / shell progress chrome; смена модели или языковой пары.
- Класть ORT load в `init_app` / блокировать `RustLib.init`.

## Decisions

### D1 — Когда стартует warm-up

**Выбор:** **первый post-frame callback `AppShell`** (когда уже открыт Drift и виден product shell Library/Vocab), не во время bootstrap-спиннера `TranslationReaderApp` (открытие БД), не до первого кадра shell, не из `RustLib.init` / `main` до `runApp`.

**Почему:** design archive запрещает тяжёлый load в `init_app`; «основной UI» в TO-09 = product shell, не экран «открываем storage»; пользователь видит Library сразу; прогрев идёт в фоне.

**Тестовый escape hatch (обязателен):** `AppShell` MUST иметь явный способ не стартовать production warm-up в widget-тестах (например `enableOfflineTranslateWarmUp: false` и/или skip auto-warm-up, когда в тест передан injectable `offlineTranslate`). Иначе `shell_test` и аналоги сдетонируют на AssetBundle/FRB. Production `TranslationReaderApp` → `AppShell` оставляет warm-up включённым по умолчанию.

**Альтернативы:** блокирующий splash (отклонено); warm-up только при входе в Reader (отклонено — дольше «холодная» кнопка на Vocab); хук из `main`/`TranslationReaderApp` до shell (отклонено — прогрев во время DB spinner / до product UI).

### D2 — Shared init future (native + web)

**Выбор:** публичный seam на facade, например `warmUpOfflineTranslate()` / `ensureOfflineTranslateReady()`, который:
- native: тот же `_ensureInit` / `_loadAndInit` + `isOfflineTranslateReady`;
- web: явный worker op `warmup` (или эквивалент), поднимающий `pipeline` до первого translate;
- Translate и warm-up **делят один in-flight Future**, чтобы двойной init не гонялся.

**Контракт shared Future (критично):** ensure/warm-up state MUST быть **process-wide / module-level** (static или один модульный слот) в native и web impl, а **не** только на экземпляре `_NativeOfflineTranslate` из одного вызова `createOfflineTranslate()`. Сейчас каждый `createOfflineTranslate()` держит свой `_initFuture`; Reader/Vocab/tests иногда создают второй facade — при in-flight init это даёт **двойной** load ~230MB / двойной pipeline. Warm-up из shell и Translate из sheets MUST сходиться в один ensure, даже если `OfflineTranslateFn` создали дважды. После успешного ready повторный warm-up — no-op (`isOfflineTranslateReady` / cached pipeline).

**Почему:** TO-09/TO-10; текущий native почти так, но per-instance `_initFuture` недостаточен для start-hook + multi-create.

**Альтернативы:** отдельный «prefetch» без общего future (риск двойного ORT/JS load); один объект-клиент вместо typedef (допустимо, но shared module slot проще сохранить совместимость `OfflineTranslateFn`).

### D3 — Отзывчивость UI на cold path (критично для TO-11)

**Выбор:** не выполнять синхронно на UI isolate всю цепочку «load ~230MB + `Uint8List.fromList` + FRB encode» без yield кадров.

Практический минимум (native) — зафиксированный порядок:

1. После `setState(translating)` / старта warm-up — **явный yield кадра** (`await Future<void>.delayed(Duration.zero)` или post-frame), чтобы spinner успел отрисоваться.
2. `AssetBundle.load` по файлам — последовательные `await` (уже async); **крупный `Uint8List.fromList` / копии** — через `Isolate.run` / `compute`, не на UI isolate.
3. Передача в FRB `init_offline_translate` остаётся `Normal` async (worker pool) для ORT session build; не добавлять синхронный pre-encode на UI без yield. Полный zero-jank на encode гигантского `Vec<u8>` не обещаем новым Rust API (D8); цель TO-11 — **не полный freeze** (кадры/жесты принимаются), loading на кнопке наблюдаем.
4. Warm-up после первого кадра shell снижает частоту cold tap; оставшийся cold path всё равно проходит шаги 1–3.

Web: warm-up в worker уже вне UI thread; bridge MUST уметь стартовать `warmup` без translate; UI isolate только `postMessage` / await Promise.

**Альтернативы:** только warm-up без правки cold path (недостаточно, если tap раньше ready — снова freeze).

### D4 — UX на кнопке

**Выбор:** сохранить текущий паттерн: `CircularProgressIndicator` child на Translate, `onPressed: null` на время запроса; ждать shared init; без текста/баннера вне кнопки.

### D5 — Точки врезки (файлы)

| Зона | Файлы / роль |
|------|----------------|
| Start hook | `lib/app/shell.dart` — первый post-frame → platform `warmUp…` (один раз); escape hatch для tests (D1) |
| Facade | `lib/data/translate/offline_translate.dart` (+ native/web/platform) — export warm-up + **process-wide** shared ensure |
| Native | `offline_translate_native.dart` — не менять bytes→FRB контракт; module-level ensure для warm-up+translate; D3 yield/isolate на load/copy |
| Web | `offline_translate_web.dart`, `web/translate_bridge.js`, `web/translate_worker.js` — op `warmup` + общий `translatorPromise` |
| UI sheets | Reader/Vocab Translate — уже spinner; ждут shared ensure через тот же translate path; без нового chrome |
| Tests | `test/features/shell_test.dart` (+ другие mounts `AppShell`) — отключить auto warm-up / no-op, чтобы не грузить модель |
| Rust | **D8:** без API change; `init_offline_translate` / ready / translate as-is; **не** в `init_app`; verify-only `cargo test` |

### D6 — Ошибки warm-up

Тихий warm-up fail → не показывать SnackBar сам по себе. Последующий Translate либо повторит init, либо покажет kind (`engineUnavailable` / `internal`) как сейчас.

### D7 — Тесты

- Widget/facade: warm-up вызывается; Translate mid-init → loading затем ok/err (TO-09, TO-10).
- TO-11: по возможности fake slow init + pump frames / interaction; плюс ручная приёмка Web/Windows/Android.
- TO-12: ops note / короткий manual; автоматом — ready persists in-process после warm-up.

### D8 — Rust / FRB scope (без смены публичного API)

**Выбор:** в этом change **нет** обязательных правок `rust/src/api/translate.rs` и domain `rust/src/translate/`. Публичный контракт остаётся:

- `init_offline_translate(Vec<OfflineModelAsset>) -> Result<(), TranslateErrorKind>`
- `translate_text(...) -> Result<TranslationResult, TranslateErrorKind>`
- `is_offline_translate_ready() -> bool`
- wasm32 stubs → `EngineUnavailable`; тяжёлый ORT **не** в `#[frb(init)] init_app`
- FRB call mode `Normal` (worker pool) для init — ок; TO-11 закрывается на Dart (AssetBundle / copy / encode), не новым Rust API

Двойной native init предотвращает **Dart shared Future** (D2). Domain уже собирает ORT вне mutex слота, затем ставит engine; гонка двух параллельных `init_offline_translate` без shared future — last-wins / лишняя работа, не новый контракт. Не вводить OS-path / disk-cache FRB в этом change.

Регрессия: при apply достаточно убедиться, что `cargo test` в `{app}/rust` по translate не сломан (verify-only), без codegen ради этого change.

Чеклист реализации — `tasks.md`.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Warm-up жрёт CPU/RAM сразу после старта | После первого кадра; тихий fail; модель и так нужна для Translate |
| Tap до конца warm-up → снова тяжёлый path | Shared future + D3 (не блокировать UI) |
| Web: двойной pipeline при гонке | Один `translatorPromise` + warmup op |
| Пик памяти native (Dart copy + ORT) | Принять в v1; disk-path out of scope |
| Android low-end OOM на warm-up | Fail → EngineUnavailable на Translate; не crash UI |

## Migration Plan

Нет миграции данных. Выкат: обычный app update; поведение улучшается in-process. Rollback: убрать start hook / warmup op — снова только lazy (хуже UX, без schema break).

## Open Questions

Нет. Хук warm-up зафиксирован на **первом post-frame `AppShell`** (D1); shared ensure — **process-wide** (D2); test escape hatch обязателен.
