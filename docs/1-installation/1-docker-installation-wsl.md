# Docker Installation. WSL

**Docker in WSL** — это оптимальное решение для разработки на Windows, сочетающее производительность Linux-контейнеров с удобством Windows-окружения.

## Архитектура Docker в WSL 2

### Схема работы

```bash
┌─────────────────────────────────────────────────┐
│               Windows Host                      │
│                                                 │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │ Docker      │        │   WSL 2 VM          │ │
│  │ Desktop     │        │                     │ │
│  │             │        │  ┌─────────────┐    │ │
│  │  GUI Tools  │◄───────┤  │ Docker      │    │ │
│  │  Dashboard  │        │  │ Engine      │    │ │
│  └─────────────┘        │  └─────────────┘    │ │
│          │              │         │           │ │
│          ▼              │         ▼           │ │
│  ┌──────────────┐       │  ┌─────────────┐    │ │
│  │ Windows      │       │  │ Linux       │    │ │
│  │CLI/PowerShell│──────►│  │ Containers  │    │ │
│  └──────────────┘       │  └─────────────┘    │ │
│                         └─────────────────────┘ │
└─────────────────────────────────────────────────┘
```

## Установка Docker в WSL

### Предварительные требования

```bash
# Проверка версии WSL
wsl --list --verbose

# Требования:
# - Windows 10 version 2004 или новее
# - WSL 2 enabled
# - Virtualization enabled в BIOS
```

### Установка Docker Desktop

