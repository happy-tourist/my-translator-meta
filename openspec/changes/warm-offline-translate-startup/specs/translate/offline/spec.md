# Delta: translate/offline

Базовый контракт: `openspec/specs/translate/offline/spec.md` (TO-01…TO-08). Этот change: warm-up/ensure + **fullscreen loading gate** на старте сессии + отзывчивый Translate wait.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| TO-09 | Fullscreen loading until engine ready | Flutter widget: после Drift open — fullscreen loader; shell недоступен; после ready — shell; повторный ensure без второго тяжёлого init |
| TO-10 | Translate waits while warm-up in flight | Flutter widget: Translate → spinner; после ready — gloss/err (напр. после fail-dismiss loader) |
| TO-11 | Loader / Translate path not frozen | Flutter: анимация loader / spinner жива; после shell — cold Translate path с yields (не полный freeze UI isolate) |
| TO-12 | Re-warm after process restart | verify/ops: новый процесс — снова gate+warm-up; in-session ready без повторного cold init |
| TO-13 | Warm-up fail dismisses fullscreen | Flutter: ensure fail → loader снят, shell доступен; без обязательного SnackBar от warm-up |

## ADDED Requirements

### Requirement: Fullscreen engine loading gate at session start
После успешного открытия локального storage (не во время DB bootstrap spinner) и **до** интерактивного product shell (Library/Vocab) система SHALL показать **полноэкранный** loading, пока process-wide ensure/warm-up offline translate не завершится успехом или ошибкой. Пока loader показан, Library/Vocab и прочий product chrome MUST NOT быть доступны для взаимодействия. Warm-up MUST делить один process-wide in-flight ensure с Translate (без двойного native load / Web pipeline). После успешного ready повторный cold init в той же сессии MUST NOT требоваться для обычного Translate. После завершения процесса / полной перезагрузки Web-вкладки gate+warm-up MAY выполняться снова. Widget-тесты MUST иметь escape hatch отключить auto warm-up/gate.

#### Scenario [TO-09]: Fullscreen loading until engine ready
- **GIVEN** приложение запущено со встроенными model assets, локальный storage открыт
- **WHEN** идёт первая инициализация offline translate в сессии
- **THEN** пользователь видит полноэкранный loading; product shell недоступен; после успешного ready показывается Library/Vocab; повторный ensure в той же сессии MUST NOT стартовать второй тяжёлый native load / второй Web pipeline

#### Scenario [TO-12]: Re-warm after process restart
- **GIVEN** в предыдущей сессии engine был успешно инициализирован, затем процесс завершён или Web-вкладка перезагружена
- **WHEN** пользователь снова открывает приложение и storage открыт
- **THEN** система снова показывает fullscreen gate и warm-up для новой сессии; после ready Translate не требует повторного cold init до следующего завершения процесса / reload

#### Scenario [TO-13]: Warm-up fail dismisses fullscreen
- **GIVEN** fullscreen loading gate активен
- **WHEN** ensure/warm-up завершается ошибкой
- **THEN** loader снимается, product shell становится доступен; warm-up MUST NOT сам по себе требовать блокирующий error dialog; последующий Translate показывает kind (`engineUnavailable` / `internal`) по main spec

### Requirement: Responsive Translate while engine initializes
Если пользователь нажимает Translate до успешной готовности движка (например после fail-dismiss gate или гонки), система SHALL дождаться того же init/warm-up (или lazy init) и MUST показать живой loading на кнопке Translate. Успешный gloss или типизированная ошибка MUST следовать kinds из main spec; ложный перевод MUST NOT вставляться. На cold path после появления shell UI isolate MUST NOT полностью замирать из‑за синхронной тяжёлой работы без yield (native asset copy / encode path).

#### Scenario [TO-10]: Translate waits while warm-up in flight
- **GIVEN** warm-up или lazy init ещё не завершён, product shell уже доступен
- **WHEN** пользователь нажимает Translate для непустого EN текста с парой en→ru
- **THEN** на кнопке Translate отображается loading; запрос ждёт того же init; после завершения — primary gloss или user-visible ошибка по kind

#### Scenario [TO-11]: Loader / Translate path not frozen
- **GIVEN** идёт fullscreen gate или cold Translate wait
- **WHEN** проходит время init
- **THEN** индикатор loading остаётся живым (кадры анимации); после shell на cold Translate path жесты/кадры MUST NOT полностью замирать из‑за синхронной тяжёлой работы на UI isolate без yield

## MODIFIED Requirements

### Requirement: Engine readiness and failure signaling
Система SHALL сообщать UI типизированной ошибкой, когда движок недоступен или инференс не удался, так чтобы UI мог показать user-visible ошибку и не вставлять ложный перевод. Пока идёт первая инициализация модели из assets (fullscreen gate / warm-up или lazy init по Translate), система MUST показывать loading (fullscreen до ready/fail на старте сессии; на кнопке Translate при активном запросе после shell). Поддерживаемая пара — только EN→RU; запрос иной пары MUST давать `UnsupportedLanguagePair` (коды case-insensitive; нормализация к `"en"` / `"ru"`).

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
