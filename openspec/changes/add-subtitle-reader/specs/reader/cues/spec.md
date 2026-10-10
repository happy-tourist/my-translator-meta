## Purpose

Показывает пользователю разобранные реплики субтитров на экране Reader: таймкод start→end и текст каждой cue со скроллом по всему документу после успешного открытия из Library.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RC-01 | Open Reader with cue list | Flutter widget: after successful open, Reader shows cues |
| RC-02 | Show start and end times per cue | Flutter widget: each cue row shows start→end |
| RC-03 | Scroll through all cues | Flutter widget: full list scrollable (not paginated) |
| RC-04 | Navigate back to Library | Flutter widget: back returns to folder without crash |
| RC-05 | Do not open Reader on parse failure | Flutter widget: stay on Library; error visible; no Reader |

## ADDED Requirements

### Requirement: Display ordered cues with timestamps
После успешного парсинга система SHALL показывать экран Reader со списком реплик в порядке следования. Каждая видимая реплика MUST отображать время начала и конца (start→end) и текст реплики. Время MUST форматироваться из миллисекунд как `HH:MM:SS,mmm` (zero-pad). Текст реплики в этом change — plain lines из parse (`text_lines`); tokenize / highlight / word card не требуются.

#### Scenario [RC-01]: Open Reader with cue list
- **GIVEN** документ успешно распарсен и содержит одну или более cues
- **WHEN** пользователь открывает документ
- **THEN** показывается Reader со всеми cues в порядке следования

#### Scenario [RC-02]: Show start and end times per cue
- **GIVEN** Reader открыт с несколькими cues, у которых заданы `start_ms` и `end_ms`
- **WHEN** пользователь просматривает список
- **THEN** у каждой реплики видны время начала и время конца в формате `HH:MM:SS,mmm` (start→end)

### Requirement: Scroll entire document
Система SHALL позволять прокручивать полный список cues документа (без обязательной постраничной загрузки в этом change).

#### Scenario [RC-03]: Scroll through all cues
- **GIVEN** документ с числом cues, превышающим высоту экрана
- **WHEN** пользователь прокручивает Reader
- **THEN** доступны как первые, так и последующие реплики того же документа

### Requirement: Return to Library from Reader
Система SHALL позволять вернуться из Reader в Library (папку, из которой открыли документ) без потери целостности приложения.

#### Scenario [RC-04]: Navigate back to Library
- **GIVEN** Reader открыт для документа из папки
- **WHEN** пользователь выполняет действие «назад»
- **THEN** отображается Library (та же папка или эквивалентный возврат), приложение не падает

### Requirement: Reader opens only after successful parse
Система SHALL открывать Reader только после успешного парсинга с ненулевым списком cues. При ошибке парсинга, encoding или пустом результате Reader MUST NOT открываться (поведение ошибки в Library — capability `library/documents`).

#### Scenario [RC-05]: Do not open Reader on parse failure
- **GIVEN** содержимое документа приводит к ошибке парсинга или нулю cues
- **WHEN** пользователь выбирает документ для открытия
- **THEN** экран Reader не показывается
