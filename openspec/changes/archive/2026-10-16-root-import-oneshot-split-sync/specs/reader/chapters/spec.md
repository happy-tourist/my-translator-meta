# reader/chapters (delta) — independent chapter pager per pane

Связано с `reader/split` (one-shot align глав при Show / load). Realtime sync индекса между панелями снимается.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RC-02 | Bottom chapter pager advances | Flutter widget: next/prev → that pane’s chapter only |
| RC-03 | Pager page count equals chapter count | unchanged |
| RC-10 | Split pager does not move other pane | Flutter widget: top pager → bottom chapter unchanged |

## MODIFIED Requirements

### Requirement: Bottom chapter pager
Система SHALL предоставлять внизу каждой book-панели chrome переключения глав: предыдущая / следующая и индикатор `n / N`, где N — число глав **этой** панели, n — 1-based индекс текущей главы **этой** панели. В single-pane book-режиме это original pane; при split с загруженным EPUB-переводом — и верх, и низ, каждый со **своим** индексом (one-shot выравнивание при Show / load — `reader/split`). Число страниц пейджера MUST равняться числу глав панели. Смена главы на панели MUST заменить список абзацев **только этой** панели содержимым выбранной главы и SHOULD прокрутить список абзацев к началу главы. Смена главы на одной панели MUST NOT менять индекс другой панели.

#### Scenario [RC-02]: Bottom chapter pager advances
- **GIVEN** Reader в book-режиме на главе 1 из N (N ≥ 2)
- **WHEN** пользователь нажимает переход к следующей главе
- **THEN** показывается глава 2, индикатор отражает `2 / N`, список абзацев соответствует главе 2

#### Scenario [RC-03]: Pager page count equals chapter count
- **GIVEN** разобранный EPUB с ровно N главами
- **WHEN** пользователь открывает его в Reader
- **THEN** пейджер сообщает N страниц (глав), по одной на главу

#### Scenario [RC-10]: Split pager does not move other pane
- **GIVEN** split включён, обе EPUB-панели на главе 1 (N ≥ 2)
- **WHEN** пользователь нажимает next на верхней панели
- **THEN** верх показывает главу 2, низ остаётся на главе 1
