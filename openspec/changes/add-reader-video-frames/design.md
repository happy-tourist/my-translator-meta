## Context

В app уже есть: Dart-фасад extract (`lib/data/video/`), Drift schema v5 + JPEG BLOB cache, AppBar load/remove, и **infinite** `VideoSecondTimeline` (все секунды + timestamp + текст по interval-cover). Explore (2026-10-13): продукт — **replace** original на **страницы по 3 кадра в столбец** + стрелки; текст по **округлянному концу** cue; **без** timestamp.

Инфра extract/persist/picker **переиспользуется**; presentation + text math + pager переписываются.

## Goals / Non-Goals

**Goals:**

- Replace cue list → вертикальная страница из 3 кадров (`t, t+1, t+2`) + prev/next.
- Кадр на каждую целую секунду видео; lazy extract текущей страницы + cache.
- Текст под кадром `t`: все cues с `round(endMs/1000) == t` (join через `\n`); иначе пусто; timestamp не показывать.
- Load/restore → страница, содержащая секунду reading time; Remove → cue list @ том же времени.
- Split: кадры только original; original page/second → sync bottom; bottom ↛ original.
- Без нового Rust/FRB video API.

**Non-Goals:**

- Горизонтальная полоса 3-в-ряд над cues.
- Infinite scroll всех секунд.
- FFmpeg в `rust/`; WASM WebCodecs.
- Кадры в нижней панели split; аудио / плеер.

## Decisions

### D1 — Слой: только Flutter (+ Drift), без Rust video API

- Без изменений: decode/orchestration в Dart; persist Drift; `rust/` / FRB для video не трогаем.

### D2 — Фасад extract + platform backends

- Без изменений API: `probeDuration`, `extractJpegAtSecond`; native FFmpeg / web canvas; usable path (не сырой `content://`).

### D3 — Пакет FFmpeg на native

- **Закрыто:** `ffmpeg_kit_flutter_new_video: ^2.5.7`. Не смешивать второй native FFmpeg.

### D4 — UI Reader: paged vertical triple (REVISION 2)

- **Было (rev1):** virtualized list всех секунд (`VideoSecondTimeline`).
- **Стало:** при attachment original body — **одна страница** из трёх слотов в **столбец** + стрелки pager. Cue list **заменён**, не дублируется под страницами.
- Page math: секунды `0..durationSeconds-1`; page size = 3; page index / last incomplete page — как у прежнего page-of-3 helpers (восстановить `video_frame_page_math` или эквивалент).
- Remount Show/Hide split: page state / контроллеры MUST переживать remount (ключ вне dispose-ветки).
- Ошибка слота → placeholder кадра; pick/probe fail → SnackBar (VF-11). Injectable extractor/picker/cache для тестов.
- **Не** возвращать горизонтальный `VideoFrameStrip` над cues.

### D5 — Текст под кадром: end-second (REVISION 2)

- Секунда текста для cue: `attachmentSecond = round(endMs / 1000)` (обычное арифметическое округление; clamp в `[0, durationSeconds-1]` при показе, если нужно).
- Под кадром `t`: **все** cues с этим `attachmentSecond`, в порядке списка оригинала; display = join `text_lines` через `\n`, cue между собой тоже через `\n` (или эквивалент «оба текста видны»).
- Cue **не** рисуется под кадрами start/middle — только под концом.
- Пустой текст → кадр остаётся; **timestamp / time label под кадром MUST NOT** показываться.

### D6 — Sync

```
reading position
      |
      v  on Load / restore
 jump to page containing second T
      |
      v  user changes page (arrows)
 sync translation (if split) via nearestCueIndex(T*1000)
      |
 user scrolls translation
      X  do NOT change page (video on)
```

- **T при Load:** `clamp(floor(readingTimeMs/1000), 0, duration-1)` → открыть страницу, содержащую T.
- **Remove:** capture current page/second → restore cue list → `scrollTo` nearest cue at `T*1000`.
- **Split on:** выровнять bottom к текущей секунде страницы; не сбрасывать page.

### D7 — Persist P1

- Schema v5: video meta + `DocumentVideoFrames` BLOB — без обязательного bump.
- Якорь: `videoLastFramePage` как **1-based page** снова **или** хранить last_second и вычислять page при restore — зафиксировать в apply одно; предпочтительно **last_second** (уже в коде) → page = f(second).
- JPEG cache keyed by `documentId` + `second`. LD-22/LD-23 без изменений.

### D8 — Форматы pick

- Без изменений: MP4/MKV, VF-11.

### D9 — Тесты (REVISION 2)

- Unit: page math; `round(endMs/1000)` attachment; multi-cue same end; no timestamp when empty.
- Widget: VF-* под paged column; jump page; remove restore; split one-way.
- Repository LD-* остаются.

Чеклист — `tasks.md` §8 (новый scope); 1–7 = prior ship, superseded for presentation.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Путаница strip vs column | Spec/skills: явно «столбец 3 + стрелки», не horizontal strip |
| endMs exclusive half-open vs round | Продукт: round(endMs/1000); unit tests edge 51000 / 51700 |
| Infinite list код / тесты | Заменить или удалить; не оставлять dual UX |
| Remount split | Page state вне dispose-ветки |

## Migration Plan

- UI rewrite поверх extract/Drift; пользователи с infinite list получат paged column.
- Persist JPEG совместим; якорь second→page.
- Rollback: cue-only body.

## Open Questions

- Нет продуктовых. Apply: точный widget name / reuse page math helpers; join separator для multi-cue.
