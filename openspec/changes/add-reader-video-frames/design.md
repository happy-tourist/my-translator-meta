## Context

Reader (`lib/features/reader/reader_screen.dart`) показывает cues + optional split; AppBar сейчас только toggle split. Persist документов — Drift (`lib/data/library/`, `lib/data/database/`). Decode видео в проекте нет; Rust FRB для video **не** вводим (решение explore: только Dart).

Explore зафиксировал: 3 кадра/страница; сэмплинг каждая секунда; P1 persist (JPEG + durable link); sync только сверху вниз; remove в chrome.

## Goals / Non-Goals

**Goals:**

- Полоса кадров над cues с pager; lazy extract; текст cue по покрытию интервала.
- Native + Web через один Dart-фасад extract.
- Persist привязки и кэша кадров у оригинала; cascade delete; re-pick при мёртвой ссылке.
- Не ломать tap/highlight/split.

**Non-Goals:**

- FFmpeg / unbundle в `rust/`; WASM WebCodecs.
- Копирование всего видео в sandbox.
- Аудио / плеер / `visibility_cached_image`.

## Decisions

### D1 — Слой: только Flutter (+ Drift), без Rust video API

- **Выбор:** вся оркестрация и decode в Dart; FRB codegen для video не трогаем.
- **Альтернатива:** `unbundle` в Rust — отвергнуто (линковка FFmpeg в cdylib).
- **Implementability / Rust boundary:** это **намеренный FE-only compute exception** (оба native и web в Dart), по духу как Web Transformers.js за translate-facade — не «забытый BE». Канон «тяжёлый NLP/parse/translate native → Rust» **не** распространяется на video frames в этом change. User persist video link/cache → Drift (канон `library/documents`), не `rust/src/db`.
- **Не менять в `rust/`:** `api/parse`, `api/tokenize`, `api/translate`, `Cargo.toml` deps, FRB surface — VF-12 / LD-10 опираются на уже существующие API без новых сигнатур.

### D2 — Фасад extract + platform backends

```
Reader UI
  -> VideoAttachmentController (session)
  -> VideoFrameExtractor (abstract)
        |-- NativeFfmpegFrameExtractor  (FFmpegKit Dart)
        \-- WebCanvasFrameExtractor     (package:web / canvas)
  -> VideoFrameCacheStore (JPEG files / web blobs)
  -> DocumentRepository (meta: link, durationSec, lastPage?)
```

- **API фасада (логический):** `probeDuration(ref) -> Duration`; `extractJpegAtSecond(ref, second) -> Uint8List`.
- **Web:** скрытый `<video>` + seek + canvas `toBlob`/data URL → bytes; по аналогии с platform-исключением `lib/data/translate/` (`offline_translate_platform.dart` conditional import).
- **Native:** FFmpegKit-family Dart API: `-ss <sec> -i <path> -frames:v 1 -q:v 2` (или эквивалент) → JPEG во временный/кэш-файл → bytes.
- **Native path contract (FE implementability):** FFmpegKit MUST получать **filesystem path**, не сырой `content://`. После `file_picker`: если `PlatformFile.path` доступен и читаем — использовать его (и сохранить как durable ref где ОС позволяет); иначе скопировать байты/`xFile` в app sandbox (`path_provider`, keyed by `documentId`) и probe/extract по sandbox path. Durable meta MAY хранить исходный URI **и/или** sandbox path — extract всегда идёт по usable path.

### D3 — Пакет FFmpeg на native

- **Intent пользователя:** «ffmpeg-kit-next».
- **Факт:** официальный `ffmpeg-kit-next` (arthenica) **не** публикует готовый пакет на pub.dev — нужна локальная сборка (Nix) и path/git integration.
- **Выбор для apply:** тонкий backend за фасадом; **первая попытка** — path/module `ffmpeg-kit-next` Flutter, если сборка артефактов доступна в окружении агента/CI; **иначе** pin pub.dev `ffmpeg_kit_flutter_new` (предпочтительно минимальный/video variant с H.264 + типичный MKV, не full GPL без нужды). Лицензия и размер APK — осознанный trade-off (explore: ок).
- **Не** смешивать два нативных FFmpeg в одном билде.

### D4 — UI Reader

- **Точки:** `reader_screen.dart` AppBar `actions`: Load video / Remove video **рядом** со split toggle (`readerSplitToggle`); body — `Column`: `VideoFrameStrip` (если attachment) → существующий single/split body.
- **Remount boundary:** Show/Hide split сегодня remount’ит original list (`_buildSplitBody` / `_buildSinglePane`). `VideoFrameStrip` MUST жить **снаружи** этой ветки (общий `Column` parent), чтобы Show/Hide **не** сбрасывал `currentPage` / in-flight extract (VF-07).
- Новый UI-хелпер рядом с reader (например `video_frame_strip.dart`, `cue_text_at_second.dart` pure).
- Страница `p` (1-based): секунды `3*(p-1) .. min(3*p, totalSeconds)-1`; total pages = `ceil(totalSeconds / 3)` при `totalSeconds >= 1`; при `totalSeconds < 1` после probe → VF-11 (strip не показывать).
- Ошибка слота → placeholder; ошибка pick/probe → SnackBar (как `readerTranslationErrorSnack` / library open errors). Injectable extractor/picker для widget-тестов (как `filePicker` / `offlineTranslate`).

