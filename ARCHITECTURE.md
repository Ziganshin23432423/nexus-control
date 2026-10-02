# Windows first. Cross-platform architecture. Linux later.

Windows 10/11 — платформа №1 и единственная поддерживаемая платформа текущего MVP. Функции сначала получают реализации для Windows. Полноценная Linux-поддержка добавляется отдельным этапом по запросу пользователя.

## CURRENT STATE — Stage 10

| Capability на Windows host | Текущая реализация |
| --- | --- |
| Remote desktop | `WindowsRdpProvider`: системный `mstsc.exe`, safe spawn |
| Interactive SSH | `WindowsSshProvider`: Windows OpenSSH + Windows Terminal; fallback **Console Host**, без PowerShell/cmd |
| Wake-on-LAN | `UdpWakeOnLanProvider`: общий Node.js `dgram` UDP4 transport, fixed burst 3 × 102 bytes |
| Platform composition | `createPlatformService('win32')` подключает все три provider; Linux/unsupported host оставляет все slots null |

`RemoteDesktopService` и `SshTerminalService` получают storage, общий `CheckCoordinator`/`NetworkService` и provider через DI. Перед запуском — fresh TCP3389/TCP22, local → remote fallback, общий pool. Один pending launch на ID защищает RDP и SSH. Standard client владеет авторизацией; launch result её не подтверждает.

`WakeOnLanService` независим от NetworkService: ID → ComputerStore.get → shared validation → Windows MVP eligibility → MAC/broadcast/port → provider → typed отправка. Он не проверяет TCP, не ждёт включения и не меняет runtime status. `shared/magicPacket.ts` формирует стандартный FF × 6 + MAC × 16. Internal provider request включает только проверенные параметры, стандартный packet и AbortSignal; provider повторно проверяет destination и точное соответствие packet MAC. Renderer не передаёт их: только `wakeComputer(id)`.

UDP provider создаёт один udp4 socket, bind(0), setBroadcast(true), три send callback с интервалами 100 ms. Deadline — 2500 ms на всю отправку. Success/error/timeout/abort закрывают socket и очищают timers/listeners; Promise завершается после close. Pending WoL guard отдельный, не блокирует RDP/SSH. CRUD/закрытие приложения отменяет ещё активный burst.

Пользовательские компьютеры сохраняются через electron-store, schemaVersion = 1 без изменения. Runtime статусы и pending операции остаются в памяти. DEMO маршрутизируется первым и не обращается к user/network/provider IPC. `wakeComputer` возвращает `packet-sent`, а не online. Direct wake не ждёт и не подключается; отдельные fixed combined flows реализованы в Stage 8.

IPC: metadata, persistent CRUD, TCP status, `openRemoteDesktop(id)`, `openSshTerminal(id)`, `wakeComputer(id)`. Явный preload allowlist; trusted main window/frame/exact URL guard, UUID и runtime validation в main. Нет универсального UDP, process или command API. Защита Electron сохранена: nodeIntegration false, contextIsolation true, sandbox true.

## Operational UX Hardening (Stage 9)

Новых providers/сетевых capabilities нет. `renderer/state/operationUx.ts` задаёт общую UX-терминологию idle/checking/sending/waiting/launching/success/cancelled/timeout/error, whitelist всех typed error codes, retry policy, безопасные success titles, formatter countdown, capability combinations и технические сведения. Это utility для представления; универсального main execution engine/API нет.

`notifyError` использует только code и безопасную служебную метаинформацию, не raw error.message/exception. Version берётся из AppInfo. Retry — callback конкретного существующего действия, только по нажатию; mounted/enabled refs и очистка toast при source switch сохраняют demo isolation. Success означает только client opened / terminal opened / packet sent. `useToasts` ограничивает очередь тремя, подавляет одинаковые видимые сообщения и краткие повторы; timers вытесненных toast очищаются. Error/warning не исчезают до закрытия/вытеснения, success/info — 4500 ms.

Progress Stage8 расширен необязательным main-owned `remainingSeconds`. В production вычисляется по monotonic deadline после WoL ACK. Один cancellable timer публикует waiting примерно раз в секунду; в launching и finally он очищается. Это UI metadata: renderer не задаёт timeout/deadline, не создаёт countdown timer и не запускает дополнительные probes. Пакеты, checks, интервалы и deadline Stage8 не менялись.

