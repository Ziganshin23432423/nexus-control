# NEXUS CONTROL — STAGE13

Windows Installer / Release Packaging, 2026-10-02. База — приложенный стабильный Stage12, без обращения к зависшему предыдущему Work-сеансу.

**Результат: исходники Stage13 и Windows x64 unpacked application готовы. Готового NSIS installer нет.** Cross-build штатного NSIS блокируется запретом Unix sockets в Wine. Не выдаём промежуточный builder EXE за установщик и не заявляем проверку Windows installation. Stage14 не начат.

## Initial audit

Stage12: version 0.1.0, package name nexus-control; Electron 44.5.1; React/React DOM 19.3.0; TypeScript 7.0.2; Vite 7.3.6; electron-vite 5.0.0; electron-store 11.0.2. Версии этих direct dependencies сохранены. Node 24.19.0 / npm 11.9.0 в среде. До Stage13 appId/installer/ICO отсутствовали; public nexus-mark.svg уже содержал фирменную N.

Main ESM `out/main/main.js`; sandboxed single CJS preload `out/preload/preload.cjs`; renderer `out/renderer`. Production security: nodeIntegration=false, contextIsolation=true, sandbox=true, webSecurity=true; прежний custom protocol nexus://app, CSP, permissions/navigation/sender guards. 15 bridge methods. Реальные RDP/SSH провайдеры и WoL подключаются только на Windows. UserData выбирается после app.setName('NEXUS CONTROL'): `%APPDATA%\NEXUS CONTROL`. Store computers.json/schemaVersion 1; appearance nexus.appearance.v1 в Chromium storage того же профиля/origin.

Исходный архив не содержал node_modules/out. Чистый npm ci Stage12 прошёл за 25 секунд. Прочитаны README, AGENTS, ROADMAP, electron-vite/TypeScript configs и Stage12 reports. GitHub plugin вернул пустой список доступных repositories; remote не подключался, push/PR не создавались. Локальный git baseline использован только для проверки scope.

## Изменения

- electron-builder 26.15.3 и @electron/asar 4.3.1 добавлены как exact devDependencies; lockfile обновлён.
- package version 1.0.0-rc.1; author network.onion; private/UNLICENSED. Public app.getVersion берёт RC из package.json.
- `electron-builder.json`: appId **network.onion.nexus-control**, productName **NEXUS CONTROL**, Windows buildVersion **1.0.0.1**, buildNumber **1**. APP_ID передаётся Windows app.setAppUserModelId.
- В BrowserWindow добавлен icon: packaged resources/icon.ico; dev — project build/icon.ico. Других runtime изменений нет.
- ICO, master PNG и копия исходного SVG в build; generator из master PNG/Pillow и contact sheet.
- `pack:win`, `dist:win`, `verify:package`, `test:packaging`; две packaging tests включены в обычный npm test. Read-only inspector парсит PE/resources и ASAR без запуска Windows EXE.
- README, Stage13 report, release notes, отдельный installer smoke. ROADMAP не менялся.

## Packaging / NSIS

Windows x64; asar=true; files whitelist out/**/* + package.json и штатные production dependencies. ExtraResource только icon.ico. Build tools, исходники, QA, docs, profiles и cache не идут в app.asar. npmRebuild=false: текущие production dependencies не содержат native addons. При будущих native dependencies эту настройку нужно пересмотреть. Electron runtime/license/locales поставляются целиком штатным packager.

NSIS штатный oneClick, perMachine=false, allowElevation=false, packElevateHelper=false. Shortcut в «Пуск», Desktop shortcut выключен; runAfterFinish=false — запуск пользователь делает явно. Default install dir для этой oneClick конфигурации: `%LOCALAPPDATA%\Programs\nexus-control`; executable NEXUS CONTROL.exe. uninstallDisplayName/shortcutName NEXUS CONTROL, publisher из author network.onion. appId определяет стабильный installer GUID; менять его в последующих выпусках без миграции нельзя.

