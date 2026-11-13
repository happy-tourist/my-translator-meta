## Context

См. `proposal.md` — Why / Scope. Уже в коде: process-wide ensure/warmUp, **fullscreen gate** в `AppShell`, native **extract → app-local dir → `init_offline_translate_from_dir`** (D8 revised / D13; bytes FRB MAY for tests), Web worker warmup, Reader zero-sync / bottom anchor / top-linked pager / AppBar delete.

Explore (до path-fix): на **native** gate-spinner был «мёртвый» — UI isolate занят ~228MB load/copy + Dart→Rust encode. Пользователь: **оставить лоадер**; вариант B — доработать cold path. Path/file init сильнее, чем ещё меньше chunk yields при полном FRB bytes.

Платформы: Web (Chrome), Windows, Android. Фокус freeze-fix: **native**.

## Goals / Non-Goals

**Goals:**

- Fullscreen loading после Drift open до ready/fail; затем shell (**gate остаётся**).
- Shared process-wide ensure; Translate wait + spinner.
- **Native:** во время gate `CircularProgressIndicator` остаётся живым (кадры) — не блокировать UI isolate полным bytes transfer модели.
- Reader: zero sync; bottom persist; top-only linked pager; text-only bottom; Delete у split (уже).

**Non-Goals:**

- Убрать gate / тихий-only product default.
- ORT внутри `init_app` / `RustLib.init`.
- Persist split/divider; mid-chapter persist оригинала EPUB сверх LD-28.
- Realtime scroll mirror; video sync.
- Менять Web pipeline (Transformers.js worker) кроме регрессии.

## Decisions

### D1 — Fullscreen loading gate (подтверждено)

После Drift open — fullscreen loading до ensure ready/fail. Shell недоступен до снятия. Fail → dismiss без обязательного SnackBar. Escape hatch для тестов.

**Explore:** предложение убрать gate **отклонено** («оставь лоадер»).

### D2 — Shared init future

Без изменений: один process-wide Future на native+Web.

### D3 — Cold path yields (недостаточен один)

Прежний bytes+chunked yield **не** закрывает TO-11 на реальных ~200MB. Оставляем yields где ещё нужны (копия на диск), но главный фикс — D8/D13 (path), не «ещё мельче chunks» как единственная мера.

### D4 — Translate button UX

Без изменений: spinner на кнопке при wait.

### D5 — Точки врезки (обновлено)

| Зона | Файлы / роль |
|------|----------------|
| Gate | `lib/app/shell.dart` — fullscreen до ensure (без смены UX) |
| Native facade | `lib/data/translate/offline_translate_native.dart` (+ helpers): extract assets → app support/cache dir с yield; вызвать path/dir FRB init |
| FRB / Rust | `rust/src/api/translate.rs` + `translate/`: init from OS paths / model dir; ORT `Session` from files; codegen |
| Web | без обязательных правок worker |
| Reader / Drift | уже по D9–D12 |
| Tests | TO-11: живой loader при реальном/приближённом heavy init; Rust path-init tests; shell gate регрессия |

### D6 — Warm-up / gate errors

Без изменений: fail → dismiss gate; Translate показывает kind.

### D7 — Тесты

- TO-09/13 gate; TO-10; **TO-11 живой индикатор на native cold path**; TO-14 path init; RS/RC/LD как раньше.

### D8 — Rust / FRB (**REVISED**)

**Было:** verify-only; только `Vec<{name,bytes}>`; запрет OS-path.

**Стало:** для **native product cold path** публичный FRB SHALL принимать **пути к файлам модели** (или один directory + фиксированные basenames), чтобы не гонять ~200MB через Dart UI isolate. Bytes-init (`init_offline_translate` с bytes) MAY остаться для unit-тестов / compat, но **product ensure/warm-up MUST NOT** полагаться на полный bytes transfer как единственный путь.

Не класть ORT в `init_app`. Web по-прежнему не вызывает FRB translate.

### D9–D12 — Reader (без изменений intent)

Zero one-shot sync; top-linked chapter; translation reading anchor; bottom text-only + Delete у split — уже в коде.

### D13 — Native extract then path init (NEW)

**Выбор:**

1. Flutter (после первого paint gate) последовательно читает/копирует required model files из AssetBundle в app-local dir (`getApplicationSupportDirectory` / эквивалент), с **chunked write + yields** (или нативный copy API, если уже в deps — не плодить лишние пакеты без нужды).
2. Если целевые файлы уже на месте и валидны (size/mtime или marker) — **пропустить** повторную полную копию (лёгкий disk reuse в рамках сессии/установок; не отдельный «product disk-cache» feature).
3. Вызвать FRB init с paths/dir → Rust ORT load from filesystem (async FRB worker; без гигантского Dart→Rust encode).
4. Shared ensure по-прежнему один; gate ждёт тот же Future.

**Отклонено как единственный фикс:** только уменьшать `_kAssetCopyChunkBytes` при полном FRB bytes — не снимает encode stall.

**Альтернатива (отклонена сейчас):** убрать gate и пустить shell под jank — пользователь оставил лоадер.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| FRB API break / dual init | Явный path API или расширенный контракт; bytes для тестов; codegen + skills update |
| Disk space ~200MB в support dir | Accept; reuse files; document |
| Copy всё ещё jankит | Chunked write + yields; измерять vs bytes+FRB encode |
| Gate всё ещё долгий | Честный loader; живая анимация важнее короткого dead screen |
| Web регрессия | Не трогать worker без нужды; smoke `:5555`+COOP/COEP |

## Migration Plan

- Существующий gate/ensure оставить; заменить native `_loadAndInit` bytes-path на extract→path.
- Старые installs: первая сессия после обновления — extract; далее reuse.
- Rollback: временно вернуть bytes init (хуже UI).
- Reader delta уже в коде — без миграции UX.

## Open Questions

Нет — gate оставлен; path init выбран для native B.