Network refresh имеет renderer duplicate guard, generation/snapshot checks и проверку полноты/уникальности/coherence результатов: онлайн допустим только при ready настроенной службы. Проверка одного ID допускает только выбранный target. Cancel возвращает проверенный baseline либо unknown и показывает пояснение; polling нет.

`AppErrorBoundary` охватывает React-tree и безопасно реагирует на render/lifecycle errors, window.error и unhandledrejection. Fallback не раскрывает исключение. Перезапуск интерфейса remounts App: перечитывает USER DATA, отменяет pending combined flows при unmount, снимает подписку и очищает timers. Main/native sessions не перезапускаются. Navigation/security прежние; privileged reload/clipboard IPC не добавлены. Splash metadata retry повторяет только getAppInfo.

Корректность capability layout проверяется в семи сочетаниях на unit и Electron UI уровнях. Storage corruption/future schema остаются защищёнными прежними main checks; UI предлагает reload данных или demo, без reset. Computer/storage schema и providers не изменены. Stage10 Guide и Stage11 redesign отложены.

## Fixed wake → wait → connect (Stage 8)

`WakeAndConnectService` предоставляет ровно два сценария: `openRdp(id)` и `openSsh(id)`. Он загружает и валидирует Computer, WoL/service config и доступность соответствующих host providers **до отправки**. После этого вызывает Stage7 `WakeOnLanService.wakeOwned` ровно один раз. Далее использует только `CheckCoordinator.checkRdp` или `checkSsh`, включая прежний local → remote fallback и общий pool.

Sequential async loop проверяет выбранную службу; после unavailable следует cancellable delay **2000 ms**. Нет overlapping checks и setInterval. **90-second global deadline** начинается после WoL ACK и включает TCP attempts, ожидание pool, паузы и финальные проверки запуска. Deadline прерывает текущий AbortSignal, а не просто останавливает будущие ticks. Production clock — monotonic performance.now и реальные timers; тесты инъектируют virtual clock.

После READY существующий Stage5/6 service вызывается через main-only `openOwned(id, ownership)`. Он повторно читает/валидирует запись, сравнивает original snapshot и делает **новую fresh проверку** перед native provider. Если порт снова исчез, сценарий возвращает controlled failure; он не запускает клиента по WAIT result и не продолжает скрытое ожидание. Разные RDP/SSH endpoints выбираются существующей NetworkService логикой.

Ownership использует уже существующую общую launch map: flow резервирует ID одним AbortController до первого await. Direct RDP/SSH и другие flow получают BUSY, другие ID независимы. Owned launch допустим только с тем же controller в map; он не снимает reservation — cleanup принадлежит flow. Set WoL claims предотвращает второй direct wake во время flow. `LaunchOwnership` существует только main, не является IPC payload.

Cancel aborts UDP при pending wake, TCP и delay при ожидании, исключает дальнейшие checks/launch. Уже отправленный пакет не отменяется. После успешного client ACK reservation освобождён и cancel не закрывает client/session. CRUD update/delete отменяют соответствующий ID; создание записи не отменяет чужие flows. Shutdown отменяет все активные операции. Snapshot comparison также защищает от external file edits/deletes.

Три narrow IPC: `wakeAndOpenRemoteDesktop(id)`, `wakeAndOpenSshTerminal(id)`, `cancelWakeAndConnect(id)`. Каждый отвергает extra args и проходит прежний trusted sender/frame/window/exact URL guard. Единственная новая read-only subscription `onWakeAndConnectProgress(listener)` получает main→renderer computerId/service/phase (sending/waiting/launching), без raw event или ipcRenderer. Renderer работает с ID/callbacks и не зависит от Windows providers.

Result расширяет прежний `client-opened` / `terminal-opened` результат полями wakeSent и waitedMs; authentication не утверждается. UI показывает отправку, ожидание выбранной службы, launch и Отменить. Runtime progress не записывается в electron-store. Polling существует только при явном start выбранного ID и прекращается по ACK/error/cancel/timeout. DEMO ветка сохраняет mock timers и не вызывает user IPC. Произвольных workflows/commands/scheduler/Agent нет.

## Две разные ОС

`Computer.operatingSystem` описывает удалённый компьютер: `windows` или `linux`. `PlatformService.hostPlatform` описывает компьютер, на котором запущено приложение: `windows`, `linux` или `unsupported`.

Эти значения независимы. В будущей версии Windows-клиент сможет обращаться к Linux-компьютеру через подходящий провайдер. Тип `linux` в текущей модели резервирует такую возможность; в MVP при добавлении устройства используется `windows`.

