# ИСХОДНЫЕ ТЕКСТЫ СКРИПТОВ РАЗВЁРТЫВАНИЯ

В данном документе представлены полные тексты всех исполняемых скриптов проекта `Veyon_Flash` (`C:\Users\Lolo\Documents\Veyon_Flash`).  
Все чувствительные данные (пароли, закрытые ключи, сетевые идентификаторы) заменены на токены вида `[REDACTED]`.

---

## 1. `VEYON.bat` (Корневой диспетчер запуска)
**Путь:** `E:\VEYON.bat` (или `C:\Users\Lolo\Documents\Veyon_Flash\VEYON.bat`)  
**Назначение:** Единая точка входа для администратора. Автоматически запрашивает права Администратора через `RunAs` и выводит меню выбора действий.

```bat
@echo off
chcp 65001 >nul
setlocal EnableDelayedExpansion
title VEYON - Центр управления компьютерным классом

:: 1. Проверка и автоматический запрос прав Администратора
net session >nul 2>&1
if %errorlevel% neq 0 (
    echo [INFO] Запрос прав Администратора...
    powershell -NoProfile -ExecutionPolicy Bypass -Command "Start-Process cmd.exe -ArgumentList '/c \"\"%~f0\"\" %*' -Verb RunAs"
    exit /b
)

cd /d "%~dp0"
set "DATA=%~dp0data"

:: Аргументы командной строки для быстрой автоматизации
if "%~1"=="1" goto opt_student
if /i "%~1"=="student" goto opt_student
if "%~1"=="2" goto opt_master
if /i "%~1"=="master" goto opt_master
if "%~1"=="3" goto opt_panel
if /i "%~1"=="panel" goto opt_panel
if "%~1"=="4" goto opt_logon
if /i "%~1"=="logon" goto opt_logon
if "%~1"=="5" goto opt_wol
if /i "%~1"=="wol" goto opt_wol

:menu
cls
echo ======================================================================
echo             VEYON - ЕДИНЫЙ ЦЕНТР УПРАВЛЕНИЯ КЛАССОМ
echo ======================================================================
echo.
echo   [1] НАСТРОИТЬ ЭТОТ КОМПЬЮТЕР КАК ПК УЧЕНИКА
echo       - Переименование пользователя в Student (все файлы сохраняются)
echo       - Установка пароля [REDACTED_STUDENT_PASSWORD] (бессрочно)
echo       - Установка 100%% надежного авто-входа Windows (ForceAutoLogon)
echo       - Английская раскладка по умолчанию на экране блокировки
echo       - Установка службы Veyon и импорт ключа учителя
echo       - Сохранение IP и MAC-адреса ПК для включения по сети (WOL)
echo.
echo   [2] НАСТРОИТЬ ЭТОТ КОМПЬЮТЕР КАК ПК УЧИТЕЛЯ (МАСТЕРА)
echo       - Установка Veyon Master и импорт ключей доступа
echo       - Добавление класса "Информатика-1" со всеми ПК
echo       - Создание ярлыков на Рабочем столе
echo.
echo   [3] ОТКРЫТЬ ВЕБ-ПАНЕЛЬ УПРАВЛЕНИЯ КЛАССОМ (В БРАУЗЕРЕ)
echo       - Мониторинг сети (онлайн/оффлайн)
echo       - Кнопки массового автовхода, включения и перезагрузки
echo.
echo   [4] ЗАЛОГИНИТЬ ВСЕ КОМПЬЮТЕРЫ УЧЕНИКОВ ПРЯМО СЕЙЧАС
echo       - Отправляет сигнал автовхода на все включенные ПК
echo.
echo   [5] ВКЛЮЧИТЬ ВСЕ КОМПЬЮТЕРЫ ПО СЕТИ (WAKE-ON-LAN)
echo       - Отправляет сигнал пробуждения на все MAC-адреса
echo.
echo   [0] ВЫХОД
echo.
echo ======================================================================
set /p "choice=Выберите действие [0-5]: "

if "%choice%"=="1" goto opt_student
if "%choice%"=="2" goto opt_master
if "%choice%"=="3" goto opt_panel
if "%choice%"=="4" goto opt_logon
if "%choice%"=="5" goto opt_wol
if "%choice%"=="0" exit /b
goto menu

:opt_student
cls
powershell -NoProfile -ExecutionPolicy Bypass -File "%DATA%\setup_student.ps1"
echo.
echo Нажмите любую клавишу для возврата в меню...
pause >nul
goto menu

:opt_master
cls
powershell -NoProfile -ExecutionPolicy Bypass -File "%DATA%\setup_master.ps1"
echo.
echo Нажмите любую клавишу для возврата в меню...
pause >nul
goto menu

:opt_panel
cls
call "%DATA%\start_panel.bat"
goto menu

:opt_logon
cls
echo ======================================================================
echo               МАССОВЫЙ АВТОВХОД ПОД STUDENT
echo ======================================================================
echo.
echo Отправка команды перезапуска сессии для авто-входа...
powershell -NoProfile -ExecutionPolicy Bypass -Command "$list = '%DATA%\computers_list.txt'; if (Test-Path $list) { $lines = Get-Content $list; foreach ($l in $lines) { if ($l -match 'IP:\s*([0-9\.]+)') { $ip = $Matches[1]; Write-Host \"Отправка на $ip...\" -ForegroundColor Gray; Invoke-Command -ComputerName $ip -ScriptBlock { logoff.exe } -ErrorAction SilentlyContinue } } Write-Host '[OK] Команда отправлена на все ПК!' -ForegroundColor Green } else { Write-Host '[ОШИБКА] Список компьютеров не найден.' -ForegroundColor Red }"
echo.
echo Нажмите любую клавишу для возврата в меню...
pause >nul
goto menu

:opt_wol
cls
echo ======================================================================
echo             ВКЛЮЧЕНИЕ ВСЕХ ПК ПО СЕТИ (WAKE-ON-LAN)
echo ======================================================================
echo.
echo Отправка сигналов включения (Magic Packet)...
powershell -NoProfile -ExecutionPolicy Bypass -Command "$list = '%DATA%\computers_list.txt'; if (Test-Path $list) { $lines = Get-Content $list; foreach ($l in $lines) { if ($l -match 'MAC:\s*([0-9A-Fa-f\-:]+)') { $mac = $Matches[1].Replace('-','').Replace(':',''); if ($mac.Length -eq 12) { Write-Host \"Бужу MAC: $mac...\" -ForegroundColor Gray; $bytes = [byte[]](@(0xFF)*6) + [byte[]](0..15 | ForEach-Object { for ($i=0; $i -lt 12; $i+=2) { [Convert]::ToByte($mac.Substring($i,2), 16) } }); $client = New-Object System.Net.Sockets.UdpClient; $client.Connect([System.Net.IPAddress]::Broadcast, 9); [void]$client.Send($bytes, $bytes.Length); $client.Close() } } } Write-Host '[OK] Сигнал пробуждения отправлен всем компьютерам!' -ForegroundColor Green } else { Write-Host '[ОШИБКА] Список компьютеров не найден.' -ForegroundColor Red }"
echo.
echo Нажмите любую клавишу для возврата в меню...
pause >nul
goto menu
```

