# Stage12 — Windows smoke checklist

Проверять на Windows 10/11. Stage12 — кандидат для тестирования, без installer. Ничего включать в firewall, BIOS, реестре или службах автоматически не нужно. Пароли NEXUS не сохраняет.

## A. Достаточно одного компьютера

### 1. Чистая папка и запуск

- [ ] Распакуйте `NEXUS-CONTROL-stage12.zip` **в новую пустую папку**, например `C:\Projects\Nexus Stage12\nexus-control`. Не распаковывайте поверх старой версии: иначе старый `starfield.ts` может остаться рядом с `Starfield.tsx`. Правильный engine — `starfieldEngine.ts`.
- [ ] Откройте терминал именно в папке с `package.json`. Нужен Node.js 24 LTS (минимум 22.12) и интернет для npm/Electron. Версии покажут `node --version` и `npm.cmd --version`.
- [ ] Выполните `npm.cmd ci`, затем `npm.cmd run dev`. Проверьте окно и Guide. Завершите dev через Ctrl+C.
- [ ] Выполните по очереди:

```powershell
npm.cmd run typecheck
npm.cmd test
npm.cmd run build
npm.cmd start
```

Не запускайте dev и production одновременно с одним профилем. Пользовательские настройки находятся в Electron userData вне исходников; чистая распаковка проекта их не удаляет. Не удаляйте userData для обновления.

### 2. Безопасная автоматическая диагностика

Закройте NEXUS, затем:

```powershell
npm.cmd run smoke:windows
```

По умолчанию проверяются все клиенты. Если вам нужен отдельный сценарий:

```powershell
npm.cmd run smoke:windows -- --scenario=app
npm.cmd run smoke:windows -- --scenario=rdp
npm.cmd run smoke:windows -- --scenario=ssh
npm.cmd run smoke:windows -- --scenario=wol
```

- **PASS** — указанная проверка пройдена либо необязательный компонент не нужен выбранному сценарию (прочитайте пояснение).
- **WARN** — выбранный клиент отсутствует/недоступен или UDP bind/close не удался; приложение может оставаться пригодным для других сценариев.
- **FAIL** — неверная host OS либо проблема source/build/userData/диагностического Electron renderer. Исправьте её до дальнейшего тестирования. Exit code: 1 для FAIL, 0 при PASS/WARN; неверные аргументы — 2.

Helper проверяет стандартные абсолютные пути SystemRoot/LOCALAPPDATA, наличие файлов mstsc/OpenSSH/Console Host и alias Terminal, версию, архитектуру, case-collisions, production-файлы, запись временного файла в настоящий userData и localStorage в **отдельном временном профиле**. Его Electron-окно невидимо, без preload/IPC; настоящие компьютеры и appearance key не читаются. UDP-проба — только bind на localhost и close, без отправки. Временный файл удаляется только после успешного exclusive create; существующие файлы не удаляются.

Helper **не открывает RDP/SSH**, не отправляет WoL, не делает TCP-проверок компьютеров, не меняет системные настройки и не проверяет remote login. Наличие Terminal alias не доказывает его исправную активацию. Presence/readability executable не доказывает успешный native launch. Appearance-проба подтверждает localStorage backend, а не сохранённые пользовательские preferences: проверьте их вручную ниже.

**OpenSSH Server на целевой машине не является локальным prerequisite.** Его отсутствие не создаёт diagnostic WARN. Если сервер уже настроен на этом же ПК, optional manual SSH к `127.0.0.1` допустим через обычную карточку; helper его не запускает и сервер не устанавливает. Аутентификация и host-key verification остаются стандартными.

### 3. Persistence, интерфейс и lifecycle

