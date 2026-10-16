# РЕЕСТР ИЗУЧЕННЫХ ФАЙЛОВ ПРОЕКТА

В данном документе приведён полный перечень всех файлов проекта `Veyon_Flash` и репозитория `veyon`, изученных в рамках аудита.

---

## 1. ФАЙЛЫ ДИСТРИБУТИВА (`C:\Users\Lolo\Documents\Veyon_Flash\`)

| Относительный путь | Размер | Назначение | Статус проверки |
|---|---|---|---|
| `VEYON.bat` | 6.5 КБ | Главный интерактивный командный центр и диспетчер меню на флешке | Проверен, синтаксис корректен |
| `data\setup_student.ps1` | 11.5 КБ | Основной скрипт развертывания ученика (ForceAutoLogon, переименование, ключи) | Проверен (AST OK, UTF-8 BOM) |
| `data\setup_student.bat` | 550 Б | Запускатель `setup_student.ps1` с элевацией UAC | Проверен |
| `data\setup_master.ps1` | 6.4 КБ | Основной скрипт развертывания мастера (импорт ключей, создание класса «Информатика-1») | Проверен (AST OK, UTF-8 BOM) |
| `data\setup_master.bat` | 508 Б | Запускатель `setup_master.ps1` с элевацией UAC | Проверен |
| `data\server.py` | 17.3 КБ | Локальный веб-сервер панели управления и сетевого развертывания (порт 8080) | Проверен (`py_compile` OK) |
| `data\start_panel.bat` | 673 Б | Скрипт запуска веб-панели и автоматического открытия браузера | Проверен |
| `data\computers_list.txt` | 832 Б | База данных 13 компьютеров класса (Имя, IP-адрес, MAC-адрес) | Проверен, валиден |
| `data\admin_public_key.pem` | 800 Б | Открытый RSA-ключ учителя для проверки подписи команд на клиентах | Проверен, валиден |
| `data\admin_private_key.pem`| 3.2 КБ | Закрытый RSA-ключ учителя для подписи управляющих команд мастера | Проверен, валиден |
| `data\veyon-setup.exe` | 16.8 МБ | Официальный инсталлятор Veyon 4.9.6 (x86_64) | Проверен, бинарный релиз |

---

## 2. ИСХОДНЫЙ КОД И КОНФИГУРАЦИЯ VEYON (`C:\Users\Lolo\veyon\`)

| Относительный путь | Назначение | Статус аудита |
|---|---|---|
| `plugins/usersessioncontrol/UserSessionControlPlugin.cpp` | Плагин управления сессиями (реализация автовхода без диалога и фичи смены пароля) | Изменён (коммит `165cf11e2`), проверен |
| `plugins/usersessioncontrol/UserSessionControlPlugin.h` | Заголовочный файл плагина сессий (объявление `m_setPasswordFeature`) | Изменён, проверен |
| `plugins/platform/windows/WindowsUserFunctions.cpp` | Платформенные функции Windows (вызовы `prepareLogon` и `performLogon`) | Изменён, проверен |
| `plugins/usersessioncontrol/CMakeLists.txt` | CMake-конфигурация плагина управления сессиями (`QT_NO_VERSION_TAGGING`) | Изменён, проверен |
| `.github/workflows/build-windows.yml` | GitHub Actions workflow сборки Windows-плагина через MinGW-w64 | Изменён, проверен |
| `.github/workflows/release.yml` | Workflow релизных сборок Linux-пакетов | Проверен (только Linux) |
| `CMakeLists.txt` | Корневой CMake файл проекта Veyon | Изменён, проверен |

---

## 3. СЛУЖЕБНЫЕ ЖУРНАЛЫ И ТРАНСКРИПТЫ
* `C:\Users\Lolo\.gemini\antigravity\brain\c45c7c06-81f2-441c-b2d1-11a5b046f21e\.system_generated\logs\transcript.jsonl` — полный хронологический протокол команд и сессий.
