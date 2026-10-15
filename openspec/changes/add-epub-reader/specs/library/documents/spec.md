# library/documents (delta) — EPUB import / open

Связанные UI/compute: `parse/epub`, `reader/chapters`, `reader/split`. Delta расширяет import/open на EPUB; subtitle-потоки и video attachment сохраняются.

**Слой:** persist bytes — Flutter Drift (+ sandbox; Web — binary-safe per design D6: base64 в `Documents.content` или Blob companion) в `lib/`; парсинг EPUB — FRB `parse/epub`; субтитры — существующий `parse/subtitles`.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| LD-01 | Import valid subtitle file | unchanged intent |
| LD-07 | Import ASS/SSA file | unchanged |
| LD-02 | Reject import without folder | unchanged |
| LD-03 | Reject unsupported file type | unit: reject non subtitle/epub; accept .epub |
| LD-24 | Import valid EPUB file | unit/widget: .epub → document format epub in folder |
| LD-10 | Open document opens Reader after successful parse | subtitle path unchanged |
| LD-25 | Open EPUB opens Reader in book mode | Flutter: valid epub → Reader chapters/paragraphs |
| LD-16 | Open original loads linked translation for Reader | subtitle unchanged; epub loads linked epub translation |
| LD-26 | Reject non-EPUB translation link for EPUB original | unit: EPUB original MUST NOT keep/open SRT as translation |
| LD-11 | Failed open stays in Library with visible error | also for failed EPUB parse |
| LD-19 | Open original restores video attachment for Reader | subtitle-only; EPUB open MUST NOT require video |
| LD-27 | Web EPUB bytes roundtrip binary-safe | unit: store/load epub bytes on Web path → identical ZIP bytes for parse |
| LD-28 | EPUB reading anchor is chapter index | unit/widget: save/load lastReadStartMs as chapter index for format epub |

## MODIFIED Requirements

### Requirement: Import subtitle into folder
Система SHALL позволять импортировать файл субтитров (SRT, VTT или ASS/SSA) **или EPUB** только внутрь существующей папки и сохранять документ в локальном хранилище библиотеки. Поддерживаемые расширения: `.srt`, `.vtt`, `.ass`, `.ssa` (`.ssa` хранится как format `ass`), `.epub` (хранится как format `epub`).

#### Scenario [LD-01]: Import valid subtitle file
- **GIVEN** существует папка «S01»
- **WHEN** пользователь выбирает валидный `.srt` файл с устройства и импортирует его в «S01»
- **THEN** в «S01» появляется документ с узнаваемым названием (из имени файла, если иное не задано)

#### Scenario [LD-07]: Import ASS/SSA file
- **GIVEN** существует папка «S01»
- **WHEN** пользователь импортирует файл `.ssa` (или `.ass`)
- **THEN** документ сохраняется с format `ass` и виден в папке

#### Scenario [LD-24]: Import valid EPUB file
- **GIVEN** существует папка «S01»
- **WHEN** пользователь импортирует валидный `.epub` файл
- **THEN** документ сохраняется с format `epub` и виден в папке

#### Scenario [LD-02]: Reject import without folder
- **GIVEN** пользователь находится в корне библиотеки (список папок)
- **WHEN** он пытается импортировать файл без выбора папки
- **THEN** система не создаёт документ в корне и требует выбрать папку (import CTA отсутствует или явно блокируется в корне)

#### Scenario [LD-03]: Reject unsupported file type
- **GIVEN** пользователь выбрал папку
- **WHEN** он выбирает файл неподдерживаемого типа (не SRT/VTT/ASS/SSA/EPUB)
- **THEN** система не сохраняет документ и сообщает в UI, что формат не поддерживается

### Requirement: Open document from folder into Reader
Система SHALL при выборе видимого документа в папке загружать его содержимое из локального хранилища библиотеки и парсить по формату: субтитры → `parse/subtitles` → Reader с cues; EPUB → `parse/epub` → Reader в book-режиме (`reader/chapters`). Если у документа есть связанный сохранённый перевод, система SHALL загрузить и распарсить перевод тем же семейством формата, что оригинал (субтитры↔субтитры, EPUB↔EPUB), и передать в сессию Reader для нижней панели split. Если у документа сохранён якорь чтения, система SHALL передать его в сессию Reader. Если у **subtitle**-документа есть привязка видео, система SHALL передать её для `reader/video-frames`. Для EPUB video attachment не требуется. Import по-прежнему не обязан валидировать содержимое; проверка оригинала выполняется при открытии. Сбой загрузки/парсинга **перевода** или недоступность видео-ссылки при успешном оригинале MUST NOT блокировать открытие Reader с оригиналом.