## Границы зависимостей

UI вызывает конкретный разрешённый метод preload по идентификатору компьютера. IPC handler в main проверяет отправителя и входные данные, загружает конфигурацию и передаёт её общему сервису. Общий сервис получает провайдеры как зависимости и не выбирает исполняемые файлы по ОС.

Platform service / composition root выбирает native implementations. Только провайдеры знают о `mstsc.exe`, Windows OpenSSH, Windows Terminal и будущей Windows Credential Manager. Команды запуска, поиск клиентов и платформенные сообщения об ошибках относятся к провайдерам и Windows-модулям.

Контракты находятся в `src/main/providers/contracts.ts`. Renderer не импортирует main process, провайдеры, Node.js или Electron API; его контракт — `NexusApi` из shared.

| Контракт | Назначение | Windows: CURRENT STATE | Linux: позже |
| --- | --- | --- | --- |
| `RemoteDesktopProvider` | Запустить выбранный desktop client | `WindowsRdpProvider`: mstsc.exe | `LinuxRemoteDesktopProvider`: выбранный клиент |
| `SshProvider` | Открыть интерактивную SSH-сессию | `WindowsSshProvider`: OpenSSH и Terminal / Console Host | `LinuxSshProvider`: native SSH и terminal |
| `WakeOnLanProvider` | Отправить UDP Magic Packet | Общий Node.js UDP transport | Общий transport при будущей поддержке |
| `PlatformService` | Дать host platform и провайдеры | `createPlatformService` composition | `LinuxPlatformService` |

## Общая логика

| Область | Где должна находиться |
| --- | --- |
| Computer model, `operatingSystem`, API types | `shared` без Node.js imports |
| UI и статусы | `renderer` с общим API |
| Validation | Общие чистые функции; main повторно проверяет входные данные |
| TCP checks и timeouts | Общие сервисы main; доступны для мокирования |
| Генерация Magic Packet | Общая чистая функция, независимая от ОС |
| UDP-отправка | `WakeOnLanProvider` в main |
| Включить, ждать, отменить, подключиться | `WakeAndConnectService` в main с injected services/clock, два fixed flow (Stage 8) |
| Версионируемая конфигурация | Общий storage в main через electron-store, реализован этап 3 |
| Native clients и Windows integrations | Отдельные Windows-провайдеры и platform modules |

«Общий» здесь означает независимый от ОС. Это не разрешение renderer использовать Node.js API или выполнять системные действия.

## История — этапы 1–2 (не CURRENT STATE)

`createPlatformService` предоставляет только метаданные host platform и пустой реестр провайдеров. Слоты `remoteDesktop`, `ssh` и `wakeOnLan` равны `null`. Контракты не создают кнопок подключения, не запускают программы и не отправляют пакеты.

На этапе 2 main, preload, platform service и контракты провайдеров сохранены. Добавлены общие типы статусов, React-экраны и отдельный demo controller в renderer. Он использует `Computer` и имитирует действия через таймеры в памяти; системные контракты пока не вызывает. Linux в форме показан недоступным. Production также работает только в demo mode, без скрытого переключения на настоящие действия.

Предусмотрены neutral request types с адресом, портом и username; паролей и shell-строк в них нет. TypeScript-типы не заменяют runtime validation: перед вызовом провайдера main обязан проверить реальные данные.

`open()` означает запуск стандартного клиента. Аутентификация, password prompt и содержимое RDP/SSH-сессии принадлежат этому клиенту. Успешный запуск программы не считается успешной аутентификацией.

Linux-модули, Linux-клиенты, xrdp, VNC, SFTP и remote management сейчас не реализуются и не тестируются. Windows-функции также добавляются только в назначенных этапах. Дополнительного универсального IPC или выполнения произвольных команд эта архитектура не допускает.

## История — этап 3 (не CURRENT STATE)

Системные провайдеры по-прежнему не реализованы. Добавлены четыре узких CRUD IPC метода, shared validation и общий storage в main. `ComputerDraft` не содержит ID и дат: main назначает их после повторной проверки входных данных. `IpcResult` отделяет успешный результат от безопасной ошибки.

`ComputerStore` работает через injected `ConfigurationRepository`; production adapter использует electron-store, unit-тесты — отдельный memory repository. Версионируемый envelope `configuration.schemaVersion` проверяется при каждом чтении. `migrateConfiguration` — точка входа будущих последовательных миграций; первая схема равна 1. Некорректные и будущие форматы не сбрасываются.