Custom include/script **нет**. Стандартный NSIS template содержит uninstall, shortcut/registration cleanup. `deleteAppDataOnUninstall=false`: обычное удаление сохраняет APPDATA. Явный штатный аргумент `--delete-app-data` может удалить данные — checklist его не использует. Отдельные config/static tests и template audit подтверждают выбранные параметры; Windows registry/UAC/filesystem/uninstall фактически не выполнялись.

Никакой автоочистки или смены userData, storage migration/IPC не добавлено. Reinstall/upgrade safety на настоящей Windows всё ещё требует smoke: статическая конфигурация не доказывает сохранение файлов на конкретном ПК. Для unpacked ZIP используется тот же профиль; это не NSIS portable target.

## Versions / signing

Public package/app version 1.0.0-rc.1. PE app FileVersion/ProductVersion и fixed numeric metadata: 1.0.0.1. NSIS final product string по штатному генератору — public RC, FileVersion numeric; финальный installer ещё не собран/не проинспектирован. `buildNumber=1` обеспечивает одинаковый numeric product version во всех генераторах.

signAndEditExecutable=true сохраняет resource editing; signExecutable=false делает RC явно unsigned. forceCodeSigning=false, publish=null и --publish never. Нет certificate или updater. PE security directory приложения пуст — signature отсутствует. Для будущего signing включить signExecutable, настроить сертификат стандартным electron-builder способом и require forceCodeSigning=true; runtime architecture менять не требуется. SmartScreen/reputation проверяется отдельно на Windows, защита не отключается.

## Icons

Фирменный SVG Stage12 побайтно сохранён. Transparent master 1024px отрендерен из него; ICO включает **16,20,24,32,40,48,64,128,256**. Иконки реально обнаружены в RT_GROUP_ICON packaged executable; resources/icon.ico соответствует исходному ICO. Installer/uninstaller config использует тот же asset. Contact sheet просмотрен, мелкие размеры узнаваемы; taskbar/shortcut/Explorer/installer visual и DPI ещё требуют Windows.

![Размеры NEXUS ICO](./stage13/icon-sizes.png)

## Regression

| Проверка | Результат |
|---|---|
| Typecheck main/web/tests/smoke | PASS |
| Unit + packaging tests | **199/199**, skipped/cancelled/fail 0 |
| Production build | PASS |
| Stage12 readiness | 6/6 |
| Stage11 appearance | 17/17 |
| Stage10 Guide | 14/14 |
| Stage9 UX | 22/22 |
| Stage8 orchestration | 13/13 |
| Stage7 WoL | 11/11 |
| Stage6 SSH | 10/10 |
| Stage5 RDP | 10/10 |
| Stage4 network | 12/12 |
| Stage3 CRUD | 13/13 |
| Electron total | **128/128** |
| Appearance combinations | **120/120** |
| Windows unpacked cross-build | PASS |
| PE + icon/version/manifest/ASAR inspection | PASS |
| Final NSIS installer | **BLOCKED: Wine AF_UNIX** |
| Native Windows install / login / physical WoL / DPI | **NOT TESTED** |

Существующие suites выполнены node tests/electron-{readiness,appearance,guide,ux,orchestration,wol,ssh,rdp,network,crud}.cjs строго от новых к старым. Readiness bundle через build:smoke/build:readiness-qa. Никакие assertions не ослаблены. Реальный Linux Electron + Xvfb TCP, изолированные профили; --no-sandbox только внешним QA argument, production preference остаётся sandbox=true. Windows providers — DI, TCP/UDP — loopback где возможно. Xvfb visibility fallback и Linux TCP22 limitation прежние; не являются Windows hardware proof.

Xvfb первоначально запускался отдельной tool process namespace и не был видим QA. Исправлено orchestration среды: Xvfb и suites в одном parent process; весь cycle затем прошёл. Первоначальный запуск readiness без display отмечен как environment attempt, не PASS. apt installation ограничивался sandbox permission/lock; X11 deb packages распакованы для временного QA. Приложение из-за этих ограничений не менялось.

## Cross-build attempts / hang protection

