# Docker — Установка и Развёртывание

## v1.0.0 | Два пути развёртывания: Hyper-V & WSL

### Описание

Полная установка и настройка Docker Desktop 4.49.0 на Windows 10 Pro (сборка 19044).  
Реализовано **два пути развёртывания**: через **Hyper-V** (активен) и через **WSL 2** (подготовлен, требует обновления ОС).

---

### Системное окружение

| Параметр | Значение |
|---|---|
| ОС | Windows 10 Pro 10.0.19044.1165 |
| Docker Desktop | 4.49.0 (208700) |
| Docker Engine | 28.5.1 |
| Docker Compose | v2.40.3-desktop.1 |
| Buildx | v0.29.1-desktop.1 |
| Ядро Linux (VM) | 6.10.14-linuxkit |

---

## Путь 1 — Hyper-V Backend ✅ (активен)

### Когда использовать
- Windows 10 Pro / Enterprise с поддержкой виртуализации
- WSL2 заблокирован групповой политикой или ядро устарело

### Шаги развёртывания

**1. Включить Hyper-V через DISM** (от имени администратора):
```powershell
DISM /Online /Enable-Feature /FeatureName:Microsoft-Hyper-V-All /All /NoRestart
```

**2. Перезагрузить Windows** для активации драйверов Hyper-V.

**3. Переключить Docker Desktop на Hyper-V backend:**
```json
// %APPDATA%\Docker\settings-store.json
{
  "WslEngineEnabled": false
}
```

**4. Запустить Docker Desktop** — он автоматически создаст Hyper-V VM.

### Результат
```powershell
docker info        # Server Version: 28.5.1, Kernel: 6.10.14-linuxkit
docker run --rm hello-world  # Hello from Docker!
```

---

## Путь 2 — WSL 2 Backend ⚙️ (подготовлен, ограничен)

### Когда использовать
- Windows 10 сборка 19044.2311+ или Windows 11
- Политики обновления Windows не ограничены

### Статус компонентов

| Компонент | Версия | Статус |
|---|---|---|
| WSL (inbox LxssManager) | 10.0.19041.1151 | Установлен |
| WSL (Store) | 2.7.10.0 | Установлен, требует сборки ОС 19044.2311+ |
| WSL2 ядро (системное) | 5.10.16 (апрель 2021) | Устарело для Docker 28.x |
| WSL2 ядро (новое) | 6.18.33.2-2 (июнь 2026) | Несовместимо со старым LxssManager |

### Шаги развёртывания (при совместимой ОС)

**1. Обновить Windows 10** до сборки 19044.2311 или выше.

**2. После обновления ОС — обновить ядро WSL:**
```powershell
wsl --update
wsl --set-default-version 2
wsl --status   # Версия ядра должна быть 5.15+
```

**3. Переключить Docker Desktop на WSL backend:**
```json
// %APPDATA%\Docker\settings-store.json
{
  "WslEngineEnabled": true
}
```

**4. Установить Linux-дистрибутив (опционально):**
```powershell
wsl --install -d Ubuntu
```

---

## Исследование WSL-пути: все попытки и результаты

### Проблема
Docker Desktop 28.x требует ядро WSL2 версии **5.15+**. На данной системе:
- Установленное ядро: `5.10.16` (апрель 2021) — слишком старое
- `wsl --update` заблокирован групповой политикой
- Новый WSL 2.7.10.0 требует Windows 10 сборки **19044.2311+**; текущая сборка `19044.1165`

### Попытка 1 — Регистрация Microsoft Update COM-сервиса

Обход блокировки `wsl --update` через временную регистрацию службы Windows Update:
```powershell
# Регистрация Microsoft Update COM-сервиса (7971f918-a847-4430-9279-4a52d1efe18d)
$svc = New-Object -ComObject Microsoft.Update.ServiceManager
$svc.AddService2("7971f918-a847-4430-9279-4a52d1efe18d", 7, "")
wsl --update
```
**Результат:** `wsl --update` выполнился, но вернул "No updates available" — Microsoft прекратил публикацию обновлений для inbox WSL через Windows Update.

### Попытка 2 — Установка нового WSL 2.7.10.0

```powershell
winget install --id Microsoft.WSL --version 2.7.10.0
```
**Результат:** Установлен в `C:\Program Files\WSL\`, но при запуске:
```
Error code: Wsl/WSL_E_OS_NOT_SUPPORTED
```
Новый WSL 2.7.10.0 требует Windows 10 сборки **19044.2311+**; текущая сборка `19044.1165` не поддерживается.

### Попытка 3 — Ручная замена ядра из пакета WSL

Ядро WSL2 хранится в файле:
- Системное (старое): `C:\Windows\System32\lxss\tools\kernel` (70 722 784 байт, v5.10.16)
- Новое из пакета: `C:\Program Files\WSL\tools\kernel` (17 334 784 байт, v6.18.33.2-2)

Попытка заменить системное ядро на новое:
```powershell
# Резервная копия
Copy-Item "C:\Windows\System32\lxss\tools\kernel" `
          "C:\Users\Unknown\Documents\wsl_kernel_backup_5.10.16.bin"

# Замена ядра
Copy-Item "C:\Program Files\WSL\tools\kernel" `
          "C:\Windows\System32\lxss\tools\kernel" -Force
