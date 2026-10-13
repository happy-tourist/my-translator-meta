# library/documents (delta) — video attachment

Связанная UI-capability: `reader/video-frames`. Этот delta добавляет persist привязки видео и кэша кадров; не меняет import субтитров.

**Слой:** source of truth video meta + JPEG-кэш — Flutter Drift (+ sandbox FS / Web BLOB|OPFS) в `lib/`, как у документов библиотеки; **Rust/FRB storage API не требуется**. Парсинг cues при open по-прежнему через существующий FRB `parse_subtitles` (без нового video API).

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| LD-05 | Delete one document | unit: delete visible doc without translation/video still works (unchanged) |
| LD-15 | Delete original cascades translation | unit: delete original removes linked translation (unchanged) |
| LD-18 | Delete original cascades video attachment | unit: delete original clears video link + frame cache |
| LD-10 | Open document opens Reader after successful parse | Flutter widget: unchanged for docs without video |
| LD-16 | Open original loads linked translation for Reader | unchanged intent |
| LD-19 | Open original restores video attachment for Reader | Flutter widget/unit: open with video meta → Reader can show strip/cache |
| LD-20 | Persist video durable link and frame cache | unit: attach saves link + duration; cached JPEG for visited seconds survive restart |
| LD-21 | Remove video clears attachment and frame cache | unit: remove from Reader clears link + JPEG cache; original remains |
| LD-22 | Dead video link keeps cached frames and allows re-pick | unit/widget: dead link → cached frames usable; re-pick restores extract for new pages |
| LD-23 | Replace video clears previous frame cache | unit: attach while already linked → new link; previous JPEG cache cleared |

## MODIFIED Requirements

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
Система SHALL при выборе видимого документа в папке загружать его содержимое из локального хранилища библиотеки, запрашивать парсинг субтитров и при успехе открывать Reader с разобранными cues. Если у документа есть связанный сохранённый перевод, система SHALL также загрузить и распарсить перевод (при успехе) и передать его в сессию Reader для нижней панели split. Если у документа сохранён якорь чтения, система SHALL передать его в сессию Reader для восстановления позиции (см. `reader/split` RS-12 и requirement Persist reading anchor). Если у документа есть привязка видео (метаданные / durable link / кэш кадров), система SHALL передать их в сессию Reader для полосы кадров (`reader/video-frames`). Import по-прежнему не обязан валидировать содержимое; проверка оригинала выполняется при открытии. Сбой загрузки/парсинга **перевода** или недоступность видео-ссылки при успешном оригинале MUST NOT блокировать открытие Reader с оригиналом.

#### Scenario [LD-10]: Open document opens Reader after successful parse
- **GIVEN** в папке есть импортированный документ с валидным содержимым субтитров (хотя бы одна реплика) без перевода
- **WHEN** пользователь выбирает этот документ
- **THEN** открывается Reader с репликами документа

#### Scenario [LD-16]: Open original loads linked translation for Reader
- **GIVEN** оригинал имеет валидный связанный перевод в хранилище
- **WHEN** пользователь открывает оригинал из папки
- **THEN** Reader открывается с cues оригинала, и cues перевода доступны для нижней панели split без повторного выбора файла с устройства

#### Scenario [LD-19]: Open original restores video attachment for Reader
- **GIVEN** оригинал имеет сохранённую привязку видео и опционально кэш кадров
- **WHEN** пользователь открывает оригинал из папки
- **THEN** сессия Reader получает данные привязки (и кэш при наличии), так что полоса кадров может быть восстановлена без обязательного повторного выбора файла, если durable link / usable extract path ещё действителен; сбой загрузки video meta MUST NOT блокировать open оригинала (как у перевода / reading anchor)

## ADDED Requirements

### Requirement: Persist video attachment without copying full media
Система SHALL при успешной привязке видео из Reader сохранять на **оригинальном** документе: durable platform reference на выбранный файл (где ОС это позволяет), длительность в секундах (или эквивалент для пагинации) и кэш JPEG уже извлечённых секунд. Система MUST NOT требовать копирования всего видеофайла в хранилище субтитров. Привязка и кэш MUST переживать перезапуск приложения на поддерживаемых платформах. Система MUST сохранять JPEG посещённых секунд по мере ленивого извлечения. Повторная успешная привязка другого файла к тому же оригиналу SHALL заменить метаданные ссылки и MUST очистить кэш кадров предыдущего файла (см. UI `reader/video-frames`).

#### Scenario [LD-20]: Persist video durable link and frame cache
- **GIVEN** Reader открыт для оригинала, пользователь успешно привязал видео и открыл хотя бы одну страницу кадров
- **WHEN** приложение перезапускается и пользователь снова открывает тот же оригинал
- **THEN** метаданные привязки и JPEG уже посещённых секунд доступны сессии Reader

#### Scenario [LD-23]: Replace video clears previous frame cache
- **GIVEN** оригинал уже имеет привязку видео и кэш JPEG для части секунд
- **WHEN** пользователь успешно привязывает другой видеофайл к тому же оригиналу
- **THEN** метаданные указывают на новый файл, кэш кадров предыдущего файла очищен, оригинал и перевод (если есть) остаются

### Requirement: Remove video attachment from Reader clears storage
При удалении видео из chrome Reader система SHALL снять привязку с оригинала и MUST удалить кэш кадров этого документа. Оригинал и перевод (если есть) MUST остаться.

#### Scenario [LD-21]: Remove video clears attachment and frame cache
- **GIVEN** оригинал имеет привязку видео и кэш кадров
- **WHEN** пользователь удаляет видео в Reader
- **THEN** durable link и кэш кадров очищены, оригинал остаётся в папке

### Requirement: Dead video link keeps cache and allows re-attach
Если при open или при запросе новой страницы кадров durable link недоступен, система SHALL сохранить уже закэшированные JPEG и MUST позволять пользователю заново выбрать видеофайл (re-pick), восстанавливая возможность извлечения недостающих секунд. Уже сохранённые кадры MUST оставаться доступны для соответствующих страниц.

#### Scenario [LD-22]: Dead video link keeps cached frames and allows re-pick
- **GIVEN** оригинал имеет кэш кадров для части секунд, а durable link больше не действителен
- **WHEN** пользователь открывает документ и запрашивает страницу с закэшированными секундами, затем выполняет re-pick для новой страницы без кэша
- **THEN** закэшированные кадры показываются без ошибки «нет файла»; после успешного re-pick недостающие секунды снова извлекаются
