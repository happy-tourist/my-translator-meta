## 1. Book scroll infrastructure

- [x] 1.1 Прочитать `design.md` (D1–D4), delta `specs/reader/split/spec.md`, skills `work-with-reader` / `split-scroll.md` + `chapters.md`, и текущую book-ветку `_toggleSplit` / `_buildBookPaneContent` в sibling Reader — зафиксировать точки врезки без правок кода в этом пункте
- [x] 1.2 Добавить на state Reader отдельные `ItemScrollController` + `ItemPositionsListener` для book paragraph lists (original / translation) и подключить их в `_buildBookPaneContent` через `ScrollablePositionedList` вместо голого `ListView.builder`; сохранить remount-на-смену-главы (RC-02, keys `*ParagraphList_$index`); не шарить cue-контроллеры — проверить компиляцией / `flutter analyze` на затронутых файлах

## 2. Preserve + one-shot

- [x] 2.1 Book-ветка `_toggleSplit` (Show **и** Hide): capture top-visible paragraph index + alignment (паттерн `_topVisible` / subtitle path) → seed initial + post-frame `jumpTo` оригинала (RS-10 / RS-11) — mid-chapter не уезжает в para 0
- [x] 2.2 При Show с translation chapters и при успешном load EPUB перевода при split on (`_loadTranslation` book): chapter align (как сейчас) **плюс** one-shot `jumpTo` перевода к `clamp(sourceParagraphIndex)` + alignment из **захваченного** якоря (RS-19 / RS-20 / RS-22); retry attach при необходимости — без settle-sync в `_onScrollNotification` (RS-21)

## 3. Tests + meta

- [x] 3.1 Widget-тесты в `test/features/reader/reader_epub_split_test.dart`: RS-10/RS-11 mid-chapter preserve; RS-19/RS-20 chapter+paragraph one-shot; RS-21 независимость после one-shot; RS-22 clamp при M ≤ p — `flutter test` проходит
- [x] 3.2 Регрессия субтитров: существующие RS-10 / RS-18 (и смежные split) в `test/features/reader/reader_screen_test.dart` — `flutter test` проходит
- [x] 3.3 `flutter analyze` в sibling app — без новых issues по затронутым путям
- [x] 3.4 Skills согласованы с delta: `split-scroll.md` / `chapters.md` / `work-with-reader/SKILL.md` / `work-with-pages/reader.md` — EPUB one-shot = глава + абзац; preserve mid-chapter на Show/Hide *(align-back + align-front; при apply сверить ещё раз)*
