## Purpose

Показывает EPUB в Reader как книгу по главам: внутри главы — вертикальный список абзацев с learning-жестами как у субтитров; между главами — нижний pager; без timed cue list и без video-frames для book-документов.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| RC-01 | Show paragraphs of current chapter | Flutter widget: open book → paragraph list for chapter 1 |
| RC-02 | Bottom chapter pager advances | Flutter widget: next/prev → chapter index + counter n/N |
| RC-03 | Pager page count equals chapter count | Flutter unit/widget: N chapters → N pages |
| RC-04 | Tap/highlight on paragraph like cues | Flutter widget: tap word on paragraph → CTA/sheet path as subtitle cues |
| RC-05 | No video chrome for EPUB document | Flutter widget: book session → no Load video for this document kind |
| RC-06 | Restore chapter from reading anchor | Flutter widget: reopen with saved chapter index → that chapter shown |
| RC-07 | Book hit-test without per-word WidgetSpan | Flutter widget: tap word → CTA; paragraph tree MUST NOT use one WidgetSpan/GestureDetector per token |
| RC-08 | Progressive tokenize keeps early tap | Flutter widget: after open, tap on early paragraph works before full-chapter tokenize/mark completes |
| RC-09 | Chapter token cache on pager return | Flutter widget: leave chapter and return → no full re-tokenize wait (cache hit); selection/highlight still work |

## ADDED Requirements

### Requirement: Display current chapter as paragraph list
В book-режиме Reader система SHALL показывать текущую главу как вертикально прокручиваемый список plain-text абзацев (без обязательных таймкодов). Заголовок текущей главы SHOULD быть видим пользователю (в списке или chrome). Пустая глава после успешного open MUST NOT встречаться (гарантирует `parse/epub`).

#### Scenario [RC-01]: Show paragraphs of current chapter
- **GIVEN** Reader открыт для EPUB с несколькими главами и абзацами
- **WHEN** пользователь видит исходную главу (первая при первом open без якоря)
- **THEN** отображается прокручиваемый список абзацев этой главы, а не timed cue list субтитров

### Requirement: Bottom chapter pager
Система SHALL предоставлять внизу каждой book-панели chrome переключения глав: предыдущая / следующая и индикатор `n / N`, где N — число глав **этой** панели, n — 1-based индекс текущей главы. В single-pane book-режиме это original pane; при split с загруженным EPUB-переводом — и верх, и низ (sync индекса — `reader/split` RS-14). Число страниц пейджера MUST равняться числу глав панели. Смена главы MUST заменить список абзацев содержимым выбранной главы и SHOULD прокрутить список абзацев к началу главы.

#### Scenario [RC-02]: Bottom chapter pager advances
- **GIVEN** Reader в book-режиме на главе 1 из N (N ≥ 2)
- **WHEN** пользователь нажимает переход к следующей главе
- **THEN** показывается глава 2, индикатор отражает `2 / N`, список абзацев соответствует главе 2

#### Scenario [RC-03]: Pager page count equals chapter count
- **GIVEN** разобранный EPUB с ровно N главами
- **WHEN** пользователь открывает его в Reader
- **THEN** пейджер сообщает N страниц (глав), по одной на главу

### Requirement: Learning gestures on paragraphs
На абзацах book-режима система SHALL поддерживать те же learning-потоки, что и на тексте cues в subtitle-режиме: contiguous tap/selection → add-vocab, lemma/phrase highlight и offline translate по выбранному фрагменту (поведение детализировано в `reader/tap`, `reader/highlight`, `translate/offline`). Таймкоды cue для абзацев не требуются и MUST NOT отображаться как subtitle time range (design D4: `showTimestamp: false` / эквивалент). Абзацы текущей главы SHALL токенизироваться через существующий `tokenize_cues(texts: …)` (список строк абзацев), без отдельного FRB tokenize API. Реализация hit-target в book-режиме MUST NOT требовать отдельный `WidgetSpan`/`GestureDetector` на каждый word-токен (design D11); subtitle CueRow path MAY сохранять прежнюю модель.