По умолчанию renderer открывает пользовательский список через `useUserComputers`. Демо-контроллер остаётся отдельным, без IPC и persistence. UI обновляется после подтверждённой записи main. Для пользовательских записей статус `unknown` означает отсутствие проверки сети; конфигурация сервиса не означает его доступность. Вычисляемые статусы, таймеры и demo fixtures не сохраняются.

## Будущее: Automations / Scripts

В [ROADMAP](../ROADMAP.md) отмечен отдельный будущий модуль: визуальные сценарии из безопасных действий NEXUS и advanced scripts, созданные пользователем. Для Windows рассматривается PowerShell, для будущей Linux-поддержки — Bash.

Будущий модуль сможет использовать общие сервисы и платформенных провайдеров через существующие границы. Хранение конфигурации компьютеров не должно требовать наличия Automation Engine или намеренно запрещать его последующее добавление. Перестраивать текущую архитектуру и вводить script-поля или новые контракты ради этого сейчас не требуется.

Принципы будущего исполнения: явный запуск пользователем, видимый целевой компьютер, разрешения, журналирование и отмена; без скрытого исполнения и хранения секретов открытым текстом. Подробное проектирование отложено до отдельного запроса.

На этапе 3 scripts, произвольное выполнение команд, script IPC, PowerShell execution, scheduler и remote execution не реализуются. Существующий запрет универсального shell IPC сохраняется.

## NEXUS GUIDE (Stage 10)

Renderer-only информационный модуль `renderer/guide`. `GuideArticleId` — literal union 59 стабильных IDs; каталог объединяет шесть typed content modules. Блоки — paragraph/steps/callout/command/checklist/data-card; команды — отдельный frozen allowlist четырёх static texts. React рендерит текст, не HTML/Markdown-код. Внешние ссылки только plain text из bundled источников.

`searchGuide` строит локальный deterministic index title/description/keywords/block text, NFKC → русский lower case → ё/е → punctuation/whitespace. AND token matching, фиксированный score и стабильный порядок. Нет API, AI, fetch и dependencies. Все данные bundled Vite; статьи/search/checklist работают offline.

`GuideContext` предоставляет только renderer navigation. Main/preload и 15 API методов не менялись. Parent hooks остаются mounted; открытие Guide не меняет источник данных, не вызывает status checks и не запускает providers. `GuideDialog` — отдельная read-only native dialog поверх ComputerFormDialog, черновик не размонтируется, Escape возвращает фокус. Article/back/breadcrumb используют внутренние IDs, не URL.

`ErrorPresentation.guideArticleId` — необязательная renderer metadata из mapping 13 typed error codes. Main semantics/retry policy неизменны. Toast вызывает только навигационный callback. Field mapping и service help ведут на статьи каталога. Storage/schema/model/dependencies не менялись, новых secret fields нет. Live enabled ref и очистка toast при source switch защищают от старого USER retry в DEMO.

Stage10 audit: 48 защищённых файлов main/preload/shared/config/dependencies/ROADMAP byte-identical принятому Stage9. Старые Electron проверки получили только уточнённые text-input selectors, поскольку рядом появились отдельные labelled help buttons; assertions и сценарии сохранены.

История Stage10: глобальная typography/cards/motion система не менялась. В Stage11 визуальная интеграция Guide завершена вместе с общей design system; content/search/security остались прежними.


## Appearance / NEXUS Motion System (Stage 11)

Renderer-only модуль `appearance`: `preferences.ts` — строгая validation двух полей, storage adapter и profiles; `AppearanceProvider` — lifecycle/settings/media query; `AppearanceDialog` — native dialog с keyboard/focus/reset; `StarfieldEngine` — pure lifecycle/render scheduler; `Starfield` — Canvas 2D DOM driver. Provider окружает Starfield и ErrorBoundary/App. Фон остаётся декоративным и не блокирует controls (`aria-hidden`, `pointer-events:none`).

`nexus.appearance.v1` — отдельная renderer localStorage запись, не configuration schema. Defaults full/normal. Read не создаёт key; write/reset ловят storage exceptions; malformed values не попадают в UI. Reset удаляет только этот key. Нет main/preload IPC, remote providers, secrets или новых dependencies.

Effective effects = minimal при `prefers-reduced-motion`; сохранённый user mode не переписывается. Full/Calm/Minimal определяют continuous Canvas profile и CSS tokens. Density задаёт spacing/control layout, без zoom. `styles/tokens.css` содержит palette, surfaces, typography, spacing, radii, shadow и duration/easing; остальные styles применяют их к screens. UI sans-serif, technical values mono.

