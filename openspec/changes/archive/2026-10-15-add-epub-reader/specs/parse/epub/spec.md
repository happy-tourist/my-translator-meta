## Purpose

Парсит содержимое файла EPUB в упорядоченные главы с plain-text абзацами для book-режима Reader; compute выполняется в Rust и отдаётся Flutter через FRB (bytes in, без filesystem path).

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| PE-01 | Parse EPUB with TOC into chapters | Rust: fixture with TOC → ordered chapters, each with ≥1 paragraph when body has text |
| PE-02 | Nested TOC entries become chapters | Rust: nested content-bearing TOC → separate chapters in document order |
| PE-03 | Missing TOC yields single chapter | Rust: EPUB without usable TOC → exactly one chapter for whole book text |
| PE-04 | Strip markup to plain paragraphs | Rust: XHTML body → plain paragraphs; no raw tags in paragraph text |
| PE-05 | Reject empty or unreadable EPUB | Rust: empty/corrupt → hard-fail kind; no partial success |
| PE-06 | Accept bytes without path | Rust: parse from byte buffer succeeds for valid EPUB (Web-friendly) |

## ADDED Requirements

### Requirement: Parse EPUB bytes into ordered chapters
Система SHALL принимать содержимое EPUB как набор байтов и возвращать упорядоченный список глав через FRB `parse_epub` → `ParsedEpub { chapters: Vec<EpubChapterDto> }` (см. design D2). Каждая глава MUST содержать индекс порядка (0-based), заголовок (из TOC или стабильный fallback `"Chapter {n}"`) и упорядоченный список plain-text абзацев. Успешный результат MUST содержать хотя бы одну главу с хотя бы одним непустым абзацем.

#### Scenario [PE-01]: Parse EPUB with TOC into chapters
- **GIVEN** валидные байты EPUB с оглавлением и текстовым содержимым глав
- **WHEN** система парсит эти байты
- **THEN** возвращается список глав в порядке чтения с заголовками и абзацами plain text

#### Scenario [PE-06]: Accept bytes without path
- **GIVEN** валидные байты EPUB доступны только как буфер (без filesystem path)
- **WHEN** система парсит этот буфер
- **THEN** парсинг завершается успехом с главами, как при наличии path

### Requirement: Chapter boundaries from TOC including nested entries
Система SHALL строить список глав по пунктам оглавления (TOC), включая вложенные пункты, у которых есть разрешаемый ресурс и непустой plain-текст после strip. Чисто структурные узлы TOC без resource / без текста MUST NOT становиться отдельными пустыми главами; их дочерние пункты с контентом MUST входить в список. Fragment (`#…`) в href в v1 MAY игнорироваться (тело = весь XHTML ресурса). Если usable TOC отсутствует, система MUST вернуть ровно одну главу на весь извлечённый текст книги (заголовок — стабильный fallback `"Chapter 1"`).

#### Scenario [PE-02]: Nested TOC entries become chapters
- **GIVEN** EPUB с вложенным TOC, где у дочерних пунктов есть текстовый контент
- **WHEN** система парсит EPUB
- **THEN** каждый такой контентный пункт представлен отдельной главой в порядке обхода TOC

#### Scenario [PE-03]: Missing TOC yields single chapter
- **GIVEN** валидный EPUB без usable оглавления, но с читаемым текстом
- **WHEN** система парсит EPUB
- **THEN** возвращается ровно одна глава, содержащая абзацы всего извлечённого текста

### Requirement: Plain-text paragraphs without images or footnotes
Система SHALL извлекать тело главы как plain text, разбитый на абзацы по блочной структуре исходной разметки. Сырые HTML/XHTML теги MUST NOT оставаться в тексте абзацев. Изображения и сноски MUST NOT требоваться для успешного парсинга и MUST NOT появляться как обязательный контент абзацев (могут быть опущены).

#### Scenario [PE-04]: Strip markup to plain paragraphs
- **GIVEN** глава EPUB с XHTML, содержащим теги разметки и текстовые блоки
- **WHEN** система извлекает абзацы главы
- **THEN** каждый абзац — непустой plain text без угловых скобок разметки тегов

### Requirement: Hard-fail unreadable or empty EPUB
Если байты не являются читаемым EPUB, либо после разбора нет ни одной главы с непустым абзацем, система MUST вернуть ошибку неуспешного парсинга и MUST NOT возвращать частичный список глав как успех. Публичные FRB kinds (`EpubParseErrorKind`): `UnsupportedFormat` (не EPUB / контейнер не читается), `Malformed` (EPUB открыт, но структура/контент непригоден), `Empty` (нет главы с ≥1 непустым абзацем). Отображение этих kinds в UI — Flutter mapping (design D10 / `library/documents` LD-11): минимум «неподдерживаемый/битый» vs «пустой текст»; не смешивать с `ParseErrorKind` субтитров без адаптации copy.

#### Scenario [PE-05]: Reject empty or unreadable EPUB
- **GIVEN** байты, которые не являются валидным EPUB, либо EPUB без извлекаемого текста
- **WHEN** система пытается распарсить их
- **THEN** возвращается `EpubParseErrorKind` (`UnsupportedFormat`, `Malformed` или `Empty`) без успешного списка глав
