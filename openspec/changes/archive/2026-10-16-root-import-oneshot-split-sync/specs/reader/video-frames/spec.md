# reader/video-frames (delta) — no page→translation sync

Связано с `reader/split` (one-shot только при Show / load перевода). Смена video page больше не двигает перевод.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| VF-06 | Original page change syncs translation when split | Flutter widget: next/prev page → bottom **unchanged** (no page sync; title kept for MODIFIED identity) |
| VF-07 | Bottom scroll does not move original when video | unchanged |
| VF-09 | Frames only in original pane | unchanged |

## MODIFIED Requirements

### Requirement: Split sync with video pages
Кадры только в original; bottom без кадров. Смена страницы video pages MUST NOT синхронизировать переводную панель. Scroll bottom MUST NOT двигать original page. One-shot выравнивание перевода при Show split / load перевода — только `reader/split` (по текущей секунде/якорю video или cues). Show split MUST NOT сбрасывать page.

#### Scenario [VF-06]: Original page change syncs translation when split
- **GIVEN** video + split + translation, перевод прокручен к известной позиции
- **WHEN** next/prev page
- **THEN** позиция нижней панели перевода не меняется из‑за смены страницы (page→translation sync отсутствует)

#### Scenario [VF-07]: Bottom scroll does not move original when video
- **GIVEN** video на T, split on
- **WHEN** scroll translation
- **THEN** original page/second остаётся на T

#### Scenario [VF-09]: Frames only in original pane
- **GIVEN** video + split
- **WHEN** обе панели видны
- **THEN** кадры только сверху