- [ ] В «Мои компьютеры» добавьте `SMOKE PC`, адрес `127.0.0.1`, **выключите RDP/SSH/WoL**. Перезапустите всё приложение: запись осталась. Измените имя, перезапустите; удалите, перезапустите: запись отсутствует.
- [ ] По очереди выберите Full/Calm/Minimal и Normal/Compact. Перезапустите: выбор сохранился. Reset appearance не удаляет компьютеры.
- [ ] В DEMO проверьте две/шесть карточек, detail, Add/Edit, ожидание/Отмена, toast/error, Guide home/article, локальный поиск и contextual links. Отключите интернет: Guide продолжает работать. Static copy копирует текст, ничего не выполняет.
- [ ] Tab/Shift+Tab, Enter и Escape: видимый focus, открытие/закрытие диалогов, возврат focus, доступная Отмена. Нет белого экрана и неожиданной ошибки.
- [ ] Размеры 1280×720 и 760×600, maximize/unmaximize, resize, minimize/restore, close/reopen. Нет горизонтального clipping или скачка фона после restore.
- [ ] На каждом масштабе Windows **100%, 125%, 150%** перезапустите NEXUS и проверьте wordmark/logo, Bahnschrift fallback, Segoe UI, кириллицу, header, карточки, Guide, settings и form footer. CSS zoom не нужен. При нескольких мониторах перенесите окно между DPI.
- [ ] Full: фон движется мягко к курсору; Calm спокойнее; Minimal без постоянной анимации. Windows «Эффекты анимации» выключены: reduced-motion уменьшает движение. При свернутом окне фон не расходует ресурсы; после restore не ускоряется.
- [ ] Если локальная RDP/SSH-служба уже включена, можно добавить localhost с соответствующей capability и нажать «Обновить». Отказ порта — нормальный результат без сервера; он не доказывает ошибку NEXUS. Проверка не вводит пароль и не запускает клиент.

## B. Нужен разрешённый target PC

Только собственное/разрешённое устройство, с уже вручную настроенными RDP/SSH/WoL. NEXUS не меняет настройки target.

- [ ] RDP: «Рабочий стол» открывает mstsc с правильным адресом. Выполните настоящий login в стандартном клиенте. Закройте NEXUS: RDP-сессия остаётся.
- [ ] SSH: Terminal открывает интерактивный OpenSSH с правильным пользователем (`user`, `DOMAIN\user` или UPN). При отсутствии Terminal используется Console Host. Проверьте пароль/host key штатно; закрытие NEXUS не закрывает сессию.
- [ ] WoL: один клик отправляет один контролируемый burst; физическое устройство включается при поддержке BIOS/NIC/сети. Отправка пакета сама по себе не подтверждает включение.
- [ ] Оба combined flow: WoL → ожидание выбранной службы → клиент; Отмена и timeout до 90 секунд работают, повторный клик не открывает дубликат. Убедитесь, что отменённый сценарий позже не запускает клиент.
- [ ] При уже настроенном VPN/remote endpoint проверьте local→remote fallback. NEXUS VPN не устанавливает.

## Если npm предупреждает об esbuild / allow-scripts

Не используйте blanket approval, `approve --all`, отключение защит или произвольное выполнение install scripts. Сначала сохраните точный warning и версии npm/Node. Postinstall esbuild проверяет версию native binary и оптимизирует CLI; пропуск может быть неблокирующим, **только если ваши clean `npm ci`, typecheck, build и startup действительно прошли**. При ошибке missing native binary или Electron установка не считается успешной. Потребуется отдельная диагностика именно этого dependency/версии и вашей npm policy, без общего разрешения scripts.

В текущей Linux-среде обычный clean npm ci проверен отдельно; его результат и ограничения указаны в [STAGE12.md](./STAGE12.md). Это не Windows install smoke. Официальные справки: [npm install-script policy](https://docs.npmjs.com/cli/v11/commands/npm-install-scripts/) и [esbuild installation](https://esbuild.github.io/getting-started/).

## Короткий отчёт после ручной проверки

Запишите Windows build, Node/npm, архитектуру, DPI, PASS/WARN/FAIL helper и какие пункты A/B реально проверили. Не присылайте пароли, private keys, terminal contents или полный environment. Для FAIL достаточно безопасного имени проверки и сообщения. Native Windows smoke пока не выполнен в среде разработки.