#### Scenario [RC-04]: Tap/highlight on paragraph like cues
- **GIVEN** Reader в book-режиме с видимым абзацем, содержащим слово
- **WHEN** пользователь выполняет tap по слову (как в subtitle Reader)
- **THEN** доступен тот же путь добавления в словарь / CTA, что для cues, без требования timestamp и без отображения timed cue chrome у абзаца

### Requirement: EPUB session excludes video frames
Для документа формата EPUB система MUST NOT предлагать привязку video-frames как замену списка абзацев. Book-режим и paged video frames MUST NOT смешиваться в одной original pane.

#### Scenario [RC-05]: No video chrome for EPUB document
- **GIVEN** Reader открыт для EPUB-документа
- **WHEN** пользователь смотрит chrome панели оригинала
- **THEN** действие загрузки видео для paged frames отсутствует или недоступно для этого документа

### Requirement: Resume at saved chapter
Если для EPUB-документа сохранён якорь чтения с индексом главы, система SHALL при повторном open показать эту главу (в пределах `0..N-1`). Persist якоря — format-gated `lastReadStartMs` = chapter index (`library/documents` LD-28 / design D6). Resume абзаца внутри главы в v1 не требуется. Сбой якоря MUST NOT блокировать open книги.

#### Scenario [RC-06]: Restore chapter from reading anchor
- **GIVEN** для EPUB сохранён якорь на главу k (валидный индекс)
- **WHEN** пользователь снова открывает этот документ
- **THEN** Reader показывает главу k, а не обязательно первую

### Requirement: Lightweight book-mode word hit-test
В book-режиме система SHALL разрешать tap по слову абзаца без построения отдельного интерактивного виджета на каждый токен (запрет per-token `WidgetSpan` + `GestureDetector` в paragraph row). Hit-test MUST отображать координату тапа на `tokenIndex` существующего batch tokenize (char spans) и вызывать тот же learning path, что RC-04. Subtitle-режим CueRow MAY продолжать использовать WidgetSpan-per-token.

#### Scenario [RC-07]: Book hit-test without per-word WidgetSpan
- **GIVEN** Reader в book-режиме, абзац токенизирован (хотя бы частично для этого абзаца)
- **WHEN** пользователь тапает по слову в абзаце
- **THEN** открывается тот же CTA/sheet path, что RC-04, и в дереве paragraph row нет отдельного WidgetSpan-hit-target на каждый токен

### Requirement: Progressive chapter tokenize with optional delayed highlight
Система SHALL показывать plain-text абзацы текущей главы немедленно и MAY выполнять `tokenize_cues` / `mark_vocab_tokens` прогрессивно (сначала префикс абзацев / видимая зона, затем остаток). Tap на уже токенизированном префиксе MUST работать до завершения tokenize всей главы. Lemma/phrase highlight MAY появиться позже tokenize (после mark); отсутствие полного highlight MUST NOT блокировать tap/CTA на готовых токенах.

#### Scenario [RC-08]: Progressive tokenize keeps early tap
- **GIVEN** Reader открыл EPUB-главу с многими абзацами
- **WHEN** tokenize/mark всей главы ещё не завершены, но первые абзацы уже имеют токены
- **THEN** tap по слову в таком абзаце активирует learning path; полный highlight словаря MAY ещё отсутствовать

### Requirement: In-memory chapter token cache
Пока Reader session открыт, система SHALL кешировать результат tokenize (+ mark, когда готов) по 0-based индексу главы оригинала. При возврате на ранее посещённую главу через pager система MUST использовать кеш и MUST NOT обязательно запускать полный повторный `tokenize_cues` для этой главы (invalidate при изменении словаря / re-mark после Add/edit/delete — как сейчас для cues).

#### Scenario [RC-09]: Chapter token cache on pager return
- **GIVEN** пользователь токенизировал главу 1, перешёл на главу 2, затем вернулся на главу 1 в той же сессии
- **WHEN** глава 1 снова отображается
- **THEN** токены/highlight главы 1 доступны без обязательного полного повторного batch tokenize этой главы
