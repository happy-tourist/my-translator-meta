# Delta: translate/offline

Базовый контракт: `openspec/specs/translate/offline/spec.md` (TO-01…TO-08). Этот change: warm-up/ensure + **fullscreen loading gate** + **native responsive cold path (path/dir init)** + отзывчивый Translate wait.

## Traceability

| SC-ID | Scenario | Test intent |
|-------|----------|-------------|
| TO-09 | Fullscreen loading until engine ready | Flutter widget: после Drift open — fullscreen loader; shell недоступен; после ready — shell; повторный ensure без второго тяжёлого init |
| TO-10 | Translate waits while warm-up in flight | Flutter widget: Translate → spinner; после ready — gloss/err |
| TO-11 | Loader / Translate path not frozen | Flutter: анимация fullscreen loader **жива** на native cold path (не полный freeze UI isolate); cold Translate path после shell тоже без hard-freeze |
| TO-12 | Re-warm after process restart | verify/ops: новый процесс — снова gate+warm-up; in-session ready без повторного cold init |
| TO-13 | Warm-up fail dismisses fullscreen | Flutter: ensure fail → loader снят, shell доступен; без обязательного SnackBar от warm-up |
| TO-08 | Native init with asset bytes | Rust: bytes-init MAY для тестов/compat — success / missing → EngineUnavailable |
| TO-14 | Native init from model paths | Rust + Flutter: path/dir init успешен на валидных файлах; product warm-up/ensure использует path-path, не полный bytes-through-UI |

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
Если пользователь нажимает Translate до успешной готовности движка (например после fail-dismiss gate или гонки), система SHALL дождаться того же init/warm-up (или lazy init) и MUST показать живой loading на кнопке Translate. Успешный gloss или типизированная ошибка MUST следовать kinds из main spec; ложный перевод MUST NOT вставляться. На **native** cold path (fullscreen gate или lazy Translate) UI isolate MUST NOT полностью замирать из‑за синхронной передачи/копирования всей модели как одного блокирующего bytes→FRB шага: product init MUST использовать path/dir (или эквивалент без полного Dart→Rust encode модели), а промежуточная работа на UI isolate MUST yield достаточно часто, чтобы индикатор loading оставался анимированным.

#### Scenario [TO-10]: Translate waits while warm-up in flight
- **GIVEN** warm-up или lazy init ещё не завершён, product shell уже доступен
- **WHEN** пользователь нажимает Translate для непустого EN текста с парой en→ru
- **THEN** на кнопке Translate отображается loading; запрос ждёт того же init; после завершения — primary gloss или user-visible ошибка по kind

#### Scenario [TO-11]: Loader / Translate path not frozen
- **GIVEN** идёт fullscreen gate или cold Translate wait на **native**
- **WHEN** проходит время init модели из поставки
- **THEN** индикатор loading остаётся живым (кадры анимации); UI isolate MUST NOT полностью замирать из‑за синхронной тяжёлой работы без yield и MUST NOT требовать передачи всего ONNX через Dart→FRB `Vec<u8>` как единственный product path

### Requirement: Native engine init from filesystem paths
На native product path система SHALL уметь инициализировать offline translate из **локальных файлов модели** (OS paths или один directory + фиксированные basenames из поставки), после извлечения/копирования из bundled assets при необходимости. Product `ensure` / `warmUp` / gate MUST использовать этот path-based init (или эквивалент без полного bytes-through-UI), а не полагаться только на передачу полных ONNX bytes через FRB на UI isolate. Bytes-based init MAY оставаться доступным для тестов и совместимости. При отсутствии обязательного файла или битом содержимом — `EngineUnavailable` (предпочтительно). На `wasm32` публичные FRB translate символы MAY быть stub; Web UI не вызывает FRB translate.

#### Scenario [TO-14]: Native init from model paths
- **GIVEN** на native доступны валидные локальные файлы модели с требуемыми basenames (после extract из assets или уже на диске)
- **WHEN** вызывается path/dir init API
- **THEN** при валидных файлах init успешен; при отсутствии обязательного файла или повреждённых данных — `EngineUnavailable` (предпочтительно); последующий `translate_text` en→ru может вернуть gloss

## MODIFIED Requirements

### Requirement: Engine readiness and failure signaling
Система SHALL сообщать UI типизированной ошибкой, когда движок недоступен или инференс не удался, так чтобы UI мог показать user-visible ошибку и не вставлять ложный перевод. Пока идёт первая инициализация модели из assets (fullscreen gate / warm-up или lazy init по Translate), система MUST показывать loading (fullscreen до ready/fail на старте сессии; на кнопке Translate при активном запросе после shell). Поддерживаемая пара — только EN→RU; запрос иной пары MUST давать `UnsupportedLanguagePair` (коды case-insensitive; нормализация к `"en"` / `"ru"`).

На native путь система MUST поддерживать инициализацию из локальных model files (paths/dir) для product cold path; bytes `Vec<{name, bytes}>` через FRB MAY оставаться для тестов/compat и MUST NOT быть единственным обязательным product путём, если это вынуждает UI isolate синхронно переносить весь ONNX. Вызов `translate_text` до успешного init MUST возвращать `EngineUnavailable`. На `wasm32` публичные символы MAY быть stub с `EngineUnavailable` (Web UI не вызывает FRB translate).

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
- **GIVEN** Flutter/тест передал полный набор named asset bytes из поставки
- **WHEN** вызывается bytes-based `init_offline_translate`
- **THEN** при валидных файлах init успешен; при отсутствии обязательного файла или битых bytes — `EngineUnavailable` (предпочтительно); `Internal` только если сбой не классифицируется как assets/init