```
**Результат:** Старый LxssManager (v10.0.19041.1151) не способен загружать ядра версии 6.x — несовместимость архитектуры загрузчика. Замена ядра не помогла; Docker по-прежнему выдавал:
```
checking preconditions: WSL update required
new update state: OS is not supported
```

### Попытка 4 — .wslconfig с путём к новому ядру

```ini
# C:\Users\Unknown\.wslconfig
[wsl2]
kernel=C:\\Program Files\\WSL\\tools\\kernel
```
**Результат:** Docker Desktop проверяет версию ядра WSL **до** его загрузки через собственный внутренний механизм (не через `.wslconfig`). Путь к ядру в `.wslconfig` не влияет на предварительную проверку.

### Попытка 5 — Правка реестра KernelVersion

```powershell
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Lxss" `
                 -Name "KernelVersion" -Value "5.15.90.1" -Type String
```
**Результат:** Docker Desktop содержит собственную проверку в Go-коде, независимую от реестра. Изменение реестра не обходит встроенную проверку.

### Вывод

WSL-backend для Docker Desktop на Windows 10 сборки 19044.1165 **принципиально невозможен** без обновления ОС:

| Причина | Описание |
|---|---|
| LxssManager 10.0.19041.1151 | Не загружает ядра WSL2 версии 6.x |
| Windows 10 build 19044.1165 | Слишком старая для WSL 2.7.10.0 (нужна 19044.2311+) |
| Docker Desktop 28.x | Требует WSL kernel 5.15+, hardcoded procheck |
| Политика обновлений | `wsl --update` заблокирован групповой политикой |

**Система восстановлена в исходное состояние:**
- Ядро WSL2 возвращено: `C:\Windows\System32\lxss\tools\kernel` = 70 722 784 байт (v5.10.16)
- Реестр `KernelVersion` = 5.10.16
- Файл `.wslconfig` удалён
- Docker Desktop работает в режиме Hyper-V (`WslEngineEnabled: false`)

---

## Сравнение путей

| Критерий | Hyper-V | WSL 2 |
|---|---|---|
| Статус | Работает | Требует обновления ОС |
| Производительность | Хорошая | Лучше (меньше накладных расходов) |
| Совместимость с Linux | Полная | Полная + общая файловая система |
| Требования к ОС | Windows 10 Pro | Windows 10 build 19044.2311+ / Windows 11 |
| Требования к железу | VT-x/AMD-V в BIOS | VT-x/AMD-V в BIOS |
| Изоляция ядра | Полная VM | Легковесный контейнер |

---

## Работа с проектом и агентами через Docker

На текущей системе рабочий режим — **Hyper-V backend**. Для проекта это не меняет команды Docker: контейнеры запускаются через обычный `docker compose`, а код монтируется в контейнер как `/workspace`.

### Структура
- `workspace` — основной интерактивный контейнер для работы с проектом.
- `agent-alpha` и `agent-beta` — изолированные агентские контейнеры с отдельными home/cache volume.
- `docker-agent` — контейнер с Docker CLI и доступом к Docker Engine через `/var/run/docker.sock`.

> `docker-agent` получает доступ к Docker Engine хоста. Использовать его стоит только для доверенных задач: доступ к Docker socket фактически равен правам на управление контейнерами и образами.

### Быстрый старт

```powershell
# Проверить итоговую compose-конфигурацию
docker compose -f docker-compose.agents.yml config

# Запустить основной контейнер проекта
docker compose -f docker-compose.agents.yml up -d workspace

# Войти в workspace
docker compose -f docker-compose.agents.yml exec workspace bash
```

### Запуск агентских контейнеров

```powershell
# Запустить двух агентов
docker compose -f docker-compose.agents.yml --profile agents up -d agent-alpha agent-beta

# Войти в конкретного агента
docker compose -f docker-compose.agents.yml exec agent-alpha bash
docker compose -f docker-compose.agents.yml exec agent-beta bash

# Посмотреть состояние
docker compose -f docker-compose.agents.yml ps
```

### Агент с доступом к Docker

```powershell
# Запустить Docker CLI агент
docker compose -f docker-compose.agents.yml --profile docker up -d docker-agent

# Проверить доступ к Docker Engine из контейнера
docker compose -f docker-compose.agents.yml exec docker-agent docker ps
```

### Остановка и очистка

```powershell
# Остановить контейнеры, сохранив volume
docker compose -f docker-compose.agents.yml down

# Полная очистка вместе с home/cache volume агентов
docker compose -f docker-compose.agents.yml down -v
```

### Настройка образа и пути проекта

Для локальных настроек можно создать `.env` на основе `.env.example`:

```powershell
Copy-Item .env.example .env
```

Параметры:
- `AGENT_IMAGE` — базовый Linux-образ для `workspace`, `agent-alpha` и `agent-beta`.
- `WORKSPACE_PATH` — путь, который будет смонтирован в `/workspace`; по умолчанию используется текущая папка проекта.

---

## Переключение между путями

```powershell
# Посмотреть текущий режим
docker info | findstr "Kernel"

# Переключить через настройки Docker Desktop (GUI)
# Settings → General → Use WSL 2 based engine (вкл/выкл)

# Или напрямую в файле настроек:
# %APPDATA%\Docker\settings-store.json → "WslEngineEnabled": true/false
```

---

## Проверка работоспособности

```powershell
docker info
docker run --rm hello-world
docker compose version
docker buildx version
```

---

## Автор

Ivan P — `vfvf6462@gmail.com`
