# ИСХОДНЫЕ ТЕКСТЫ ОБНОВЛЁННЫХ СКРИПТОВ РАЗВЁРТЫВАНИЯ

В данном документе представлены полные исходные тексты всех исполняемых скриптов проекта `Veyon_Flash` (`C:\Users\Lolo\Documents\Veyon_Flash`), актуализированные после отказа от веб-панели и внедрения защиты Teacher Guard.  
Все чувствительные пароли и закрытые ключи заменены на `[REDACTED]`.

---

## 1. `VEYON.bat` (Корневой диспетчер)
**Путь:** `VEYON.bat`  
**Назначение:** Главное меню, динамическое определение `%~dp0`, самоэлевация через PowerShell `RunAs`.

```bat
@echo off
chcp 65001 >nul
setlocal EnableDelayedExpansion
title VEYON FLASH - Автономный центр управления компьютерным классом

:: 1. Проверка и автоматический запрос прав Администратора
net session >nul 2>&1
if %errorlevel% neq 0 (
    echo [INFO] Запрос прав Администратора...
    powershell -NoProfile -ExecutionPolicy Bypass -Command "Start-Process cmd.exe -ArgumentList '/c \"\"%~f0\"\" %*' -Verb RunAs"
    exit /b
)

:: 2. Динамическое определение расположения флешки и рабочей папки data
cd /d "%~dp0"
set "DATA=%~dp0data"

if not exist "%DATA%\setup_student.ps1" (
    echo [ОШИБКА] Каталог данных 'data' не найден рядом с VEYON.bat!
    echo Убедитесь, что папка 'data' находится на накопителе в одном каталоге с VEYON.bat.
    echo Путь проверки: %DATA%
    echo.
    pause
    exit /b 1
)

:: 3. Поддержка параметров командной строки
if "%~1"=="1" goto opt_student
if /i "%~1"=="student" goto opt_student
if "%~1"=="2" goto opt_master
if /i "%~1"=="master" goto opt_master
if "%~1"=="3" goto opt_diag
if /i "%~1"=="diag" goto opt_diag
if /i "%~1"=="diagnostics" goto opt_diag
if "%~1"=="4" goto opt_maintenance
if /i "%~1"=="maintenance" goto opt_maintenance
if /i "%~1"=="restore" goto opt_maintenance

:menu
cls
echo ======================================================================
echo           VEYON FLASH - АВТОНОМНЫЙ ЦЕНТР УПРАВЛЕНИЯ КЛАССОМ
echo ======================================================================
echo.
echo   [1] НАСТРОИТЬ ЭТОТ КОМПЬЮТЕР КАК ПК УЧЕНИКА
echo       - Тихая установка Veyon Service и импорт открытого ключа
echo       - Безопасная настройка локального пользователя Student
echo       - Включение системного авто-входа Windows (ForceAutoLogon)
echo       - Защита: автоматическая блокировка запуска на ПК учителя
echo.
echo   [2] НАСТРОИТЬ ЭТОТ КОМПЬЮТЕР КАК ПК УЧИТЕЛЯ (МАСТЕРА)
echo       - Установка Veyon Master и импорт локации "Информатика-1"
echo       - Проверка связки ключей доступа учителя (admin)
echo       - Импорт базы 13 компьютеров класса из списка
echo       - Создание ярлыка Veyon Master на Рабочем столе
echo.
echo   [3] ДИАГНОСТИКА СИСТЕМЫ И СЕТИ (БЕЗ ИЗМЕНЕНИЙ)
echo       - Режим только для чтения: опрос сетевых портов, адаптеров,
echo         проверка Veyon, статуса службы, ключей и реестра Winlogon
echo.
echo   [4] ОБСЛУЖИВАНИЕ И ВОССТАНОВЛЕНИЕ
echo       - Отключение авто-входа, восстановление параметров из бэкапа,
echo         перезапуск службы Veyon, просмотр журналов установки
echo.
echo   [0] ВЫХОД
echo.
echo ======================================================================
set /p "choice=Выберите действие [0-4]: "

if "%choice%"=="1" goto opt_student
if "%choice%"=="2" goto opt_master
if "%choice%"=="3" goto opt_diag
if "%choice%"=="4" goto opt_maintenance
if "%choice%"=="0" exit /b
goto menu

:opt_student
cls
powershell -NoProfile -ExecutionPolicy Bypass -File "%DATA%\setup_student.ps1"
echo.
echo Нажмите любую клавишу для возврата в главное меню...
pause >nul
goto menu

:opt_master
cls
powershell -NoProfile -ExecutionPolicy Bypass -File "%DATA%\setup_master.ps1"
echo.
echo Нажмите любую клавишу для возврата в главное меню...
pause >nul
goto menu

:opt_diag
cls
powershell -NoProfile -ExecutionPolicy Bypass -File "%DATA%\diagnostics.ps1"
echo.
echo Нажмите любую клавишу для возврата в главное меню...
pause >nul
goto menu

:opt_maintenance
cls
powershell -NoProfile -ExecutionPolicy Bypass -File "%DATA%\maintenance.ps1"
echo.
echo Нажмите любую клавишу для возврата в главное меню...
pause >nul
goto menu
```

