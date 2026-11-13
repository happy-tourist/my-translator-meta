# Delta: reader/chapters

Базовый контракт: `openspec/specs/reader/chapters/spec.md`. В split: pager только наверху; стрелки двигают низ; rewind к началу главы.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RC-02 | Top pager advances | Flutter |
| RC-03 | Pager page count | Flutter |
| RC-10 | Top arrows move bottom (title kept; THEN updated) | Flutter |
| RC-11 | No pager on translation pane | Flutter |
| RC-12 | Rewind both lists on chapter change | Flutter |

## MODIFIED Requirements

### Requirement: Bottom chapter pager
В single-pane book-режиме система SHALL показывать chapter pager на панели оригинала. В split с EPUB-переводом система SHALL показывать pager **только на верхней** панели и MUST NOT показывать pager (и chapter title) на нижней. Prev/next наверху SHALL выставлять индекс главы перевода в clamp(индекс оригинала) и MUST прокрутить списки абзацев **верха и низа** к **началу** новой главы. Прокрутка абзацев одной панели MUST NOT менять другую. Индикатор `n / N` = главы оригинала. Число страниц пейджера MUST равняться числу глав оригинала.

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
