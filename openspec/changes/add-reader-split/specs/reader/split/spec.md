## Purpose

Позволяет на экране Reader включить горизонтальное разделение: сверху оригинал субтитров, снизу догруженный переводный трек с синхронным скроллом по таймингу и пиксельной подстройкой, чтобы сверять перевод при чтении.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RS-01 | Toggle split shows empty bottom pane | Flutter widget: split on → bottom empty + load CTA |
| RS-02 | Hide split returns to single pane | Flutter widget: split off → one pane; translation kept in session |
| RS-03 | Resize divider changes pane ratio | Flutter widget: drag divider changes relative heights |
| RS-04 | Load translation shows cues in bottom | Flutter widget: successful load → bottom list like top |
| RS-05 | Sync after load by nearest start time | Flutter widget/unit: after load, bottom anchors to nearest `CueDto.startMs` of top anchor |
| RS-06 | Scroll sync time then pixel on settle | Flutter widget/unit: scroll end → time match then pixel settle; no continuous pixel fight during fling |
| RS-07 | Delete translation returns to empty | Flutter widget: delete → empty + load; file unlinked |
| RS-08 | Failed load shows error in bottom flow | Flutter widget: parse/load fail → visible error; bottom stays empty / no cues |
| RS-09 | Reopen restores translation in split | Flutter widget: reopen original with linked translation → bottom available after split |

## ADDED Requirements

### Requirement: Toggle horizontal split on Reader
Система SHALL позволять на экране Reader включать и выключать горизонтальное разделение экрана: верхняя область — cues оригинального документа; нижняя — область перевода. По умолчанию Reader MUST открываться в режиме одной панели (без split). При выключении split система MUST сохранять загруженный перевод в рамках текущей сессии Reader (не сбрасывать cues до явного удаления или выхода из Reader).

#### Scenario [RS-01]: Toggle split shows empty bottom pane
- **GIVEN** Reader открыт для оригинального документа без привязанного перевода
- **WHEN** пользователь включает режим split
- **THEN** экран делится горизонтально: сверху видны cues оригинала, снизу — пустое состояние с действием загрузки файла

#### Scenario [RS-02]: Hide split returns to single pane
- **GIVEN** split включён и в нижней панели уже загружены cues перевода
- **WHEN** пользователь выключает split
- **THEN** показывается только одна панель с оригиналом, приложение не падает, и при повторном включении split переводные cues снова доступны без повторной загрузки в той же сессии

### Requirement: Resizable split divider
Система SHALL позволять пользователю изменять относительную высоту верхней и нижней панелей перетаскиванием разделителя в режиме split. Сохранение ratio между перезапусками приложения не требуется.

#### Scenario [RS-03]: Resize divider changes pane ratio
- **GIVEN** Reader в режиме split с двумя панелями
- **WHEN** пользователь перетаскивает разделитель
- **THEN** относительные высоты верхней и нижней панелей изменяются соответственно жесту

### Requirement: Load translation into bottom pane
В пустой нижней панели система SHALL предоставлять действие загрузки файла субтитров с устройства (те же поддерживаемые типы, что и при import в Library). При успешном парсинге система SHALL показать cues перевода в нижней панели в том же представлении, что и оригинал сверху (время start→end и текст), и SHALL сохранить перевод как связанный документ библиотеки (поведение persist — capability `library/documents`). Замена файла без удаления не требуется: пока перевод загружен, система SHALL предоставлять удаление, а не отдельное «заменить». При ошибке загрузки/парсинга перевода система MUST показать заметную ошибку с той же классификацией `ParseErrorKind` (и load fail), что open-flow Library (тексты/ключи — design D5), без показа cues перевода.

#### Scenario [RS-04]: Load translation shows cues in bottom
- **GIVEN** split включён, нижняя панель пуста
- **WHEN** пользователь успешно загружает валидный файл субтитров с одной или более репликами
- **THEN** в нижней панели отображаются cues этого файла (время и текст), аналогично верхней панели

#### Scenario [RS-07]: Delete translation returns to empty
- **GIVEN** в нижней панели отображаются cues перевода
- **WHEN** пользователь удаляет перевод
- **THEN** нижняя панель возвращается в пустое состояние с действием загрузки, и связанный переводный документ больше не привязан к оригиналу

#### Scenario [RS-08]: Failed load shows error in bottom flow
- **GIVEN** split включён, нижняя панель пуста
- **WHEN** пользователь выбирает файл, который не удаётся загрузить или распарсить (включая ноль cues / `ParseErrorKind.Empty`)
- **THEN** пользователь видит заметную ошибку в контексте Reader / нижней панели с той же классификацией kind, что Library open (`ParseErrorKind` / load fail), cues перевода не показываются, и верхняя панель оригинала остаётся доступной

### Requirement: Synchronized scroll by time then pixel settle
В режиме split при наличии cues в обеих панелях система SHALL синхронизировать прокрутку так: (1) определить якорную реплику панели, получившей жест пользователя (или верхнюю панель при первом выравнивании после загрузки / включения split), (2) найти в другой панели реплику с ближайшим временем начала (`CueDto.startMs`, сравнение как `BigInt`), (3) прокрутить другую панель к этой реплике, (4) выполнить пиксельную подстройку только после settle (окончание жеста / programmatic jump), без непрерывного зеркалирования offset во время fling. Сразу после успешной первой загрузки перевода система MUST выполнить такое выравнивание относительно текущей позиции верхней панели.

#### Scenario [RS-05]: Sync after load by nearest start time
- **GIVEN** верхняя панель прокручена так, что якорная реплика имеет известный `start`, и нижняя панель пуста
- **WHEN** пользователь успешно загружает перевод с несколькими cues с разными `start`
- **THEN** нижняя панель выравнивается к реплике с ближайшим `start` к якорю верха

#### Scenario [RS-06]: Scroll sync time then pixel on settle
- **GIVEN** обе панели содержат cues и split включён
- **WHEN** пользователь прокручивает одну панель и жест завершается (settle)
- **THEN** другая панель оказывается на реплике с ближайшим `start` к якорю прокрученной панели, с пиксельной подстройкой после settle, без требования 1:1 копирования offset во время fling

### Requirement: Restore translation when reopening original
Если у оригинального документа есть сохранённый связанный перевод, система SHALL при открытии Reader сделать перевод доступным в нижней панели после включения split (без обязательной повторной загрузки с устройства в той же установке приложения). Поведение хранения связи — capability `library/documents`.

#### Scenario [RS-09]: Reopen restores translation in split
- **GIVEN** оригинал ранее получил связанный перевод через нижнюю панель, приложение снова открывает этот оригинал в Reader
- **WHEN** пользователь включает split
- **THEN** нижняя панель показывает cues сохранённого перевода (не пустое состояние загрузки)
