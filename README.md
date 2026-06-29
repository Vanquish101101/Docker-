# Docker — Установка и Развёртывание

## v1.0.0 | Два пути развёртывания: Hyper-V & WSL

### Описание

Полная установка и настройка Docker Desktop 4.49.0 на Windows 10 Pro (сборка 19044).  
Реализовано **два пути развёртывания**: через **Hyper-V** (активен) и через **WSL 2** (подготовлен, требует обновления ОС).

---

### Системное окружение

| Параметр | Значение |
|---|---|
| ОС | Windows 10 Pro 10.0.19044 |
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

## Путь 2 — WSL 2 Backend ⚙️ (подготовлен)

### Когда использовать
- Windows 10 сборка 19044.2311+ или Windows 11
- Политики обновления Windows не ограничены

### Статус компонентов

| Компонент | Версия | Статус |
|---|---|---|
| WSL (inbox) | 10.0.19041.1151 | Установлен |
| WSL (Store) | 2.7.10.0 | Установлен, требует обновления ОС |
| WSL2 ядро | 5.10.16 | Устарело для Docker 28.x |

### Шаги развёртывания

**1. Обновить Windows 10** до сборки 19044.2311 или выше:
```powershell
# Скачать обновление вручную с Microsoft Update Catalog
# KB5020030 или позднее для Windows 10 21H2
```

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

### Почему WSL backend ограничен сейчас

- Ядро WSL2 `5.10.16` (апрель 2021) слишком старое для Docker Desktop 28.x
- `wsl --update` заблокирован групповой политикой (параметр Windows Update отключён)
- Новый WSL `2.7.10.0` установлен, но требует Windows 10 сборки 19044.2311+

---

## Сравнение путей

| Критерий | Hyper-V | WSL 2 |
|---|---|---|
| Статус | Работает | Требует обновления ОС |
| Производительность | Хорошая | Лучше (меньше накладных расходов) |
| Совместимость с Linux | Полная | Полная + общая файловая система |
| Требования к ОС | Windows 10 Pro | Любая редакция Windows 10/11 |
| Требования к железу | VT-x/AMD-V в BIOS | VT-x/AMD-V в BIOS |

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
