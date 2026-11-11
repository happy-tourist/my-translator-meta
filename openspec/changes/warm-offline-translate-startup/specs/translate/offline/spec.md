# Delta: translate/offline

Базовый контракт: `openspec/specs/translate/offline/spec.md` (TO-01…TO-08). Этот change добавляет фоновый warm-up и отзывчивый cold path на Translate.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| TO-09 | Background warm-up after UI start | Flutter widget/facade test: warm-up seam вызван после первого кадра; повторный ready без повторного тяжёлого init |
| TO-10 | Translate waits while warm-up in flight | Flutter widget test: Translate → spinner/loading на кнопке; после ready — gloss или типизированная ошибка |
| TO-11 | UI stays responsive during cold init | Flutter test или ручная приёмка Web/Windows/Android: на cold init UI принимает жесты / кадры (не полный freeze); loading на кнопке виден |
| TO-12 | Re-warm after process restart | verify/ops: новый процесс / reload вкладки — снова warm-up; внутри сессии после ready — без повторного cold init |

## ADDED Requirements

### Requirement: Background engine warm-up at session start
После первого отображения **product shell** (Library/Vocab после открытия локального storage; не bootstrap-спиннер открытия БД) система SHALL инициировать тихий фоновый warm-up офлайн-движка перевода (native и Web) в рамках текущей сессии процесса / вкладки. Warm-up MUST NOT блокировать показ первого кадра product shell и MUST NOT требовать отдельного splash или постоянного индикатора вне кнопки Translate. Пока warm-up не завершён успешно, вызов перевода через UI MUST оставаться допустимым (см. ожидание на кнопке). После успешного warm-up повторный cold init в той же сессии MUST NOT требоваться для обычного Translate. Init/warm-up MUST делить один process-wide in-flight ensure (без двойного native load / двойного Web pipeline при гонке warm-up∥Translate или повторном создании facade). После завершения процесса приложения или полной перезагрузки Web-вкладки warm-up MAY выполняться снова.

#### Scenario [TO-09]: Background warm-up after UI start
- **GIVEN** приложение запущено со встроенными model assets и product shell (Library/Vocab) впервые отрисован
- **WHEN** сессия продолжает работу без нажатия Translate
- **THEN** система инициирует фоновый warm-up offline translate; отдельный splash / статус прогрева вне Translate не показывается; повторный ensure в той же сессии (в т.ч. из Translate / второго facade) MUST NOT стартовать второй тяжёлый native load / второй Web pipeline

#### Scenario [TO-12]: Re-warm after process restart
- **GIVEN** в предыдущей сессии engine уже был успешно инициализирован, затем процесс приложения завершён или Web-вкладка полностью перезагружена
- **WHEN** пользователь снова открывает приложение и product shell отрисован
- **THEN** система снова инициирует warm-up для новой сессии; успешный Translate после ready не требует повторного cold init до следующего завершения процесса / reload

### Requirement: Responsive Translate while engine initializes
Если пользователь нажимает Translate до успешной готовности движка, система SHALL дождаться завершения того же init/warm-up (или выполнить lazy init как fallback) и MUST показать живой loading на кнопке Translate на время ожидания. На время cold init / warm-up UI MUST оставаться отзывчивым: кадры и базовые жесты MUST NOT «замирать» из‑за синхронной тяжёлой работы на UI-потоке. Успешный gloss или типизированная ошибка по-прежнему MUST следовать контракту kinds из main spec; ложный перевод MUST NOT вставляться.

#### Scenario [TO-10]: Translate waits while warm-up in flight
- **GIVEN** warm-up или lazy init ещё не завершён
- **WHEN** пользователь нажимает Translate для непустого EN текста с парой en→ru
- **THEN** на кнопке Translate отображается loading; запрос ждёт готовности того же init; после завершения показывается primary gloss или user-visible ошибка по kind (без ложного gloss)

#### Scenario [TO-11]: UI stays responsive during cold init
- **GIVEN** идёт первая загрузка / init offline модели в сессии (warm-up или Translate до ready)
- **WHEN** пользователь взаимодействует с UI (например закрытие sheet, прокрутка, другой жест)
- **THEN** интерфейс продолжает отвечать (не полный freeze); loading на Translate остаётся наблюдаемым, пока ожидание перевода активно

## MODIFIED Requirements

### Requirement: Engine readiness and failure signaling
Система SHALL сообщать UI типизированной ошибкой, когда движок недоступен или инференс не удался, так чтобы UI мог показать user-visible ошибку и не вставлять ложный перевод. Пока идёт первая инициализация модели из assets (фоновый warm-up или lazy init по Translate), система MUST показывать состояние загрузки на кнопке Translate при активном запросе перевода и MAY выполнять тихий warm-up без отдельного chrome вне кнопки. Поддерживаемая пара — только EN→RU; запрос иной пары MUST давать `UnsupportedLanguagePair` (коды case-insensitive; нормализация к `"en"` / `"ru"`).

На native путь система MUST принимать model assets как `Vec<{name, bytes}>` через FRB `init_offline_translate` (Flutter читает AssetBundle); MUST NOT требовать OS-путь к модели в публичном API. Вызов `translate_text` до успешного init MUST возвращать `EngineUnavailable`. На `wasm32` публичные символы MAY быть stub с `EngineUnavailable` (Web UI не вызывает FRB translate).

#### Scenario [TO-04]: Engine unavailable
- **GIVEN** модель или рантайм недоступны (нет init / повреждённый asset / init fail)
- **WHEN** UI запрашивает перевод
- **THEN** возвращается `TranslateErrorKind::EngineUnavailable`; успешного gloss нет

#### Scenario [TO-05]: Inference failure
- **GIVEN** engine инициализирован
- **WHEN** инференс завершается сбоем
- **THEN** возвращается `TranslateErrorKind::InferenceFailed`; UI MUST NOT заполнять Translation успешным значением

#### Scenario [TO-06]: Unsupported language pair
- **GIVEN** engine готов
- **WHEN** UI запрашивает перевод с парой, отличной от EN→RU (после casefold нормализации)
- **THEN** возвращается `TranslateErrorKind::UnsupportedLanguagePair`

#### Scenario [TO-08]: Native init with asset bytes
- **GIVEN** Flutter передал полный набор named asset bytes из поставки
- **WHEN** вызывается `init_offline_translate`
- **THEN** при валидных файлах init успешен; при отсутствии обязательного файла или битых bytes — `EngineUnavailable` (предпочтительно); `Internal` только если сбой не классифицируется как assets/init
