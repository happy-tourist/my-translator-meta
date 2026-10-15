# library/documents Specification

## Purpose

Позволяет импортировать файлы субтитров с устройства в выбранную папку библиотеки, хранить их локально и удалять вместе с метаданными, чтобы позже открывать в Reader. Source of truth документов — Flutter Drift (+ native sandbox FS / Web BLOB) в `lib/`; Rust/FRB storage API не требуется; парсинг cues — отдельная capability.

## Requirements

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

### Requirement: List documents in folder
Система SHALL показывать документы, принадлежащие выбранной папке, за исключением скрытых переводных документов, созданных через загрузку в нижней панели Reader и привязанных к оригиналу. Обычный import в папку по-прежнему SHALL делать документ видимым в списке.

#### Scenario [LD-04]: List documents after import
- **GIVEN** в папке «S01» импортированы два subtitle-документа обычным import
- **WHEN** пользователь открывает «S01»
- **THEN** оба документа видны в содержимом папки

#### Scenario [LD-12]: Hidden translation not listed
- **GIVEN** в папке есть видимый оригинал и связанный с ним перевод, загруженный через нижнюю панель Reader
- **WHEN** пользователь открывает эту папку в Library
- **THEN** в списке виден оригинал, а переводный документ не отображается как отдельная строка

### Requirement: Delete document
Система SHALL позволять удалить отдельный видимый документ; после удаления он не появляется в библиотеке, а связанные локальные данные документа недоступны. Если у удаляемого оригинала есть связанный скрытый перевод, система MUST удалить и этот перевод (cascade). Если у удаляемого оригинала есть привязка видео (durable link и/или кэш кадров), система MUST удалить и эти данные (cascade).

#### Scenario [LD-05]: Delete one document
- **GIVEN** в папке есть документ «episode1.srt» без связанного перевода и без видео
- **WHEN** пользователь удаляет этот документ
- **THEN** документ исчезает из списка папки, соседние документы и папки остаются

#### Scenario [LD-15]: Delete original cascades translation
- **GIVEN** оригинал имеет связанный скрытый переводный документ
- **WHEN** пользователь удаляет оригинал из Library
- **THEN** оригинал и связанный перевод удаляются из хранилища, и перевод не остаётся сиротой в папке

#### Scenario [LD-18]: Delete original cascades video attachment
- **GIVEN** оригинал имеет сохранённую привязку видео и/или кэш кадров
- **WHEN** пользователь удаляет оригинал из Library
- **THEN** привязка видео и кэш кадров этого документа удаляются и не остаются сиротами

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

### Requirement: Persist across app restart
Система SHALL сохранять импортированные документы и их привязку к папкам между перезапусками приложения на поддерживаемых платформах (включая Web / browser storage через Drift).

#### Scenario [LD-06]: Documents survive restart
- **GIVEN** пользователь импортировал документ в папку
- **WHEN** приложение полностью перезапускается (на Web — тот же origin) и пользователь снова открывает ту же папку
- **THEN** документ по-прежнему отображается в этой папке

### Requirement: Platform storage split for bytes
Система SHALL на native сохранять байты документа в app sandbox (path/key в метаданных) и SHALL на Web сохранять содержимое в Drift (BLOB/text), без зависимости от долговечного filesystem path.

#### Scenario [LD-09]: Web import stores content
- **GIVEN** приложение запущено на Web
- **WHEN** пользователь импортирует валидный subtitle в папку
- **THEN** документ доступен после reopen без опоры на путь исходного файла на диске пользователя

### Requirement: Persist Reader translation as hidden linked document
Система SHALL при успешной загрузке перевода из нижней панели Reader сохранять файл перевода в той же папке, что и оригинал, как полноценный документ хранилища (байты/format по тем же правилам platform storage, что и обычный import), и SHALL установить связь original→translation (один слот перевода на оригинал). Такой переводный документ MUST NOT появляться в списке папки (см. List documents). Связь MUST переживать перезапуск приложения на поддерживаемых платформах. Повторная загрузка поверх уже привязанного перевода без предварительного удаления не требуется (см. `reader/split`: delete, затем load).