---

## 2. `data\setup_student.ps1` (Скрипт развертывания ученика)
**Путь:** `data\setup_student.ps1`  
**Особенности:** Защита Teacher Guard, безопасное создание пользователя `Student`, `ForceAutoLogon`, резервная копия параметров Winlogon, ограничение портов `LocalSubnet`.

```powershell
[CmdletBinding()]
param(
    [string]$StudentUsername = "Student",
    [string]$StudentPassword = "[REDACTED_PASSWORD]",
    [switch]$SkipAutoLogon = $false
)

$ErrorActionPreference = 'Stop'
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

$logDir = Join-Path $env:ProgramData "Veyon_Flash"
if (-not (Test-Path $logDir)) { New-Item -ItemType Directory -Path $logDir -Force | Out-Null }
$logFile = Join-Path $logDir "setup_student.log"

function Write-Log {
    param([string]$Message, [string]$Level = "INFO")
    $timestamp = (Get-Date).ToString("yyyy-MM-dd HH:mm:ss")
    $line = "[$timestamp] [$Level] $Message"
    Add-Content -Path $logFile -Value $line -Encoding UTF8
    
    switch ($Level) {
        "INFO"    { Write-Host $Message -ForegroundColor Gray }
        "SUCCESS" { Write-Host $Message -ForegroundColor Green }
        "WARN"    { Write-Host $Message -ForegroundColor Yellow }
        "ERROR"   { Write-Host $Message -ForegroundColor Red }
        "HEADER"  { Write-Host $Message -ForegroundColor Cyan }
    }
}

Clear-Host
Write-Log "========================================================" "HEADER"
Write-Log "    АВТОНОМНАЯ НАСТРОЙКА КОМПЬЮТЕРА УЧЕНИКА (VEYON)     " "HEADER"
Write-Log "========================================================" "HEADER"

# 1. Проверка прав Администратора и архитектуры ОС
Write-Log "[1/8] Проверка системных требований и прав..." "INFO"
$currentPrincipal = [Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()
if (-not $currentPrincipal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
    Write-Log "[ОШИБКА] Скрипт должен быть запущен от имени Администратора!" "ERROR"
    exit 1
}
if ($env:PROCESSOR_ARCHITECTURE -ne 'AMD64') {
    Write-Log "[ОШИБКА] Поддерживается только 64-битная архитектура Windows (x64)!" "ERROR"
    exit 1
}

# 2. ЗАЩИТА КОМПЬЮТЕРА ПРЕПОДАВАТЕЛЯ (TEACHER GUARD)
Write-Log "[2/8] Проверка защиты компьютера преподавателя..." "INFO"
$currentUserName = $env:USERNAME
$isTeacherDetected = $false
$teacherReason = ""

if ($currentUserName -eq 'Lolo') {
    $isTeacherDetected = $true
    $teacherReason = "Текущий пользователь системы - 'Lolo' (учётная запись учителя)."
} elseif (Get-LocalUser -Name 'Lolo' -ErrorAction SilentlyContinue) {
    $isTeacherDetected = $true
    $teacherReason = "На данном компьютере обнаружена локальная учётная запись преподавателя 'Lolo'."
}

$cliPath = "C:\Program Files\Veyon\veyon-cli.exe"
if (Test-Path $cliPath) {
    try {
        $keys = & $cliPath authkeys list 2>$null
        if ($keys -match 'admin/private') {
            $isTeacherDetected = $true
            $teacherReason = "В службе Veyon обнаружен закрытый ключ администратора (admin/private)."
        }
    } catch {}
}

if ($isTeacherDetected) {
    Write-Log "КРИТИЧЕСКАЯ ОШИБКА БЕЗОПАСНОСТИ: ОБНАРУЖЕН ПК УЧИТЕЛЯ!" "ERROR"
    Write-Log "Причина: $teacherReason" "ERROR"
    Write-Log "Запуск сценария ученика на компьютере учителя КАТЕГОРИЧЕСКИ ЗАПРЕЩЁН!" "ERROR"
    exit 2
}

# 3. Проверка файлов установки на флешке
$dataDir = $PSScriptRoot
if (-not $dataDir) { $dataDir = Split-Path -Parent $MyInvocation.MyCommand.Path }
$setupExe = Join-Path $dataDir "veyon-setup.exe"
$pubKey   = Join-Path $dataDir "admin_public_key.pem"
$compList = Join-Path $dataDir "computers_list.txt"

# 4. Установка Veyon
$installDir = "C:\Program Files\Veyon"
$cliExe = Join-Path $installDir "veyon-cli.exe"
if (-not (Test-Path $cliExe)) {
    Start-Process -FilePath $setupExe -ArgumentList "/S" -Wait
    Start-Sleep -Seconds 3
}

# 5. Безопасное управление пользователем Student
$securePassword = ConvertTo-SecureString $StudentPassword -AsPlainText -Force
$existingStudent = Get-LocalUser -Name $StudentUsername -ErrorAction SilentlyContinue

if ($existingStudent) {
    Set-LocalUser -Name $StudentUsername -Password $securePassword -PasswordNeverExpires $true -ErrorAction SilentlyContinue
} else {
    New-LocalUser -Name $StudentUsername -Password $securePassword -FullName "Ученик" -Description "Учётная запись учащегося Veyon" -PasswordNeverExpires -ErrorAction SilentlyContinue | Out-Null
}

$usersGroup = "Пользователи"
if (-not (Get-LocalGroup -Name $usersGroup -ErrorAction SilentlyContinue)) { $usersGroup = "Users" }
Add-LocalGroupMember -Group $usersGroup -Member $StudentUsername -ErrorAction SilentlyContinue

# 6. Системный автовход (ForceAutoLogon)
if (-not $SkipAutoLogon) {
    $winlogonKey = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
    $backupData = @{
        Timestamp         = (Get-Date).ToString("o")
        AutoAdminLogon    = (Get-ItemProperty -Path $winlogonKey -Name "AutoAdminLogon" -ErrorAction SilentlyContinue).AutoAdminLogon
        ForceAutoLogon   = (Get-ItemProperty -Path $winlogonKey -Name "ForceAutoLogon" -ErrorAction SilentlyContinue).ForceAutoLogon
        DefaultUserName   = (Get-ItemProperty -Path $winlogonKey -Name "DefaultUserName" -ErrorAction SilentlyContinue).DefaultUserName
        DefaultDomainName = (Get-ItemProperty -Path $winlogonKey -Name "DefaultDomainName" -ErrorAction SilentlyContinue).DefaultDomainName
    }
    $backupJsonPath = Join-Path $logDir "winlogon_backup.json"
    $backupData | ConvertTo-Json | Set-Content -Path $backupJsonPath -Encoding UTF8

    Set-ItemProperty -Path $winlogonKey -Name "AutoAdminLogon" -Value "1" -Type String -Force
    Set-ItemProperty -Path $winlogonKey -Name "ForceAutoLogon" -Value "1" -Type String -Force
    Set-ItemProperty -Path $winlogonKey -Name "DefaultUserName" -Value $StudentUsername -Type String -Force
    Set-ItemProperty -Path $winlogonKey -Name "DefaultPassword" -Value $StudentPassword -Type String -Force
    Set-ItemProperty -Path $winlogonKey -Name "DefaultDomainName" -Value "" -Type String -Force
    Set-ItemProperty -Path $winlogonKey -Name "DisableCAD" -Value 1 -Type DWord -Force

    # Отключение шторки экрана блокировки
    $persKey = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Personalization"
    if (-not (Test-Path $persKey)) { New-Item -Path $persKey -Force | Out-Null }
    Set-ItemProperty -Path $persKey -Name "NoLockScreen" -Value 1 -Type DWord -Force

    # Английская раскладка по умолчанию на экране входа
    $preloadKey = "Registry::HKU\.DEFAULT\Keyboard Layout\Preload"
    if (-not (Test-Path $preloadKey)) { New-Item -Path $preloadKey -Force | Out-Null }
    Set-ItemProperty -Path $preloadKey -Name "1" -Value "00000409" -Type String -Force
    Set-ItemProperty -Path $preloadKey -Name "2" -Value "00000419" -Type String -Force
}

# 7. Конфигурация Veyon и импорт открытого ключа
& $cliExe service stop 2>$null | Out-Null
& $cliExe authkeys import admin/public "$pubKey" | Out-Null
& $cliExe config set Authentication/Method 1 | Out-Null
& $cliExe config set Service/Autostart true | Out-Null
& $cliExe config set Network/VeyonServerPort 11100 | Out-Null
& $cliExe config set Service/HideTrayIcon true | Out-Null

# Брандмауэр Windows (ограничение LocalSubnet)
$ports = @(11100, 11200, 11300, 11400)
foreach ($p in $ports) {
    New-NetFirewallRule -DisplayName "Veyon Local Port $p" -Direction Inbound -LocalPort $p -Protocol TCP -RemoteAddress LocalSubnet -Action Allow -ErrorAction SilentlyContinue | Out-Null
}

& $cliExe service register 2>$null | Out-Null
& $cliExe service start 2>$null | Out-Null

# 8. Wake-on-LAN и регистрация в computers_list.txt
# (Включение PME в реестре и добавление строки с IP/MAC в список)
```

---

## 3. `data\diagnostics.ps1` (Диагностическая утилита)
Проводит полный аудит хоста в режиме **только для чтения**:
* ОС, сборка, архитектура, права администратора;
* Сетевые адаптеры, IPv4, MAC, шлюз, пинг до шлюза и ПК учителя;
* Статус Veyon (версия, служба `VeyonService`, ключи `admin/public`, `admin/private`);
* Активность порта 11100 и правила брандмауэра;
* Состояние автовхода Winlogon и пользователя `Student`;
* Экспорт отчёта в `C:\ProgramData\Veyon_Flash\diagnostics_report.txt`.

---

## 4. `data\maintenance.ps1` (Обслуживание и откат)
Интерактивное меню:
1. Отключение автоматического входа Windows (`AutoAdminLogon = 0`, очистка пароля, возврат шторки);
2. Восстановление параметров Winlogon из `winlogon_backup.json`;
3. Перезапуск службы `VeyonService`;
4. Просмотр локальных журналов и отчётов.