### D5 — Текст под кадром

- Pure: среди `originalCues` найти cue с `startMs <= t*1000 < endMs` (`CueDto.startMs`/`endMs` как `BigInt`); иначе пусто. При нескольких — первая по порядку списка.
- Display-текст: join `CueDto.textLines` через `\n` (как cues-first в Reader до tokenize) — не изобретать другой join.

### D6 — Sync сверху вниз

- Смена страницы кадров → `jumpTo` / nearest на `_originalScrollController` (и bottom, если `_splitEnabled` + translation cues) по времени первой секунды страницы (reuse существующий `nearestCueIndex` + `BigInt.from(pageFirstSecond * 1000)`).
- Show split → существующий `_toggleSplit` sync bottom к captured original anchor; дополнительно: если strip виден, якорь времени полосы MAY уточнять sync (не сбрасывать strip page — D4 remount boundary).
- Scroll cues / bottom → **не** менять `currentPage` strip (не слушать `ItemPositionsListener` для strip page).

### D7 — Persist P1 (не копия видео)

- Drift на `Documents` (или satellite table `document_video_attachments`): `video_uri` / platform bookmark string, optional sandbox extract path, `duration_sec`, optional `last_frame_page`. Schema bump с текущей **v4**.
- JPEG кэш: native — файлы в app sandbox (`path_provider`) keyed by `documentId` + `second`; Web — OPFS или Drift BLOB table для JPEG (не для полного видео). Метаданные ссылки на Web — File System Access handle id / stored permission где возможно; иначе **cache-only restore + re-pick** (LD-22) — это ожидаемо, не blocker.
- `DocumentRepository`: `attachVideo`, `detachVideo`, `getVideoAttachment`, `putFrameCache`, `getFrameCache`, cascade в `deleteDocument` (рядом с cascade translation). Повторный `attachVideo` при уже существующей привязке MUST заменить link/duration и очистить JPEG-кэш предыдущего файла (LD-23).
- **Library → Reader:** open (`library_screen.dart` `_onDocumentTap`) best-effort `getVideoAttachment(doc.id)` (fail MUST NOT block open, как translation/anchor); передать meta в `ReaderScreen` (новый optional ctor field **или** Reader грузит по `originalDocumentId` через уже injected `documentRepository`). JPEG кэш не обязательно preload’ить целиком — lazy через cache store.

### D8 — Форматы pick

- `file_picker` с фильтром video (`FileType.custom` extensions `mp4`/`mkv` или `FileType.video` + post-filter); принимать расширения как минимум `.mp4`, `.mkv` (и типичные алиасы при необходимости). Soft-fail на кодек, который backend не декодирует → VF-10/VF-11.
- Отдельный injectable `ReaderVideoPicker` (не смешивать с translation `filePicker`), чтобы тесты VF-01/VF-11 не трогали split load.

### D9 — Тесты

- Unit: page math, cue-at-second, nearest sync helpers.
- Widget: strip visibility, pager, remove, cue scroll isolation, split show sync — с fake `VideoFrameExtractor` / fake cache (injectable), без реального FFmpeg в CI.
- Repository unit: attach/detach/cascade/dead-link + cache.

Чеклист реализации — `tasks.md`.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| ffmpeg-kit-next не на pub.dev / Nix недоступен | Фасад + fallback `ffmpeg_kit_flutter_new`; зафиксировать выбранный artifact в PR |
| Android `content://` / SAF без path | D2: sandbox copy → extract path; не кормить FFmpeg сырым content URI |
| Seek медленный / keyframe drift | Placeholder на слоте; JPEG quality умеренный; кэш посещённых секунд |
| Durable URI протухает | LD-22: cache остаётся; UI re-pick |
| Web: нет durable handle | D7: cache-only + re-pick; LD-19 «если link действителен» |
| Web квота на много JPEG | Только visited seconds; optional cap позже (out of scope) |
| APK size | Минимальный codec pack, не full |
| Split + strip layout тесный на phone | Strip компактный (thumb height); не дублировать кадры внизу |
| Show/Hide split remount сбрасывает strip | D4: strip вне remount branch |

## Migration Plan

- Drift schema bump + migration для video meta / frame cache.
- Существующие документы без video — поведение без изменений.
- Rollback: drop columns/table + убрать UI actions (feature isolated).

## Open Questions

- Нет продуктовых. При apply: подтвердить конкретный pub/path artifact FFmpegKit (D3) по результатам первой сборки.
