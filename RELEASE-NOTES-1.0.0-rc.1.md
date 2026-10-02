# NEXUS CONTROL 1.0.0-rc.1 — Release notes

Windows x64 release candidate, by network.onion. Принятая функциональность и визуальная система Stage12 сохранены. Добавлены packaging electron-builder/NSIS, единая Windows identity и полноценная фирменная ICO.

Поставка содержит готовое unpacked Windows-приложение и исходники. **Готовый installer не собран**: среда Linux запрещает Unix sockets, необходимые Wine для извлечения NSIS uninstaller. Промежуточный EXE исключён из поставки. Installer можно собрать на Windows командой `npm.cmd run dist:win` из исходников.

Публичная версия 1.0.0-rc.1, Windows numeric metadata 1.0.0.1, appId network.onion.nexus-control. Будущий installer: per-user, «Пуск», без Desktop shortcut, без администратора для обычной установки, обычное удаление сохраняет userData. Пока это конфигурация и статическая проверка; native installer behavior требует checklist.

Данные Stage12: `%APPDATA%\NEXUS CONTROL`; база и appearance не мигрируют и не удаляются. Готовый Windows ZIP распаковать целиком в новую папку, запускать NEXUS CONTROL.exe; Node/npm не нужны. Не запускать одновременно с dev или другим экземпляром на том же профиле.

Сборка unsigned. Windows может показать SmartScreen/reputation warning. Updater, telemetry, agent, service, автозапуск и новые IPC не добавлены.

Верификация: typecheck/build PASS; 199 unit/packaging tests; 128 Electron groups Stage12→3; 120 appearance matrix views. PE/resources/ASAR проверены без запуска Windows executable. Windows login, физический WoL, installer GUI/UAC/shortcuts/uninstall/reinstall/DPI не проверялись.

[Полный отчёт](./STAGE13.md) · [Windows checklist](./WINDOWS-SMOKE-STAGE13.md)