---

## 2. `data\setup_student.ps1` (Скрипт развертывания ученика)
**Путь:** `C:\Users\Lolo\Documents\Veyon_Flash\data\setup_student.ps1`  
**Кодировка:** `UTF-8 with BOM`

```powershell
# =====================================================================
# НАСТРОЙКА КОМПЬЮТЕРА УЧЕНИКА (VEYON + STUDENT + AUTO-LOGON)
# =====================================================================

$ErrorActionPreference = 'SilentlyContinue'
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

Write-Host "========================================================" -ForegroundColor Cyan
Write-Host "         НАСТРОЙКА КОМПЬЮТЕРА УЧЕНИКА" -ForegroundColor Cyan
Write-Host "========================================================" -ForegroundColor Cyan
Write-Host ""

$dataDir = $PSScriptRoot
if (-not $dataDir) { $dataDir = Split-Path -Parent $MyInvocation.MyCommand.Path }

$setupExe = Join-Path $dataDir "veyon-setup.exe"
$pubKey   = Join-Path $dataDir "admin_public_key.pem"
$compList = Join-Path $dataDir "computers_list.txt"
$installDir = "C:\Program Files\Veyon"
$cliExe = Join-Path $installDir "veyon-cli.exe"

# 1. Установка Veyon
Write-Host "[1/6] Проверка и установка Veyon..." -ForegroundColor Yellow
if (-not (Test-Path $cliExe)) {
    if (Test-Path $setupExe) {
        Write-Host "  -> Запуск тихой установки Veyon..." -ForegroundColor Gray
        Start-Process -FilePath $setupExe -ArgumentList "/S" -Wait
        Start-Sleep -Seconds 3
    } else {
        Write-Host "  -> [ВНИМАНИЕ] veyon-setup.exe не найден в $dataDir" -ForegroundColor Red
    }
} else {
    Write-Host "  -> Veyon уже установлен." -ForegroundColor Green
}

# 2. Остановка службы перед настройкой
Write-Host "[2/6] Остановка службы VeyonService..." -ForegroundColor Yellow
Stop-Service -Name "VeyonService" -Force -ErrorAction SilentlyContinue
Stop-Process -Name "veyon-server", "veyon-service", "veyon-worker" -Force -ErrorAction SilentlyContinue

# 3. Настройка пользователя Student
Write-Host "[3/6] Настройка пользователя Student..." -ForegroundColor Yellow
$excludedSIDs = @('-500', '-501', '-503', '-504')
$oldUser = Get-LocalUser | Where-Object {
    $sid = $_.SID.Value
    $name = $_.Name
    $isSystem = $false
    foreach ($s in $excludedSIDs) { if ($sid.EndsWith($s)) { $isSystem = $true } }
    -not $isSystem -and $name -ne 'Student'
} | Select-Object -First 1

if ($oldUser) {
    Write-Host "  -> Найден существующий пользователь: '$($oldUser.Name)'. Переименование в 'Student'..." -ForegroundColor Gray
    if (Get-LocalUser -Name 'Student' -ErrorAction SilentlyContinue) {
        & net.exe user Student /delete 2>$null | Out-Null
    }
    try {
        Rename-LocalUser -Name $oldUser.Name -NewName 'Student' -ErrorAction Stop
        Write-Host "  -> [OK] Пользователь '$($oldUser.Name)' переименован в 'Student' (все файлы сохранены)." -ForegroundColor Green
    } catch {
        (Get-CimInstance Win32_UserAccount | Where-Object { $_.Name -eq $oldUser.Name -and $_.LocalAccount }).Rename('Student') | Out-Null
        Write-Host "  -> [OK] Пользователь переименован через WMI в 'Student'." -ForegroundColor Green
    }
} else {
    if (-not (Get-LocalUser -Name 'Student' -ErrorAction SilentlyContinue)) {
        Write-Host "  -> Создание пользователя Student..." -ForegroundColor Gray
        & net.exe user Student [REDACTED_STUDENT_PASSWORD] /add /y 2>$null | Out-Null
    }
    Write-Host "  -> [OK] Пользователь Student готов." -ForegroundColor Green
}

# Установка пароля и снятие срока действия
& net.exe user Student [REDACTED_STUDENT_PASSWORD] 2>$null | Out-Null
& net.exe user Student /passwordchg:yes /expires:never 2>$null | Out-Null
& net.exe localgroup "Пользователи" Student /add 2>$null | Out-Null
& net.exe localgroup "Users" Student /add 2>$null | Out-Null
Write-Host "  -> [OK] Пароль [REDACTED_STUDENT_PASSWORD] установлен (бессрочно)." -ForegroundColor Green

# 4. Настройка экрана входа и 100% надежного авто-входа Windows
Write-Host "[4/6] Настройка экрана входа и авто-входа Windows..." -ForegroundColor Yellow

# Английская раскладка по умолчанию на экране блокировки
New-Item -Path "Registry::HKU\.DEFAULT\Keyboard Layout\Preload" -Force | Out-Null
Set-ItemProperty -Path "Registry::HKU\.DEFAULT\Keyboard Layout\Preload" -Name "1" -Value "00000409" -Force
Set-ItemProperty -Path "Registry::HKU\.DEFAULT\Keyboard Layout\Preload" -Name "2" -Value "00000419" -Force

# Отключение шторки экрана блокировки (Windows 10/11 Lock Screen curtain)
$persKey = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Personalization"
if (-not (Test-Path $persKey)) { New-Item -Path $persKey -Force | Out-Null }
Set-ItemProperty -Path $persKey -Name "NoLockScreen" -Value 1 -Type DWord -Force

# Отключение требования Ctrl+Alt+Del и скрытия последнего пользователя
$sysKey = "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System"
Set-ItemProperty -Path $sysKey -Name "dontdisplaylastusername" -Value 0 -Type DWord -Force
Set-ItemProperty -Path $sysKey -Name "SoftwareSASGeneration" -Value 3 -Type DWord -Force

# Автоматический вход под Student (AutoAdminLogon + ForceAutoLogon)
$winlogonKey = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
Set-ItemProperty -Path $winlogonKey -Name "AutoAdminLogon" -Value "1" -Type String -Force
Set-ItemProperty -Path $winlogonKey -Name "ForceAutoLogon" -Value "1" -Type String -Force
Set-ItemProperty -Path $winlogonKey -Name "DefaultUserName" -Value "Student" -Type String -Force
Set-ItemProperty -Path $winlogonKey -Name "DefaultPassword" -Value "[REDACTED_STUDENT_PASSWORD]" -Type String -Force
Set-ItemProperty -Path $winlogonKey -Name "DefaultDomainName" -Value "" -Type String -Force
Set-ItemProperty -Path $winlogonKey -Name "DisableCAD" -Value 1 -Type DWord -Force

Write-Host "  -> [OK] Авто-вход включен (Student / [REDACTED_STUDENT_PASSWORD] / Английская раскладка)." -ForegroundColor Green

# 5. Импорт ключа учителя и настройка Veyon
Write-Host "[5/6] Настройка Veyon и импорт ключа учителя..." -ForegroundColor Yellow
if (Test-Path $cliExe) {
    if (Test-Path $pubKey) {
        & $cliExe authkeys import admin/public $pubKey | Out-Null
        Write-Host "  -> [OK] Публичный ключ учителя успешно импортирован." -ForegroundColor Green
    } else {
        Write-Host "  -> [ПРЕДУПРЕЖДЕНИЕ] Файл ключа admin_public_key.pem не найден." -ForegroundColor Red
    }

    # Настройки аутентификации и сети
    & $cliExe config set Service/Autostart true | Out-Null
    & $cliExe config set Authentication/Method 1 | Out-Null
    & $cliExe config set Network/PrimaryServicePort 11100 | Out-Null

    # Брандмауэр Windows (открытие портов для Veyon)
    New-NetFirewallRule -DisplayName "Veyon Server Port 11100" -Direction Inbound -LocalPort 11100 -Protocol TCP -Action Allow -ErrorAction SilentlyContinue | Out-Null
    New-NetFirewallRule -DisplayName "Veyon Manager Port 11200" -Direction Inbound -LocalPort 11200 -Protocol TCP -Action Allow -ErrorAction SilentlyContinue | Out-Null
    New-NetFirewallRule -DisplayName "Veyon Feature Port 11300" -Direction Inbound -LocalPort 11300 -Protocol TCP -Action Allow -ErrorAction SilentlyContinue | Out-Null
    New-NetFirewallRule -DisplayName "Veyon Demo Port 11400" -Direction Inbound -LocalPort 11400 -Protocol TCP -Action Allow -ErrorAction SilentlyContinue | Out-Null
    New-NetFirewallRule -DisplayName "Veyon Service App" -Direction Inbound -Program "$installDir\veyon-service.exe" -Action Allow -ErrorAction SilentlyContinue | Out-Null
    New-NetFirewallRule -DisplayName "Veyon Server App" -Direction Inbound -Program "$installDir\veyon-server.exe" -Action Allow -ErrorAction SilentlyContinue | Out-Null

    # Запуск службы
    Start-Service -Name "VeyonService" -ErrorAction SilentlyContinue
    Write-Host "  -> [OK] Служба Veyon запущена." -ForegroundColor Green
}

# 6. Включение Wake-on-LAN и сохранение данных ПК
Write-Host "[6/6] Настройка сетевого адаптера (Wake-on-LAN)..." -ForegroundColor Yellow
$mac = ""
$ip = ""

$adapters = Get-CimInstance Win32_NetworkAdapterConfiguration | Where-Object { $_.IPEnabled -eq $true }
foreach ($adapter in $adapters) {
    $currentIP = ($adapter.IPAddress | Where-Object { $_ -match '^\d+\.\d+\.\d+\.\d+$' -and $_ -notmatch '^127\.' -and $_ -notmatch '^169\.254\.' } | Select-Object -First 1)
    if ($currentIP) {
        $ip = $currentIP
        $mac = $adapter.MACAddress.Replace(':', '-')
        break
    }
}

# Включение Wake-on-LAN в реестре сетевых адаптеров
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4d36e972-e325-11ce-bfc1-08002be10318}\*" | Where-Object { $_.DriverDesc } | ForEach-Object {
    Set-ItemProperty -Path $_.PSPath -Name "EnablePME" -Value "1" -Force -ErrorAction SilentlyContinue
    Set-ItemProperty -Path $_.PSPath -Name "WakeOn" -Value "6" -Force -ErrorAction SilentlyContinue
    Set-ItemProperty -Path $_.PSPath -Name "WakeOnMagicPacket" -Value "1" -Force -ErrorAction SilentlyContinue
}

$computerName = $env:COMPUTERNAME
Write-Host "  -> Имя ПК:   $computerName" -ForegroundColor Cyan
Write-Host "  -> IP адрес:  $ip" -ForegroundColor Cyan
Write-Host "  -> MAC адрес: $mac" -ForegroundColor Cyan

# Сохранение в список компьютеров на флешке
if ($mac -and (Test-Path (Split-Path -Parent $compList))) {
    $line = "{0,-15} | IP: {1,-15} | MAC: {2}" -f $computerName, $ip, $mac
    $existing = ""
    if (Test-Path $compList) { $existing = Get-Content $compList -Raw }
    if ($existing -notmatch [regex]::Escape($mac)) {
        Add-Content -Path $compList -Value $line -Encoding UTF8
        Write-Host "  -> [OK] Данные компьютера добавлены в computers_list.txt на флешке." -ForegroundColor Green
    } else {
        Write-Host "  -> [OK] Компьютер уже зарегистрирован в списке." -ForegroundColor Green
    }
}

Write-Host ""
Write-Host "========================================================" -ForegroundColor Green
Write-Host "         НАСТРОЙКА УСПЕШНО ЗАВЕРШЕНА!" -ForegroundColor Green
Write-Host "========================================================" -ForegroundColor Green
Write-Host "  Логин:    Student"
Write-Host "  Пароль:   [REDACTED_STUDENT_PASSWORD]"
Write-Host "  Авто-вход: ВКЛЮЧЕН (входит сам при включении и выходе)"
Write-Host "========================================================" -ForegroundColor Green
```

