# Stage13 — Windows 10/11 smoke

RC unsigned. Готового NSIS installer в этой поставке нет: Linux cross-build блокируется запретом Unix sockets в Wine. ZIP Windows-x64 содержит готовое приложение для ручного запуска, без установки и uninstall entry. Не считать запуск ZIP проверкой installer.

## Получить installer на Windows

Распакуйте исходники в новую папку. Node.js 24 LTS, минимум 22.12; Windows x64. В папке package.json:

```powershell
npm.cmd ci
npm.cmd run typecheck
npm.cmd test
npm.cmd run dist:win
npm.cmd run verify:package
Get-FileHash .\release\NEXUS-CONTROL-Setup-1.0.0-rc.1.exe -Algorithm SHA256
```

Ожидается `release\NEXUS-CONTROL-Setup-1.0.0-rc.1.exe` и успешная проверка приложения/installer. Если сборка не завершилась успешно, промежуточный EXE не использовать. `npm.cmd run pack:win` создаёт только `release\win-unpacked`.

## Проверка: действие → ожидаемый результат

Перед проверкой закройте все dev/ZIP/installed экземпляры NEXUS. Скопируйте существующую `%APPDATA%\NEXUS CONTROL` в отдельную резервную папку; не удаляйте исходную. Если данных нет, отметьте это. Не отправляйте содержимое конфигурации или учётные данные.

| № | Действие | Ожидаемый результат |
|---|---|---|
| 1 | Запустить успешно собранный Setup обычным пользователем | Per-user установка без запроса администратора. Unsigned RC может вызвать SmartScreen/reputation warning; имя издателя сертификата не подтверждено. Не отключать защиту Windows. |
| 2 | Проверить папку и меню «Пуск» | По умолчанию `%LOCALAPPDATA%\Programs\nexus-control`, `NEXUS CONTROL.exe`; ярлык «NEXUS CONTROL» в «Пуск». Desktop shortcut не создаётся. Нет автозапуска/службы. |
| 3 | Запустить из «Пуск» | Окно с заголовком NEXUS CONTROL и фирменной N; проверить taskbar/Alt+Tab/Explorer/ярлык и кириллицу при DPI 100/125/150%. |
| 4 | Открыть существующие данные Stage12 | Те же компьютеры, schemaVersion 1, прежние Full/Calm/Minimal и Normal/Compact. Путь `%APPDATA%\NEXUS CONTROL` прежний. |
| 5 | Добавить запись TEST, изменить, перезапустить, удалить | CRUD сохраняется после полного закрытия и запуска; DEMO не записывается в USER DATA. |
| 6 | Выбрать Calm/Compact и перезапустить | Выбор сохранён. Reset appearance не удаляет компьютеры. Проверить Full/Minimal и reduced-motion. |
| 7 | Отключить интернет и открыть Guide, поиск, статью | Все 59 статей доступны offline, ссылки помощи и static copy работают без выполнения команд. |
| 8 | На разрешённом настроенном ПК открыть RDP | Системный mstsc с правильным адресом; выполнить штатную авторизацию. Закрытие NEXUS оставляет сессию открытой. |
| 9 | Открыть SSH | Системный OpenSSH в Terminal либо Console Host; правильный user/DOMAIN/UPN, стандартные host-key/authentication. Закрытие NEXUS не закрывает SSH. |
| 10 | Отправить WoL разрешённому настроенному ПК | Контролируемый burst, при поддержке BIOS/NIC/сети устройство включается. Toast подтверждает отправку, не физическую загрузку. |
| 11 | Wake → RDP и Wake → SSH; отдельно Отмена и timeout | Одна отправка, ожидание выбранной службы до 90 секунд, один клиент. Отмена не запускает клиент позже, повторный клик не создаёт дубликат. |
| 12 | Закрыть и снова открыть; minimize/restore/resize | Компьютеры и appearance сохранены, Guide доступен, нет белого экрана, clipping или ускорения фона. |
| 13 | Закрыть NEXUS и повторно установить тот же Setup | Данные сохранены, один ярлык и один uninstall entry; путь установки тот же. Будущее обновление с другим номером версии этим НЕ доказано. |
| 14 | Проверить «Приложения и возможности», затем удалить | Запись NEXUS CONTROL, версия RC и publisher network.onion там, где поддерживает NSIS; uninstall без администраторских прав, файлы приложения/ярлык/entry удалены. |
| 15 | Проверить `%APPDATA%\NEXUS CONTROL` | `computers.json` и Chromium storage appearance остались. Обычный uninstall не удаляет userData. Не использовать `--delete-app-data`: штатный NSIS допускает этот явный destructive flag. |
| 16 | Повторно установить и запустить из «Пуск» | Компьютеры и appearance восстановились; повторить Guide и выбранный клиент. |

RDP/SSH/WoL без настроенного target помечать NOT TESTED, а не PASS. Для отчёта: Windows build, Node/npm, DPI, build checksum, результат каждого шага. Не присылать пароли/ключи/terminal contents. Системные настройки target вручную; NEXUS их не включает.

Без installer можно сейчас распаковать весь Windows-x64 ZIP в новую папку и запустить `NEXUS CONTROL.exe`: Node.js и npm для готового приложения не нужны. Пункты установки/shortcut/uninstall/reinstall остаются NOT TESTED. Это обычный unpacked package, не NSIS portable target; данные по-прежнему в APPDATA.

Диагностика Stage12 из исходников: [WINDOWS-SMOKE.md](./WINDOWS-SMOKE.md). После появления подписанного релиза нужно отдельно проверить code signing и reputation; никакого обхода SmartScreen здесь нет.