#### Scenario [LD-13]: Persist translation linked to original
- **GIVEN** Reader открыт для оригинала из папки «S01», split включён
- **WHEN** пользователь успешно загружает переводный subtitle-файл в нижнюю панель
- **THEN** перевод сохраняется в хранилище папки «S01», связан с оригиналом, и после перезапуска приложения связь и содержимое доступны при повторном открытии оригинала

### Requirement: Delete translation from Reader unlinks and removes storage
При удалении перевода из нижней панели Reader система SHALL удалить связанный переводный документ из локального хранилища и MUST снять связь с оригиналом. Оригинал MUST остаться в папке.

#### Scenario [LD-14]: Delete translation unlinks and removes
- **GIVEN** оригинал имеет сохранённый связанный перевод
- **WHEN** пользователь удаляет перевод в нижней панели Reader
- **THEN** переводный документ и его байты удалены, связь снята, оригинал остаётся видимым в папке

### Requirement: Persist reading anchor for original document
Система SHALL хранить на **оригинальном** документе якорь последней позиции чтения: время начала якорной реплики (`start` в миллисекундах) и относительное alignment видимой строки (если известно). Система MUST обновлять якорь при завершении жеста прокрутки оригинала в Reader (settle) и при выходе из Reader (pop), если позиция известна. Якорь MUST переживать перезапуск приложения на поддерживаемых платформах. При открытии оригинала система SHALL отдавать сохранённый якорь в сессию Reader (если есть). Если cues после парсинга пусты или якоря нет — поведение как open без resume. При несовпадении сохранённого `start` с текущими cues потребитель (Reader) использует ближайшую реплику по времени. Persist режима split и divider не требуется. Скрытый translation-документ не обязан хранить отдельный якорь чтения.

#### Scenario [LD-17]: Persist and load reading anchor for original
- **GIVEN** оригинал открыт в Reader и пользователь остановился на реплике с известным `start` (не начало файла)
- **WHEN** якорь сохранён (после settle и/или выхода из Reader) и пользователь снова открывает этот оригинал из папки
- **THEN** в сессию Reader передаётся сохранённый якорь (`start` и alignment при наличии), и он доступен для восстановления позиции списка

### Requirement: Persist video attachment and durable media
Система SHALL при успешной привязке видео из Reader сохранять на **оригинальном** документе: durable platform reference (или sentinel для Drift-media), длительность в секундах, кэш JPEG уже извлечённых секунд и, для durable reopen, **копию медиа** (`DocumentVideoMedia`: bytes + fileName). На **Web** система MUST сохранять media bytes и MUST NOT полагаться на ephemeral `blob:` URL как sandboxPath (использовать sentinel вроде `drift-media`); при reopen — воссоздать session object URL из bytes. На **native** система MAY хранить sandbox filesystem path и MAY также сохранять media bytes. Система MAY сохранять `last_second` и/или page index (якорь paged UI). Привязка, кэш и media MUST переживать перезапуск на поддерживаемых платформах. Система MUST сохранять JPEG посещённых секунд по мере ленивого извлечения. Повторная успешная привязка другого файла к тому же оригиналу SHALL заменить метаданные ссылки / media и MUST очистить кэш кадров предыдущего файла (см. UI `reader/video-frames`).

#### Scenario [LD-20]: Persist video durable link and frame cache
- **GIVEN** Reader открыт для оригинала, пользователь успешно привязал видео и просмотрел хотя бы одну секунду на странице кадров (кадр извлечён/закэширован)
- **WHEN** приложение перезапускается и пользователь снова открывает тот же оригинал
- **THEN** метаданные привязки, JPEG уже посещённых секунд и (на Web) media bytes доступны сессии Reader для restore paged frames без обязательного повторного pick

#### Scenario [LD-23]: Replace video clears previous frame cache
- **GIVEN** оригинал уже имеет привязку видео, media и кэш JPEG для части секунд
- **WHEN** пользователь успешно привязывает другой видеофайл к тому же оригиналу
- **THEN** метаданные и media указывают на новый файл, кэш кадров предыдущего файла очищен, оригинал и перевод (если есть) остаются

### Requirement: Remove video attachment from Reader clears storage
При удалении видео из chrome Reader система SHALL снять привязку с оригинала и MUST удалить кэш кадров и media (`DocumentVideoMedia`) этого документа. Оригинал и перевод (если есть) MUST остаться.