1. **Скачивание и установка:**

   - Загрузите [Docker Desktop](https://www.docker.com/products/docker-desktop/)
   - Установите с опцией "Use WSL 2 based engine"

2. **Настройка WSL интеграции:**

   ```bash
   # В Docker Desktop Settings → Resources → WSL Integration
   # Включите интеграцию с вашими WSL дистрибутивами
   ```

### Ручная установка Docker Engine в WSL

```bash
# Обновление пакетов
sudo apt update && sudo apt upgrade -y

# Установка зависимостей
sudo apt install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Добавление Docker GPG ключа
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Добавление репозитория
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Установка Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Добавление пользователя в группу docker
sudo usermod -aG docker $USER

# Запуск службы
sudo systemctl enable docker
sudo systemctl start docker

# Проверка установки
docker --version
docker run hello-world
```

## Перенос WSL дистрибутива на другой диск

### Экспорт дистрибутива

```bash
# Просмотр установленных дистрибутивов
wsl --list --verbose

# Экспорт дистрибутива в файл
wsl --export Ubuntu-22.04 C:\wsl-backup\ubuntu-22.04.tar

# Экспорт с конкретным дистрибутивом
wsl --export Ubuntu-22.04 D:\wsl\ubuntu-22.04.tar --vhd
```

### Импорт на новый диск

```bash
# Импорт на другой диск
wsl --import Ubuntu-22.04-new D:\wsl\ubuntu-22.04\ D:\wsl\ubuntu-22.04.tar --version 2

# Указание версии WSL
wsl --import Ubuntu-22.04 D:\wsl\ubuntu-22.04\ ubuntu-22.04.tar --version 2

# Установка как дистрибутива по умолчанию
wsl --set-default Ubuntu-22.04-new
```

### Пакетный скрипт для миграции

```powershell
# migrate-wsl.ps1
$DistroName = "Ubuntu-22.04"
$BackupPath = "D:\wsl-backup"
$NewPath = "E:\wsl\ubuntu-22.04"

# Остановка дистрибутива
wsl --terminate $DistroName

# Экспорт
wsl --export $DistroName "$BackupPath\$DistroName.tar"

# Удаление старого дистрибутива
wsl --unregister $DistroName

# Импорт в новое расположение
wsl --import $DistroName $NewPath "$BackupPath\$DistroName.tar" --version 2

# Установка как дистрибутива по умолчанию
wsl --set-default $DistroName

Write-Host "Миграция $DistroName завершена успешно!"
```

## Установка из готового образа Ubuntu

### Скачивание официального образа

```bash
# Скачивание через Microsoft Store
# - Откройте Microsoft Store
# - Найдите "Ubuntu 22.04 LTS"
# - Установите приложение

# Или через командную строку
wsl --install -d Ubuntu-22.04
```

### Установка кастомного образа

```bash
# Скачивание официального дистрибутива
Invoke-WebRequest -Uri https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-amd64-wsl.rootfs.tar.gz -OutFile ubuntu-22.04.tar.gz

# Импорт как нового дистрибутива
wsl --import Ubuntu-22.04-custom D:\wsl\ubuntu-custom\ ubuntu-22.04.tar.gz

# Запуск и настройка
wsl -d Ubuntu-22.04-custom
```

### Автоматизированная установка с Docker

```powershell
# install-docker-wsl.ps1
param(
    [string]$DistroName = "Ubuntu-Docker",
    [string]$InstallPath = "D:\wsl\$DistroName",
    [string]$TarPath = "ubuntu-22.04-docker.tar"
)

# Скачивание базового образа
if (-not (Test-Path $TarPath)) {
    Write-Host "Скачивание образа Ubuntu..."
    Invoke-WebRequest -Uri "https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-amd64-wsl.rootfs.tar.gz" -OutFile "ubuntu-22.04.tar.gz"

    # Распаковка
    tar -xzf ubuntu-22.04.tar.gz
    Rename-Item rootfs.tar $TarPath
}

# Импорт дистрибутива
Write-Host "Импорт дистрибутива $DistroName..."
wsl --import $DistroName $InstallPath $TarPath --version 2

# Запуск и установка Docker
Write-Host "Установка Docker в $DistroName..."
$installScript = @"
#!/bin/bash
apt update && apt upgrade -y
apt install -y curl gnupg lsb-release

# Установка Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Настройка пользователя
usermod -aG docker $USER

# Создание скрипта инициализации
echo 'sudo service docker start' >> ~/.bashrc
"@

$installScript | Out-File -FilePath "install-docker.sh" -Encoding ASCII
wsl -d $DistroName -u root bash -c "chmod +x /install-docker.sh && /install-docker.sh"

Write-Host "Установка завершена! Запуск: wsl -d $DistroName"
```

## Оптимизация производительности

### Настройка WSL конфигурации

```ini
# .wslconfig в %UserProfile%
[wsl2]
memory=8GB
processors=4
swap=4GB
localhostForwarding=true

# Оптимизация для Docker
kernelCommandLine = vsyscall=emulate
```

### Конфигурация Docker в WSL

```bash
# /etc/docker/daemon.json в WSL
{
  "storage-driver": "overlay2",
  "data-root": "/mnt/wsl/docker-data",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  }
}
```

## Интеграция с Windows

### Доступ к файловым системам

```bash
# Доступ к Windows дискам из WSL
/mnt/c/Users/YourName
/mnt/d/Projects

# Доступ к WSL из Windows
\\wsl$\Ubuntu-22.04\home\username
```

### Сетевые порты

```bash
# Проброс портов автоматически работает
# localhost:8080 в Windows → localhost:8080 в WSL

# Проверка проброса портов
netstat -an | grep 8080
```

## Резервное копирование и восстановление

### Автоматический бэкап

```powershell
# backup-wsl.ps1
$BackupDir = "D:\wsl-backups"
$Date = Get-Date -Format "yyyy-MM-dd"
$Distros = @("Ubuntu-22.04", "Ubuntu-Docker")

foreach ($Distro in $Distros) {
    $BackupFile = "$BackupDir\$Distro-$Date.tar"
    Write-Host "Бэкап $Distro в $BackupFile"
    wsl --export $Distro $BackupFile
}
```

### Восстановление из бэкапа

```powershell
# restore-wsl.ps1
param(
    [string]$DistroName = "Ubuntu-22.04",
    [string]$BackupFile = "D:\wsl-backups\ubuntu-22.04-2024-01-15.tar",
    [string]$RestorePath = "D:\wsl\ubuntu-restored"
)

wsl --terminate $DistroName
wsl --unregister $DistroName
wsl --import $DistroName $RestorePath $BackupFile --version 2
wsl --set-default $DistroName
```

## Устранение проблем

### Распространенные проблемы и решения

```bash
# Ошибка: Cannot connect to the Docker daemon
sudo service docker start

# Ошибка: WSL 2 installation is incomplete
wsl --set-version Ubuntu-22.04 2

# Ошибка: No space left on device
wsl --shutdown
wsl --export Ubuntu-22.04 backup.tar
wsl --unregister Ubuntu-22.04
wsl --import Ubuntu-22.04 D:\new-path\ backup.tar
```

### Диагностика WSL

```bash
# Проверка состояния WSL
wsl --status

# Проверка версии WSL
wsl -l -v

# Логи WSL
Get-EventLog -LogName Application -Source "*WSL*" -Newest 10
```

**Архитектурный принцип**: Docker в WSL 2 обеспечивает нативную производительность Linux-контейнеров в Windows-окружении, сочетая преимущества обеих платформ через эффективную виртуализацию и бесшовную интеграцию файловых систем и сетей.