#### Scenario [LD-10]: Open document opens Reader after successful parse
- **GIVEN** в папке есть импортированный документ с валидным содержимым субтитров (хотя бы одна реплика) без перевода
- **WHEN** пользователь выбирает этот документ
- **THEN** открывается Reader с репликами документа

#### Scenario [LD-25]: Open EPUB opens Reader in book mode
- **GIVEN** в папке есть импортированный EPUB, успешно парсящийся хотя бы в одну главу с абзацем, без перевода
- **WHEN** пользователь выбирает этот документ
- **THEN** открывается Reader в book-режиме с главами/абзацами документа

#### Scenario [LD-16]: Open original loads linked translation for Reader
- **GIVEN** оригинал имеет валидный связанный перевод в хранилище того же семейства формата
- **WHEN** пользователь открывает оригинал из папки
- **THEN** Reader открывается с содержимым оригинала, и перевод доступен для нижней панели split без повторного выбора файла с устройства

#### Scenario [LD-19]: Open original restores video attachment for Reader
- **GIVEN** subtitle-оригинал имеет сохранённую привязку видео и опционально кэш кадров
- **WHEN** пользователь открывает этот оригинал из папки
- **THEN** сессия Reader получает данные привязки (и кэш при наличии), так что paged vertical frames могут быть восстановлены без обязательного повторного выбора файла, если durable link / usable extract path ещё действителен; сбой загрузки video meta MUST NOT блокировать open оригинала (как у перевода / reading anchor)

### Requirement: Failed open shows error without leaving Library
При ошибке загрузки содержимого, парсинга (субтитры или EPUB), encoding, нуля реплик или нуля глав/абзацев система SHALL оставаться в Library (в текущей папке), SHALL показать пользователю заметную ошибку (SnackBar / эквивалент) и MUST NOT открывать Reader.

#### Scenario [LD-11]: Failed open stays in Library with visible error
- **GIVEN** в папке есть импортированный документ, содержимое которого не парсится успешно (включая EPUB без текста / битый файл)
- **WHEN** пользователь выбирает этот документ
- **THEN** пользователь остаётся в Library, видит ошибку открытия, и Reader не открывается

## ADDED Requirements

### Requirement: EPUB original links only EPUB translation
Для оригинала с format `epub` система MUST принимать и сохранять в слоте перевода только документ format `epub`. Попытка связать subtitle-файл как перевод EPUB MUST быть отклонена с заметной ошибкой; существующая subtitle↔subtitle связь не затрагивается.

#### Scenario [LD-26]: Reject non-EPUB translation link for EPUB original
- **GIVEN** Reader открыт для EPUB-оригинала, split включён, нижняя панель пуста
- **WHEN** пользователь выбирает файл субтитров как перевод
- **THEN** перевод не сохраняется как linked document, пользователь видит ошибку несовместимого формата, оригинал остаётся открытым

### Requirement: Binary-safe EPUB persistence on Web
На Web система MUST сохранять и загружать байты EPUB без UTF-8 text roundtrip, который портит ZIP (design D6: base64 в `Documents.content` format-gated, либо Blob companion). После import→load байты MUST быть пригодны для успешного `parse_epub` того же файла. Native sandbox path остаётся binary file I/O.

#### Scenario [LD-27]: Web EPUB bytes roundtrip binary-safe
- **GIVEN** валидные байты `.epub` импортированы на Web (или в Web-режиме content store)
- **WHEN** система загружает bytes документа для open/parse
- **THEN** полученные байты совпадают с исходными (или иначе успешно парсятся тем же `parse_epub`), а не являются результатом `utf8.decode`/`utf8.encode` поверх сырого ZIP

### Requirement: EPUB reading anchor stores chapter index
Для документа format `epub` система SHALL сохранять и восстанавливать якорь чтения как **0-based индекс главы** в существующей колонке `lastReadStartMs` (format-gated семантика; design D6). Для subtitle-документов `lastReadStartMs` остаётся cue start ms. Resume абзаца внутри главы в v1 не требуется. Сбой якоря MUST NOT блокировать open.

#### Scenario [LD-28]: EPUB reading anchor is chapter index
- **GIVEN** пользователь читал EPUB на главе k (0-based) и якорь сохранён
- **WHEN** документ снова открывается
- **THEN** Reader получает якорь с chapter index k (через `ReadingAnchor` / эквивалент) и показывает главу k (`reader/chapters` RC-06), без интерпретации значения как subtitle `startMs`