#### Scenario [LD-21]: Remove video clears attachment and frame cache
- **GIVEN** оригинал имеет привязку видео, media и кэш кадров
- **WHEN** пользователь удаляет видео в Reader
- **THEN** durable link, media и кэш кадров очищены, оригинал остаётся в папке

### Requirement: Dead video link keeps cache and allows re-attach
Если при open или при запросе кадра секунды durable link / session playback ref недоступен, система SHALL сохранить уже закэшированные JPEG и MUST позволять пользователю заново выбрать видеофайл (re-pick), восстанавливая возможность извлечения недостающих секунд. Когда media bytes сохранены, система SHOULD восстановить playback ref из них без re-pick. Уже сохранённые кадры MUST оставаться доступны для соответствующих секунд.

#### Scenario [LD-22]: Dead video link keeps cached frames and allows re-pick
- **GIVEN** оригинал имеет кэш кадров для части секунд, а durable link больше не действителен
- **WHEN** пользователь открывает документ и запрашивает секунды с закэшированными кадрами, затем выполняет re-pick для секунд без кэша
- **THEN** закэшированные кадры показываются без ошибки «нет файла»; после успешного re-pick недостающие секунды снова извлекаются

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

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| LD-01 | Import valid subtitle file | unit/integration: import → row in folder |
| LD-02 | Reject import without folder | unit/widget: import to null/root rejected |
| LD-03 | Reject unsupported file type | unit: reject non subtitle/epub; accept .epub |
| LD-04 | List documents after import | unit/widget: two visible imports listed (unchanged intent) |
| LD-05 | Delete one document | unit: delete visible doc without translation/video still works (unchanged) |
| LD-06 | Documents survive restart | integration (close/reopen DB) + verify/ops Web same-origin restart |
| LD-07 | Import ASS/SSA file | unit: `.ssa`/`.ass` → format `ass` |
| LD-09 | Web import stores content | unit (fake platform) or verify/ops Web: content persisted |
| LD-10 | Open document opens Reader after successful parse | Flutter widget: tap valid doc without translation → Reader |
| LD-11 | Failed open stays in Library with visible error | also for failed EPUB parse |
| LD-12 | Hidden translation not listed | unit/widget: reader-attached translation absent from folder list |
| LD-13 | Persist translation linked to original | unit: attach creates stored doc + link in same folder |
| LD-14 | Delete translation unlinks and removes | unit: delete translation removes bytes/row and clears link |
| LD-15 | Delete original cascades translation | unit: delete original removes linked translation too |
| LD-16 | Open original loads linked translation for Reader | subtitle unchanged; epub loads linked epub translation |
| LD-17 | Persist and load reading anchor for original | unit/widget: save anchor → reopen → Reader gets same startMs (+ alignment) |
| LD-18 | Delete original cascades video attachment | unit: delete original clears video link + frame cache |
| LD-19 | Open original restores video attachment for Reader | subtitle-only; EPUB open MUST NOT require video |
| LD-20 | Persist video durable link and frame cache | unit: attach saves link + duration; cached JPEG for visited seconds survive restart |
| LD-21 | Remove video clears attachment and frame cache | unit: remove from Reader clears link + JPEG cache; original remains |
| LD-22 | Dead video link keeps cached frames and allows re-pick | unit/widget: dead link → cached frames usable; re-pick restores extract for missing seconds |
| LD-23 | Replace video clears previous frame cache | unit: attach while already linked → new link; previous JPEG cache cleared |
| LD-24 | Import valid EPUB file | unit/widget: .epub → document format epub in folder |
| LD-25 | Open EPUB opens Reader in book mode | Flutter: valid epub → Reader chapters/paragraphs |
| LD-26 | Reject non-EPUB translation link for EPUB original | unit: EPUB original MUST NOT keep/open SRT as translation |
| LD-27 | Web EPUB bytes roundtrip binary-safe | unit: store/load epub bytes on Web path → identical ZIP bytes for parse |
| LD-28 | EPUB reading anchor is chapter index | unit/widget: save/load lastReadStartMs as chapter index for format epub |
