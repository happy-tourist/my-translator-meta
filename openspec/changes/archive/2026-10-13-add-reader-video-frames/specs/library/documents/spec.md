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
| LD-19 | Open original restores video attachment for Reader | Flutter widget/unit: open with video meta → Reader can show paged vertical frames/cache |
| LD-20 | Persist video durable link and frame cache | unit: attach saves link + duration; cached JPEG for visited seconds survive restart |
| LD-21 | Remove video clears attachment and frame cache | unit: remove from Reader clears link + JPEG cache; original remains |
| LD-22 | Dead video link keeps cached frames and allows re-pick | unit/widget: dead link → cached frames usable; re-pick restores extract for missing seconds |
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
Система SHALL при выборе видимого документа в папке загружать его содержимое из локального хранилища библиотеки, запрашивать парсинг субтитров и при успехе открывать Reader с разобранными cues. Если у документа есть связанный сохранённый перевод, система SHALL также загрузить и распарсить перевод (при успехе) и передать его в сессию Reader для нижней панели split. Если у документа сохранён якорь чтения, система SHALL передать его в сессию Reader для восстановления позиции (см. `reader/split` RS-12 и requirement Persist reading anchor). Если у документа есть привязка видео (метаданные / durable link / кэш кадров), система SHALL передать их в сессию Reader для **paged vertical frames** (`reader/video-frames`). Import по-прежнему не обязан валидировать содержимое; проверка оригинала выполняется при открытии. Сбой загрузки/парсинга **перевода** или недоступность видео-ссылки при успешном оригинале MUST NOT блокировать открытие Reader с оригиналом.

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
- **THEN** сессия Reader получает данные привязки (и кэш при наличии), так что paged vertical frames могут быть восстановлены без обязательного повторного выбора файла, если durable link / usable extract path ещё действителен; сбой загрузки video meta MUST NOT блокировать open оригинала (как у перевода / reading anchor)

## ADDED Requirements

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