Engine owns at most one RAF. Pointer/target/interpolation/time — plain instance state, не React setState. Draw ограничен сверху 30 FPS в Full, 20 в Calm; Minimal не планирует RAF и не подписывается на pointer. Stars — seeded data (70–220, 3 слоя), не DOM/React components. DPR cap 1.5. ResizeObserver и window resize меняют Canvas backing size; document hidden отменяет RAF и замораживает active time. Effect cleanup вызывает stop, отменяет frame и удаляет pointer/visibility/resize listeners + observer. Guards игнорируют уже поставленные callbacks после stop.

Переходы: page fade/малый translate, route-keyed Guide content, modal/backdrop, control hover/press/focus, toast entrance. Minimal оставляет короткие fade/controls; системный reduced-motion убирает CSS animations/transitions. Motion не управляет timeout, retry, TCP, provider launch или workflow ownership.

Форма сохраняет native disabled fieldset внутри отдельной scroll surface; header/footer вне scroll geometry. Состав actions не менялся, добавлена лишь layout class для двух фиксированных combined buttons. Toast semantic и лимит три сохранены.

Stage11 audit: 48 main/preload/shared/package/build/ROADMAP files byte-identical Stage10; 15 Guide data/metadata .ts files unchanged. Нет Linux providers, shell/PowerShell/arbitrary execution или installer. QA instrumentation/DI находится только в tests. Linux/Xvfb limitations и native Windows smoke checklist: `docs/STAGE11.md`.


### Stage11 Visual Direction Correction

Presentation layer `visual-direction.css` уточняет surface hierarchy, cards/action weighting, controls, reference Guide и Appearance previews. Превью — только aria-hidden CSS, не настройки/исполнение. Canvas population остаётся capped 220; adaptive density /4400 и depth alpha усилены. Full parallax 4/11/22 px; single RAF, lifecycle, storage schema, Calm/Minimal/reduced-motion architecture сохранены. Новая QA-проверка отслеживает фактическое смещение Canvas arc positions только в tests. Никаких изменений main/preload/shared/Guide data/provider/network/IPC.


### Stage11C brand and Windows module resolution

`BrandWordmark` и открытый `NexusMark` — reusable presentation components. Bahnschrift/display stack ограничен identity/navigation/titles; body остаётся Segoe UI Text, technical values — Cascadia Mono. `brand-typography.css` уточняет typography/dividers без изменения motion/appearance semantics. SVG app mark — renderer public asset, не native installer icon.

Pure engine переименован `starfield.ts` → `starfieldEngine.ts`, чтобы Windows extensionless resolution не смешивал его с `Starfield.tsx`. Engine bytes и preferences/provider architecture прежние. Unit regression проверяет case-insensitive paths и module stems across extensions. Main/preload/shared/providers/network/store/Guide content/IPC неизменны.


### Stage12 Windows readiness

WindowsRdpProvider и SSH discovery используют общий `trustedWindowsPath.ts`: drive-absolute directories, Unicode/внутренние пробелы допустимы; slash/traversal, expansion, UNC, reserved names, trailing dot/space и control characters отвергаются до доступа к файлам. Фиксированные System32/WindowsApps targets, аргументы и spawn options прежние.

`main/readiness` — отдельные диагностические функции (source/asset case audit, archive cleanliness, PASS/WARN/FAIL mapping и exclusive writable probe). Они не зарегистрированы в composition root/IPC. `scripts/windowsSmokeLauncher.ts` запускает только установленный npm Electron с фиксированным диагностическим entry, shell:false и bounded watchdog. `windowsSmokeMain.ts` использует прежние security/protocol helpers в минимальном изолированном renderer без preload, IPC или настоящих компьютеров. Actual userData не читается, только exclusive own-file write/cleanup. UDP — localhost bind/close, zero send. Appearance schema/storage и production renderer не изменены.

Клиенты диагностируются только по доверенным абсолютным путям, без PATH. Scenario-specific missing clients дают WARN, irrelevant clients не предупреждают, core prerequisites дают FAIL. Target OpenSSH Server не локальный prerequisite. Helper не доказывает native Terminal activation, session authentication, Windows DPI или физическое WoL. `docs/WINDOWS-SMOKE.md` — manual gate перед Stage13. DI/instrumentation для regression остаются tests-only; безопасный smoke не является privileged app API.
