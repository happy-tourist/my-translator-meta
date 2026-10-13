## Context

Первая реализация change уже в app: Dart-фасад extract (`lib/data/video/`), Drift schema v5 + JPEG BLOB cache, AppBar load/remove, **полоса 3-в-ряд** (`VideoFrameStrip`) над неизменным cue list. Explore-revision: полоса **не** соответствует продукту — нужен **replace** original pane вертикальным second-timeline с jump-on-load.

Инфра extract/persist/picker **переиспользуется**; presentation + sync rules переписываются.

## Goals / Non-Goals

**Goals:**

- Вертикальный поток: каждая секунда → кадр → текст|пусто + timestamp.
- При Load/restore: replace cue list; jump к `floor(readingTimeMs/1000)`.
- Remove → обычный cue list на том же времени.
- Кадры только в original; original→translation sync при split; bottom↛original при video.
- Lazy extract вокруг viewport; tap/highlight на тексте под кадром.
- Без нового Rust/FRB video API.

**Non-Goals:**

- FFmpeg / unbundle в `rust/`; WASM WebCodecs.
- Копирование всего видео в sandbox.
- Кадры в нижней панели split.
- Аудио / плеер.
- Decode всех секунд сразу.

## Decisions

### D1 — Слой: только Flutter (+ Drift), без Rust video API

- Без изменений относительно первой реализации: decode/orchestration в Dart; persist Drift; `rust/` / FRB surface для video не трогаем.

### D2 — Фасад extract + platform backends

- Без изменений API: `probeDuration`, `extractJpegAtSecond`; `NativeFfmpegFrameExtractor` / `WebCanvasFrameExtractor`; usable filesystem path (не `content://`).
- Пакет уже pinned: `ffmpeg_kit_flutter_new_video` (D3 fallback выполнен).

### D3 — Пакет FFmpeg на native

- **Закрыто:** `ffmpeg_kit_flutter_new_video: ^2.5.7`. Не смешивать второй native FFmpeg.

### D4 — UI Reader: interleaved second timeline (REVISION)

- **Было:** `Column(VideoFrameStrip?, cuesBody)` — strip 3-в-ряд + pager.
- **Стало:** при attachment original body — **один** virtualized list длиной `durationSeconds`; item `t` = frame + timestamp + optional cue text. Обычный cue list **скрыт/заменён**, не дублируется под timeline.
- Split: верх = timeline (или cues без video); низ = translation cues **без** кадров.
- Remount Show/Hide split: timeline state (current second / scroll) MUST переживать remount original list (ключ/контроллер вне ветки, которая dispose'ится зря) — аналог прежнего remount boundary.
- Ошибка строки → placeholder кадра; pick/probe → SnackBar. Injectable extractor/picker/cache для тестов.
- Удалить/перестать использовать pager page-of-3 math как UX (helpers MAY остаться мёртвыми до cleanup).

### D5 — Текст под кадром

- Без изменений: покрытие `[startMs, endMs)` → join `textLines` через `\n`; иначе пусто, строка кадра остаётся.

### D6 — Sync (REVISION)

```
reading position (cue scroll / anchor)
        |
        v  on Load / restore
   jump timeline to second T
        |
        v  user scrolls timeline
   sync translation (if split) via nearestCueIndex(T*1000)
        |
   user scrolls translation
        X  do NOT move timeline (video on)
```

- **Jump source при Load:** время видимого верха текущего cue list (или reading anchor, если list ещё не built) → `T = clamp(floor(ms/1000), 0, duration-1)`.
- **Remove:** capture current timeline second → restore cue list → `scrollTo` nearest cue at `T*1000`.
- **Split on:** выровнять bottom к текущей секунде; не сбрасывать T.
- **Отмена VF-08 старой модели** («cue scroll не двигает strip»): strip больше нет; при video list **и есть** timeline — скролл cues без video не применим; при video bottom не двигает original.

### D7 — Persist P1

- Schema v5 уже есть: video meta + `DocumentVideoFrames` BLOB.
- Предпочтительно хранить / писать **`last_second`** (или переиспользовать `lastFramePage` как second index с миграцией семантики — зафиксировать в apply: колонка `last_frame_page` MAY быть переименована/переиспользована как last second без нового bump, если совместимо; иначе bump v6).
- JPEG cache keyed by `documentId` + `second` — без изменений.
- LD-22/LD-23 heuristics (clear cache on replace; keep on same-duration re-pick) — сохранить.

### D8 — Форматы pick

- Без изменений: `ReaderVideoPicker`, mp4/mkv, VF-11.

### D9 — Тесты (REVISION)

- Unit: cue-at-second; second clamp/jump helpers (не page-of-3 как acceptance).
- Widget: VF-01…VF-13 с fake extractor; split sync one-way; remove restore.
- Repository unit: LD-* остаются.

Чеклист — `tasks.md` (секции 1–3/часть 6 — done; presentation rewrite — новые unchecked).

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Длинный список (часы × 3600 rows) | Virtualized list (`ScrollablePositionedList` / аналог); lazy extract + cache |
| Jump неточный (cue vs continuous time) | floor(visible cue startMs/1000); clamp |
| Remount split сбрасывает scroll | Контроллер/ключ timeline вне dispose-ветки |
| Старые widget-тесты strip | Переписать/удалить под VF-* revision |
| Семантика `last_frame_page` | Явно last_second в apply + тесты restore |

## Migration Plan

- UI rewrite поверх существующего extract/Drift; пользователи со strip UX получат interleaved после обновления.
- Persist: совместимый cache JPEG; якорь page→second.
- Rollback: вернуть cue-only body; meta/cache можно оставить.

## Open Questions

- Нет продуктовых. Apply: выбрать virtual list API уже в Reader (`ItemScrollController` / `ScrollablePositionedList`) и точную колонку last_second.
