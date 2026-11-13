# Delta: reader/chapters

Базовый контракт: `openspec/specs/reader/chapters/spec.md`. В split: pager только наверху; стрелки двигают низ; rewind к началу главы. У оригинала — без закреплённого chapter title chrome.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RC-01 | Paragraphs without fixed title chrome | Flutter widget: original pane — no pinned chapter.title above list |
| RC-02 | Top pager advances | Flutter |
| RC-03 | Pager page count | Flutter |
| RC-06 | Restore chapter (+ mid-chapter via LD-33) | Flutter: reopen → chapter; paragraph via reader/split / library |
| RC-10 | Top arrows move bottom | Flutter |
| RC-11 | No pager on translation pane | Flutter |
| RC-12 | Rewind both lists on chapter change | Flutter |
| RC-13 | Original has no fixed chapter title | Flutter widget: title chrome absent; pager arrows remain |

## MODIFIED Requirements

### Requirement: Display current chapter as paragraph list
В book-режиме Reader система SHALL показывать текущую главу как вертикально прокручиваемый список plain-text абзацев (без обязательных таймкодов). Система MUST NOT показывать **закреплённый** (fixed / chrome) заголовок `chapter.title` над списком на панели оригинала (и по-прежнему MUST NOT на низу в split). Название главы MAY присутствовать только если оно уже есть в тексте абзацев контента. Пустая глава после успешного open MUST NOT встречаться (гарантирует `parse/epub`).

#### Scenario [RC-01]: Show paragraphs of current chapter
- **GIVEN** Reader открыт для EPUB с несколькими главами и абзацами
- **WHEN** пользователь видит исходную главу (первая при первом open без якоря)
- **THEN** отображается прокручиваемый список абзацев этой главы без timed cue list и **без** отдельного закреплённого title chrome над списком

### Requirement: Bottom chapter pager
В single-pane book-режиме система SHALL показывать chapter pager (prev/next + `n / N`) на панели оригинала. В split с EPUB-переводом система SHALL показывать pager **только на верхней** панели и MUST NOT показывать pager (и chapter title) на нижней. Prev/next наверху SHALL выставлять индекс главы перевода в clamp(индекс оригинала) и MUST прокрутить списки абзацев **верха и низа** к **началу** новой главы. Прокрутка абзацев одной панели MUST NOT менять другую. Индикатор `n / N` = главы оригинала. Число страниц пейджера MUST равняться числу глав оригинала.

#### Scenario [RC-02]: Bottom chapter pager advances
- **GIVEN** Reader в book-режиме на главе 1 из N (N ≥ 2)
- **WHEN** пользователь нажимает переход к следующей главе на верхней/единственной панели
- **THEN** показывается глава 2, индикатор отражает `2 / N`, список абзацев верха соответствует главе 2 и начинается с начала главы

#### Scenario [RC-03]: Pager page count equals chapter count
- **GIVEN** разобранный EPUB с ровно N главами
- **WHEN** пользователь открывает его в Reader
- **THEN** пейджер сообщает N страниц (глав), по одной на главу

#### Scenario [RC-10]: Split pager does not move other pane
- **GIVEN** split включён, обе EPUB-панели на главе 1 (N ≥ 2)
- **WHEN** пользователь нажимает next на верхней панели
- **THEN** верх показывает главу 2 **и** низ также показывает главу 2 (или clamp); оба списка абзацев с начала главы

### Requirement: Resume at saved chapter
Если для EPUB-документа сохранён якорь чтения с индексом главы, система SHALL при повторном open показать эту главу (в пределах `0..N-1`). Persist главы — format-gated `lastReadStartMs` = chapter index (`library/documents` LD-28). **ДОПОЛНЕНИЕ:** mid-chapter paragraph оригинала — LD-33 / `reader/split` RS-12; формулировка main «Resume абзаца внутри главы в v1 не требуется» для оригинала **снимается**. Смена главы через pager по-прежнему SHALL открывать список с начала главы (RC-12). Сбой якоря MUST NOT блокировать open книги.

#### Scenario [RC-06]: Restore chapter from reading anchor
- **GIVEN** для EPUB сохранён якорь на главу k (валидный индекс)
- **WHEN** пользователь снова открывает этот документ
- **THEN** Reader показывает главу k, а не обязательно первую (и mid-chapter абзац при наличии LD-33)

## ADDED Requirements

### Requirement: Translation pane has no chapter pager chrome
В split с EPUB-переводом нижняя панель MUST NOT содержать prev/next / `n / N` chapter pager и MUST NOT показывать chapter title chrome над текстом.

#### Scenario [RC-11]: No chapter pager on translation pane
- **GIVEN** EPUB split с загруженным переводом
- **WHEN** пользователь смотрит нижнюю панель
- **THEN** prev/next и `n / N` на низу отсутствуют

### Requirement: Chapter change rewinds paragraph lists to top
При смене главы через верхний pager в split система MUST показать начало главы на верхе и на низу (index 0), не сохраняя mid-chapter scroll offset для этого переключения.

#### Scenario [RC-12]: Chapter change rewinds both lists to top
- **GIVEN** split EPUB, прокрутка абзацев не в начале наверху и/или внизу
- **WHEN** пользователь переключает главу стрелками сверху
- **THEN** списки абзацев верха и низа показывают начало новой главы

### Requirement: Original book pane has no fixed chapter title chrome
Панель оригинала в book-режиме (single и split top) MUST NOT показывать закреплённый заголовок главы над списком абзацев. Система SHALL сохранять chapter pager (стрелки и `n / N`) у низа этой панели.

#### Scenario [RC-13]: Original pane has no fixed chapter title
- **GIVEN** Reader в book-режиме на EPUB (single или split)
- **WHEN** пользователь смотрит панель оригинала
- **THEN** над списком абзацев нет отдельного chrome с `chapter.title`; pager prev/next и `n / N` остаются