1. dist:win: production build и Windows package успешно; NSIS intermediate compiled, затем `spawn wine ENOENT`.
2. Поддерживаемый electron-builder wine toolset 1.0.1 (Wine 11): download успешен; wineserver сразу отказал `socket: Operation not permitted`. Это ограничение Unix sockets среды. Повторять или запускать Windows installer GUI не стали.
3. pack:win --dir — отдельная успешная сборка unpacked приложения, без повторения NSIS failure.

Все долгие команды имели wall watchdog (90–180 секунд, финальный pack 130), QA каждую suite 100 секунд/appearance 150 и cleanup process group при timeout. Не было многократного повтора одной причины или многочасового ожидания. Wine здесь использовался только штатным builder для **извлечения uninstaller**, не для заявления Windows smoke. Intermediate ~168KB EXE, NSIS payload archive и incomplete artifacts исключены из deliverables.

## Artifact inspection / security audit

PE: x64/unsigned; CompanyName network.onion, ProductName NEXUS CONTROL; versions 1.0.0.1; manifest asInvoker; девять icon sizes. ASAR 10,287,769 bytes, 703 entries, roots node_modules/out/package.json. No application tests/docs/src/scripts/build/git/env/profiles/credentials files or electron-builder/TypeScript/updater inside ASAR. Проверены entry main/preload/renderer и public RC. Electron runtime крупнее app payload; это ожидаемый runtime, не accidental build dependencies. Upstream dependency auxiliary files/maps остаются штатно в production modules.

119 runtime source files неизменны; изменены только src/main/main.ts и src/shared/constants.ts (AUMID/window icon/constant). Renderer/Guide, preload, IPC/security/protocol, store/schema/migrations, network/orchestration/providers unchanged. Новых secret fields, stored credentials, shell execution, IPC, telemetry, autostart/service/agent/updater нет. Secret-pattern scan собственных файлов не нашёл private keys/API token markers; это scoped inspection, не внешняя security certification. Подробности: [security audit](./stage13/security-audit.json), [package inspection](./stage13/package-inspection.json), [NSIS audit](./stage13/nsis-audit.json), [regression results](./stage13/regression-results.json). Runtime application functionality не расширялась.

## Delivery

| Артефакт | Статус |
|---|---|
| NEXUS-CONTROL-Windows-x64-1.0.0-rc.1.zip | 159,814,091 bytes; готовый unpacked Windows application |
| NEXUS-CONTROL-stage13.zip | Полные чистые исходники/config/assets/tests/docs; без node_modules/out/release/git/profiles |
| STAGE13.md | Этот отчёт |
| WINDOWS-SMOKE-STAGE13.md | 16 шагов с ожидаемыми результатами и точными build commands |
| RELEASE-NOTES-1.0.0-rc.1.md | Release notes |
| SHA256SUMS.txt | SHA256 всех доставленных файлов; source ZIP hash внешне, чтобы избежать рекурсивного checksum |
| NEXUS-CONTROL-Setup-1.0.0-rc.1.exe | **НЕ ПОСТАВЛЯЕТСЯ**: финальная NSIS сборка blocked |

Windows application ZIP SHA256: `5af90a5148208c51090c0fce06173946498514e215ea8b10e93a146ba87f213d`.

Распаковать весь Windows ZIP в новую папку, запустить NEXUS CONTROL.exe. Нельзя переносить только EXE без resources/runtime. Node/npm не нужны для этого готового приложения. Данные вне папки приложения. Источник распаковать отдельно в пустую папку; команда на Windows: npm.cmd ci → npm.cmd run dist:win → npm.cmd run verify:package. [Release notes](./RELEASE-NOTES-1.0.0-rc.1.md) · [Windows smoke](./WINDOWS-SMOKE-STAGE13.md).

Официальные основания config: [electron-builder NSIS](https://www.electron.build/docs/nsis/), [Windows](https://www.electron.build/v26/docs/win/); фактическое поведение сверено с установленными templates/types версии 26.15.3. Настоящий Windows smoke полностью остаётся открытым gate. Stage13 завершён в пределах доступного cross-build; Stage14 не начат.