---

## 3. `data\setup_master.ps1` (Скрипт развертывания ПК учителя)
**Путь:** `C:\Users\Lolo\Documents\Veyon_Flash\data\setup_master.ps1`  
**Кодировка:** `UTF-8 with BOM`

```powershell
# =====================================================================
# НАСТРОЙКА ПК УЧИТЕЛЯ (VEYON MASTER + КЛЮЧИ + КЛАСС)
# =====================================================================

$ErrorActionPreference = 'SilentlyContinue'
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

Write-Host "========================================================" -ForegroundColor Cyan
Write-Host "         НАСТРОЙКА КОМПЬЮТЕРА УЧИТЕЛЯ (МАСТЕРА)" -ForegroundColor Cyan
Write-Host "========================================================" -ForegroundColor Cyan
Write-Host ""

$dataDir = $PSScriptRoot
if (-not $dataDir) { $dataDir = Split-Path -Parent $MyInvocation.MyCommand.Path }

$setupExe  = Join-Path $dataDir "veyon-setup.exe"
$privKey   = Join-Path $dataDir "admin_private_key.pem"
$pubKey    = Join-Path $dataDir "admin_public_key.pem"
$compList  = Join-Path $dataDir "computers_list.txt"
$installDir = "C:\Program Files\Veyon"
$cliExe    = Join-Path $installDir "veyon-cli.exe"
$masterExe = Join-Path $installDir "veyon-master.exe"

# 1. Установка Veyon
Write-Host "[1/5] Проверка и установка Veyon..." -ForegroundColor Yellow
if (-not (Test-Path $cliExe)) {
    if (Test-Path $setupExe) {
        Write-Host "  -> Запуск тихой установки Veyon..." -ForegroundColor Gray
        Start-Process -FilePath $setupExe -ArgumentList "/S" -Wait
        Start-Sleep -Seconds 3
    } else {
        Write-Host "  -> [ОШИБКА] veyon-setup.exe не найден в $dataDir" -ForegroundColor Red
        exit 1
    }
}
Write-Host "  -> [OK] Veyon установлен." -ForegroundColor Green

# 2. Импорт ключей доступа
Write-Host "[2/5] Импорт ключей доступа учителя..." -ForegroundColor Yellow
if (Test-Path $privKey) {
    & $cliExe authkeys import admin/private $privKey | Out-Null
    Write-Host "  -> [OK] Приватный ключ admin импортирован." -ForegroundColor Green
} else {
    Write-Host "  -> [ВНИМАНИЕ] admin_private_key.pem не найден, генерирую новую пару..." -ForegroundColor Yellow
    & $cliExe authkeys create admin | Out-Null
    & $cliExe authkeys export admin/private $privKey | Out-Null
    & $cliExe authkeys export admin/public $pubKey | Out-Null
    Write-Host "  -> [OK] Новая пара ключей создана и сохранена в data." -ForegroundColor Green
}

if (Test-Path $pubKey) {
    & $cliExe authkeys import admin/public $pubKey | Out-Null
    Write-Host "  -> [OK] Публичный ключ admin импортирован." -ForegroundColor Green
}

# 3. Базовая конфигурация Veyon Master
Write-Host "[3/5] Настройка Veyon Master..." -ForegroundColor Yellow
& $cliExe config set Authentication/Method 1 | Out-Null
& $cliExe config set Network/PrimaryServicePort 11100 | Out-Null
& $cliExe config set NetworkObjectDirectory/Plugin "BuiltinDirectory" | Out-Null
& $cliExe config set Master/HideOwnComputer true | Out-Null
& $cliExe config set Master/AutoAdjustComputerPositions true | Out-Null
Write-Host "  -> [OK] Аутентификация по ключам и встроенный каталог настроены." -ForegroundColor Green

# 4. Импорт компьютеров класса в комнату "Информатика-1"
Write-Host "[4/5] Добавление компьютеров класса в Veyon..." -ForegroundColor Yellow
$locationName = "Информатика-1"
& $cliExe networkobjects add location $locationName | Out-Null

if (Test-Path $compList) {
    $lines = Get-Content $compList -Encoding UTF8
    $count = 0
    foreach ($line in $lines) {
        $parts = $line.Split('|')
        if ($parts.Count -ge 3) {
            $cName = $parts[0].Trim()
            $cIP   = $parts[1].Replace('IP:', '').Trim()
            $cMAC  = $parts[2].Replace('MAC:', '').Trim()
            if ($cName -and $cIP) {
                & $cliExe networkobjects add computer $cName $cIP $cMAC $locationName | Out-Null
                $count++
            }
        }
    }
    Write-Host "  -> [OK] Добавлено компьютеров в класс '$locationName': $count" -ForegroundColor Green
} else {
    Write-Host "  -> [ИНФО] Список компьютеров computers_list.txt пуст или не найден." -ForegroundColor Gray
}

# 5. Создание ярлыков на Рабочем столе
Write-Host "[5/5] Создание ярлыков на Рабочем столе..." -ForegroundColor Yellow
$wsh = New-Object -ComObject WScript.Shell
$desktop = [System.Environment]::GetFolderPath('Desktop')

# Ярлык Veyon Master
$masterShortcut = Join-Path $desktop "Veyon Master.lnk"
$s = $wsh.CreateShortcut($masterShortcut)
$s.TargetPath = $masterExe
$s.WorkingDirectory = $installDir
$s.Description = "Управление компьютерами класса Veyon"
$s.Save()

# Ярлык Панели управления (Веб-сервер)
$panelBat = Join-Path $dataDir "start_panel.bat"
$panelShortcut = Join-Path $desktop "Панель управления классом.lnk"
$s2 = $wsh.CreateShortcut($panelShortcut)
$s2.TargetPath = $panelBat
$s2.WorkingDirectory = $dataDir
$s2.Description = "Веб-панель: автовход, включение и управление всеми ПК"
$s2.Save()

Write-Host "  -> [OK] Ярлыки 'Veyon Master' и 'Панель управления классом' созданы на Рабочем столе." -ForegroundColor Green

Write-Host ""
Write-Host "========================================================" -ForegroundColor Green
Write-Host "      НАСТРОЙКА ПК УЧИТЕЛЯ УСПЕШНО ЗАВЕРШЕНА!" -ForegroundColor Green
Write-Host "========================================================" -ForegroundColor Green
Write-Host "  - Класс 'Информатика-1' настроен"
Write-Host "  - Ключи доступа импортированы"
Write-Host "  - Запустите Veyon Master с Рабочего стола"
```

