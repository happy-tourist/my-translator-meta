# Delta: library/documents

Базовый контракт: `openspec/specs/library/documents/spec.md` (LD-17 / LD-28 и др.). Этот change добавляет persist якоря чтения **перевода** для restore нижней панели Reader (`reader/split`) и **mid-chapter** якоря **оригинала** EPUB (абзац + alignment рядом с chapter index).

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| LD-17 | Original anchor unchanged (subtitle) | regression: original still persists |
| LD-28 | EPUB original keeps chapter index | unit/repo: startMs = chapter |
| LD-31 | Persist translation reading anchor | unit/repo + Reader: save bottom position; reopen restores |
| LD-32 | Translation EPUB anchor is paragraph-in-chapter | unit: epub translation stores paragraph index (+ alignment); chapter follows session/original link rules |
| LD-33 | Persist original EPUB mid-chapter paragraph | unit/repo + Reader: save chapter + paragraph (+ alignment); reopen restores mid-chapter |

## ADDED Requirements

### Requirement: Persist reading anchor for linked translation
Система SHALL хранить якорь последней позиции **нижней** панели (перевод) так, чтобы он переживал перезапуск на поддерживаемых платформах. Для **subtitle**-перевода якорь MUST включать время/`start` якорной реплики (или эквивалентный cue key) и alignment при наличии. Для **EPUB**-перевода якорь MUST включать индекс абзаца в текущей главе перевода и alignment при наличии; индекс главы перевода при reopen SHALL согласовываться с главой оригинала после restore оригинала и linked chapter rules (`reader/chapters`), а не one-shot sync скролла. Система MUST обновлять якорь перевода при settle скролла нижней панели и при выходе из Reader / Hide split, когда позиция известна. При open оригинала с linked translation система SHALL отдавать сохранённый якорь перевода в сессию Reader (если есть). Сбой якоря перевода MUST NOT блокировать open оригинала. Persist режима split и divider по-прежнему не требуется.

#### Scenario [LD-31]: Persist and load translation reading anchor
- **GIVEN** оригинал открыт в Reader со split и переводом, пользователь прокрутил низ не к началу
- **WHEN** якорь перевода сохранён (settle и/или выход) и пользователь снова открывает этот оригинал с доступным переводом
- **THEN** сессия Reader получает сохранённый якорь перевода, и нижняя панель может восстановить позицию без sync к верху

#### Scenario [LD-32]: Translation EPUB anchor stores paragraph position
- **GIVEN** EPUB-перевод в split, пользователь на абзаце p главы, согласованной с оригиналом, якорь сохранён
- **WHEN** документ снова открывается
- **THEN** Reader может восстановить нижнюю панель около абзаца p (clamp), без требования one-shot align к абзацу верха

### Requirement: Persist mid-chapter reading position for original EPUB
Для документа format `epub` (оригинал) система SHALL хранить не только 0-based индекс главы (`lastReadStartMs` / LD-28), но и **индекс абзаца** внутри этой главы и alignment при наличии — тем же классом данных, что mid-chapter якорь низа (paragraph + alignment). Система MUST обновлять этот mid-chapter якорь при settle скролла панели оригинала и при выходе из Reader, когда позиция известна. При open система SHALL отдавать chapter + paragraph (+ alignment) в сессию Reader. Старые якоря без paragraph MAY открывать главу с начала списка абзацев. Сбой якоря MUST NOT блокировать open.

#### Scenario [LD-33]: Persist and load original EPUB mid-chapter paragraph
- **GIVEN** EPUB-оригинал открыт в Reader, пользователь на главе k и абзаце p (не начало главы), якорь сохранён
- **WHEN** пользователь снова открывает этот документ
- **THEN** сессия Reader получает главу k и абзац p (+ alignment при наличии), чтобы панель оригинала открылась mid-chapter без sync к переводу

## MODIFIED Requirements

### Requirement: Persist reading anchor for original document
(Без изменения сценария LD-17 для subtitle.) Уточнение: скрытый translation-документ / слот перевода **теперь** MAY/SHALL хранить отдельный якорь чтения низа (см. ADDED выше); фраза main spec «не обязан хранить отдельный якорь» для перевода в этом change снимается в части нижней панели. Для **EPUB**-оригинала mid-chapter paragraph — см. ADDED LD-33 (расширение LD-28).

#### Scenario [LD-17]: Persist and load reading anchor for original
- **GIVEN** оригинал открыт в Reader и пользователь остановился на реплике с известным `start` (не начало файла)
- **WHEN** якорь сохранён (после settle и/или выхода из Reader) и пользователь снова открывает этот оригинал из папки
- **THEN** в сессию Reader передаётся сохранённый якорь оригинала (`start` и alignment при наличии)

### Requirement: EPUB reading anchor stores chapter index
Для документа format `epub` система SHALL сохранять и восстанавливать **0-based индекс главы** в существующей колонке `lastReadStartMs` (format-gated семантика). Для subtitle-документов `lastReadStartMs` остаётся cue start ms. **ДОПОЛНЕНИЕ этого change:** для **оригинала** EPUB система SHALL также persist mid-chapter paragraph (+ alignment) — LD-33; формулировка main «Resume абзаца внутри главы в v1 не требуется» для оригинала **снимается**. Сбой якоря MUST NOT блокировать open.

#### Scenario [LD-28]: EPUB reading anchor is chapter index
- **GIVEN** EPUB-оригинал на главе k, якорь сохранён
- **WHEN** документ снова открывается
- **THEN** Reader получает chapter index k (и при наличии mid-chapter данных — paragraph/alignment по LD-33)
