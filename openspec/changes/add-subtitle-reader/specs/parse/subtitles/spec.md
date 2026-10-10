## Purpose

Парсит содержимое файла субтитров (SRT, WebVTT, ASS/SSA) в упорядоченные реплики с таймкодами и текстом для экрана Reader; compute выполняется в Rust и отдаётся Flutter через FRB.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| PS-01 | Parse SRT into cues | Rust `cargo test`: SRT fixture → cues with times + text |
| PS-02 | Parse WebVTT into cues | Rust: VTT fixture → cues |
| PS-03 | Parse ASS/SSA into cues | Rust: ASS/SSA fixture → cues |
| PS-04 | Reject unsupported or unrecognizable content | Rust: garbage / unknown → error |
| PS-05 | Reject non-UTF-8 under encoding policy | Rust: invalid bytes → encoding error |
| PS-06 | Reject empty cue list | Rust: valid-looking empty → empty error |
| PS-07 | Lossy tolerates recoverable cue defects | Rust: one bad block still yields other cues when format known |

## ADDED Requirements

### Requirement: Parse supported subtitle formats into cues
Система SHALL принимать содержимое субтитров форматов SRT, WebVTT и ASS/SSA и возвращать упорядоченный список реплик (cues). Каждая реплика MUST содержать индекс порядка, время начала и конца в миллисекундах и одну или более текстовых строк реплики (plain text, без обязательного сохранения разметки стилей).

#### Scenario [PS-01]: Parse SRT into cues
- **GIVEN** валидное содержимое SRT с хотя бы одной репликой
- **WHEN** система парсит это содержимое
- **THEN** возвращается список cues с корректными `start_ms` / `end_ms` и текстом реплик в порядке следования

#### Scenario [PS-02]: Parse WebVTT into cues
- **GIVEN** валидное содержимое WebVTT с хотя бы одной cue
- **WHEN** система парсит это содержимое
- **THEN** возвращается список cues с временами и текстом в порядке следования

#### Scenario [PS-03]: Parse ASS/SSA into cues
- **GIVEN** валидное содержимое ASS или SSA с хотя бы одним Dialogue-событием
- **WHEN** система парсит это содержимое
- **THEN** возвращается список cues с временами и plain-text репликами (override-коды не обязаны сохраняться в тексте)

### Requirement: Detect format and reject unsupported content
Система SHALL определять формат по содержимому (с учётом подсказки расширения, если она согласована). Если формат не распознан или содержимое не является поддерживаемым субтитром, система MUST вернуть ошибку неуспешного парсинга и MUST NOT возвращать частичный список cues как успех.

#### Scenario [PS-04]: Reject unsupported or unrecognizable content
- **GIVEN** содержимое, которое не является SRT, WebVTT или ASS/SSA
- **WHEN** система пытается распарсить его
- **THEN** возвращается ошибка неуспешного парсинга без успешного списка cues

### Requirement: Encoding policy UTF-8
Система SHALL принимать UTF-8 (включая BOM). Байты, которые нельзя декодировать как UTF-8 по политике change, MUST приводить к ошибке encoding; система MUST NOT молча подменять текст испорченными символами без явной ошибки.

#### Scenario [PS-05]: Reject non-UTF-8 under encoding policy
- **GIVEN** байты документа, не являющиеся валидным UTF-8
- **WHEN** система пытается декодировать и распарсить их
- **THEN** возвращается ошибка encoding и успешного списка cues нет

### Requirement: Empty cue list is a hard failure
Система SHALL считать результат с нулём реплик ошибкой парсинга (пустой документ неуспешен для открытия Reader). Успешный результат MUST содержать хотя бы одну cue.

#### Scenario [PS-06]: Reject empty cue list
- **GIVEN** содержимое, которое распознаётся как поддерживаемый формат, но не содержит ни одной реплики
- **WHEN** система парсит это содержимое
- **THEN** возвращается ошибка пустого результата, а не успешный пустой список

### Requirement: Lossy parse for recoverable defects
Для пользовательских файлов система SHALL использовать lossy-режим (или эквивалент): локальные дефекты отдельных блоков MUST NOT обязательно валить весь документ, если формат распознан и после разбора остаётся хотя бы одна валидная cue. Полный hard-fail сохраняется для нераспознанного формата, encoding и нуля cues.

#### Scenario [PS-07]: Lossy tolerates recoverable cue defects
- **GIVEN** файл распознанного формата, в котором один блок повреждён, а другие реплики валидны
- **WHEN** система парсит файл в lossy-режиме
- **THEN** возвращается успешный список, включающий валидные cues (повреждённый блок может быть пропущен)