---

## 4. `data\server.py` (HTTP веб-сервер панели развёртывания)
**Путь:** `C:\Users\Lolo\Documents\Veyon_Flash\data\server.py`  
**Порт по умолчанию:** `8080`

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Veyon Control Panel & Deployment Server
Панель управления компьютерным классом
"""

import os
import sys
import json
import socket
import struct
import subprocess
import threading
import time
from http.server import HTTPServer, BaseHTTPRequestHandler
from urllib.parse import urlparse, parse_qs

PORT = 8080
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
COMPUTERS_FILE = os.path.join(BASE_DIR, "computers_list.txt")

def get_local_ip():
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        s.connect(("8.8.8.8", 80))
        ip = s.getsockname()[0]
        s.close()
        return ip
    except Exception:
        return "127.0.0.1"

LOCAL_IP = get_local_ip()

def load_computers():
    computers = []
    if os.path.exists(COMPUTERS_FILE):
        try:
            with open(COMPUTERS_FILE, "r", encoding="utf-8") as f:
                for line in f:
                    line = line.strip()
                    if not line or line.startswith("#"):
                        continue
                    parts = [p.strip() for p in line.split("|")]
                    if len(parts) >= 3:
                        name = parts[0]
                        ip = parts[1].replace("IP:", "").strip()
                        mac = parts[2].replace("MAC:", "").strip()
                        computers.append({"name": name, "ip": ip, "mac": mac})
        except Exception as e:
            print(f"Error reading computers_list.txt: {e}")
    return computers

def save_computers(computers):
    try:
        with open(COMPUTERS_FILE, "w", encoding="utf-8") as f:
            for c in computers:
                f.write(f"{c['name']:<15} | IP: {c['ip']:<15} | MAC: {c['mac']}\n")
        return True
    except Exception as e:
        print(f"Error saving computers_list.txt: {e}")
        return False

def ping_host(ip):
    try:
        cmd = ["ping", "-n", "1", "-w", "400", ip] if sys.platform == "win32" else ["ping", "-c", "1", "-W", "1", ip]
        res = subprocess.run(cmd, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        return res.returncode == 0
    except Exception:
        return False

def send_wol(mac_address):
    try:
        clean_mac = mac_address.replace(":", "").replace("-", "").replace(".", "").upper()
        if len(clean_mac) != 12:
            return False
        hw_addr = bytes.fromhex(clean_mac)
        magic_packet = b"\xff" * 6 + hw_addr * 16
        with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
            s.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
            s.sendto(magic_packet, ("255.255.255.255", 9))
            s.sendto(magic_packet, ("255.255.255.255", 7))
        return True
    except Exception as e:
        print(f"Error sending WOL to {mac_address}: {e}")
        return False

def run_remote_command(ip, cmd):
    try:
        ps_cmd = f"Invoke-Command -ComputerName {ip} -ScriptBlock {{ {cmd} }} -ErrorAction SilentlyContinue"
        subprocess.Popen(["powershell", "-NoProfile", "-Command", ps_cmd], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
    except Exception:
        pass

# HTML интерфейс опущен для краткости, структура и функционал сохранены

class RequestHandler(BaseHTTPRequestHandler):
    def log_message(self, format, *args):
        pass

    def do_GET(self):
        parsed = urlparse(self.path)
        path = parsed.path

        if path == "/" or path == "/index.html":
            # Выдача HTML панели
            self.send_response(200)
            self.send_header("Content-Type", "text/html; charset=utf-8")
            self.end_headers()
            return

        if path == "/api/computers":
            computers = load_computers()
            def check_online(c):
                c["online"] = ping_host(c["ip"])
            threads = [threading.Thread(target=check_online, args=(c,)) for c in computers]
            for t in threads: t.start()
            for t in threads: t.join(timeout=1.0)
            
            self.send_response(200)
            self.send_header("Content-Type", "application/json; charset=utf-8")
            self.end_headers()
            self.wfile.write(json.dumps(computers, ensure_ascii=False).encode("utf-8"))
            return

        if path == "/setup":
            setup_script_path = os.path.join(BASE_DIR, "setup_student.ps1")
            if os.path.exists(setup_script_path):
                with open(setup_script_path, "r", encoding="utf-8") as f:
                    content = f.read()
                network_prefix = f"$server = 'http://{LOCAL_IP}:8080'\n"
                content = network_prefix + content
                self.send_response(200)
                self.send_header("Content-Type", "text/plain; charset=utf-8")
                self.end_headers()
                self.wfile.write(content.encode("utf-8"))
                return

        if path.startswith("/files/"):
            filename = os.path.basename(path)
            filepath = os.path.join(BASE_DIR, filename)
            if os.path.exists(filepath):
                self.send_response(200)
                self.send_header("Content-Type", "application/octet-stream")
                self.send_header("Content-Length", str(os.path.getsize(filepath)))
                self.end_headers()
                with open(filepath, "rb") as f:
                    while chunk := f.read(65536):
                        self.wfile.write(chunk)
                return

        self.send_response(404)
        self.end_headers()

    def do_POST(self):
        parsed = urlparse(self.path)
        path = parsed.path
        length = int(self.headers.get('Content-Length', 0))
        body = self.rfile.read(length) if length > 0 else b"{}"
        data = json.loads(body.decode("utf-8")) if body else {}

        if path == "/api/wol":
            mac = data.get("mac", "")
            if mac: send_wol(mac)
            self.send_response(200); self.end_headers(); self.wfile.write(b'{"status":"ok"}'); return

        if path == "/api/wol_all":
            for c in load_computers():
                if c.get("mac"): send_wol(c["mac"])
            self.send_response(200); self.end_headers(); self.wfile.write(b'{"status":"ok"}'); return

        if path == "/api/logon_all":
            for c in load_computers():
                ip = c.get("ip")
                if ip:
                    run_remote_command(ip, "logoff.exe")
            self.send_response(200); self.end_headers(); self.wfile.write(b'{"status":"ok"}'); return

        if path == "/api/reboot_all":
            for c in load_computers():
                ip = c.get("ip")
                if ip: run_remote_command(ip, "Restart-Computer -Force")
            self.send_response(200); self.end_headers(); self.wfile.write(b'{"status":"ok"}'); return

        if path == "/api/shutdown_all":
            for c in load_computers():
                ip = c.get("ip")
                if ip: run_remote_command(ip, "Stop-Computer -Force")
            self.send_response(200); self.end_headers(); self.wfile.write(b'{"status":"ok"}'); return

        self.send_response(404)
        self.end_headers()

def main():
    print(f"Панель запущена: http://localhost:{PORT}")
    server = HTTPServer(("0.0.0.0", PORT), RequestHandler)
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        pass

if __name__ == "__main__":
    main()
```

---

## 5. Вспомогательные bat-файлы
### `data\setup_student.bat`
```bat
@echo off
chcp 65001 >nul
setlocal EnableDelayedExpansion
title Настройка ПК Ученика

net session >nul 2>&1
if %errorlevel% neq 0 (
    echo [INFO] Запрос прав Администратора...
    powershell -NoProfile -ExecutionPolicy Bypass -Command "Start-Process cmd.exe -ArgumentList '/c \"\"%~f0\"\"' -Verb RunAs"
    exit /b
)

cd /d "%~dp0"
cls
powershell -NoProfile -ExecutionPolicy Bypass -File "%~dp0setup_student.ps1"
echo.
pause
```

### `data\setup_master.bat`
```bat
@echo off
chcp 65001 >nul
setlocal EnableDelayedExpansion
title Настройка ПК Учителя (Мастера)

net session >nul 2>&1
if %errorlevel% neq 0 (
    echo [INFO] Запрос прав Администратора...
    powershell -NoProfile -ExecutionPolicy Bypass -Command "Start-Process cmd.exe -ArgumentList '/c \"\"%~f0\"\"' -Verb RunAs"
    exit /b
)

cd /d "%~dp0"
cls
powershell -NoProfile -ExecutionPolicy Bypass -File "%~dp0setup_master.ps1"
echo.
pause
```

### `data\start_panel.bat`
```bat
@echo off
chcp 65001 >nul
title Панель управления классом Veyon
cd /d "%~dp0"

echo Запуск панели управления классом...
start http://localhost:8080

where python >nul 2>&1
if %errorlevel% equ 0 (
    python server.py
) else (
    if exist "%LOCALAPPDATA%\Programs\Python\Python314\python.exe" (
        "%LOCALAPPDATA%\Programs\Python\Python314\python.exe" server.py
    ) else (
        echo [ОШИБКА] Python не найден на этом компьютере.
        pause
    )
)
```

---

## 6. Пример `data\computers_list.txt` (Маскированные данные)
```
PC-1            | IP: 192.168.1.10    | MAC: 00-E0-1D-[REDACTED]
PC-2            | IP: 192.168.1.11    | MAC: 00-E0-1E-[REDACTED]
PC-3            | IP: 192.168.1.12    | MAC: 00-E0-1B-[REDACTED]
PC-4            | IP: 192.168.1.13    | MAC: 00-E0-1F-[REDACTED]
PC-5            | IP: 192.168.1.14    | MAC: 00-E0-1D-[REDACTED]
PC-6            | IP: 192.168.1.15    | MAC: 00-E0-1B-[REDACTED]
PC-7            | IP: 192.168.1.16    | MAC: 00-E0-1D-[REDACTED]
PC-8            | IP: 192.168.1.17    | MAC: 00-E0-1E-[REDACTED]
PC-9            | IP: 192.168.1.18    | MAC: 00-E0-1D-[REDACTED]
PC-10           | IP: 192.168.1.19    | MAC: 00-E0-1F-[REDACTED]
PC-11           | IP: 192.168.1.20    | MAC: 00-E0-1D-[REDACTED]
PC-12           | IP: 192.168.1.21    | MAC: 00-E0-1D-[REDACTED]
main-pc         | IP: 192.168.1.2     | MAC: 00-E0-1B-[REDACTED]
```
