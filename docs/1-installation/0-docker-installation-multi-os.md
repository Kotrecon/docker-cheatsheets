# Установка Docker на разных операционных системах

## 1. Установка на Windows

### Системные требования

- **Windows 10/11 64-bit**: Pro, Enterprise или Education (сборка 19041 или выше)
- Включенная виртуализация в BIOS/UEFI
- Поддержка WSL 2 (Windows Subsystem for Linux 2)

### Процесс установки

1. **Скачивание Docker Desktop**

   - Перейдите на [официальный сайт Docker](https://www.docker.com/products/docker-desktop/)
   - Скачайте установочный файл Docker Desktop

2. **Установка**

   - Запустите скачанный `Docker Desktop Installer.exe`
   - В мастере установки выберите:
     - ☑️ Enable WSL 2 Features
     - ☑️ Install required Windows components
   - Следуйте инструкциям установщика

3. **Завершение**
   - Перезагрузите компьютер после установки
   - Запустите Docker Desktop из меню "Пуск"
   - Дождитесь завершения первоначальной настройки

## 2. Установка на Linux (Ubuntu/Debian)

### Предварительные требования

- Ubuntu Jammy 22.04 (LTS), Impish 21.10, или Focal 20.04 (LTS)
- Архитектура x86_64 (amd64), armhf, arm64, или s390x

### Пошаговая установка

1. **Обновление пакетного менеджера**

   ```bash
   sudo apt update
   sudo apt upgrade -y
   ```

2. **Установка зависимостей**

   ```bash
   sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release -y
   ```

3. **Добавление официального GPG-ключа Docker**

   ```bash
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
   ```

4. **Добавление Docker APT репозитория**

   ```bash
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```

5. **Установка Docker Engine**

   ```bash
   sudo apt update
   sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
   ```

6. **Запуск и автозагрузка Docker**

   ```bash
   sudo systemctl start docker
   sudo systemctl enable docker
   ```

7. **Настройка прав пользователя**

   ```bash
   sudo usermod -aG docker $USER
   ```

   **Перелогиньтесь** или выполните `newgrp docker` для применения изменений

## 3. Установка на macOS

### Требования

- macOS 10.15 или новее
- Минимум 4 GB RAM
- Intel или Apple Silicon процессор

### Установка

1. Скачайте Docker Desktop с [официального сайта](https://docs.docker.com/desktop/install/mac-install/)
2. Откройте скачанный `.dmg` файл
3. Перетащите Docker.app в папку Applications
4. Запустите Docker из Launchpad

## Проверка установки

После установки на любой системе проверьте работу Docker:

```bash
docker --version
docker run hello-world
```

**Ожидаемый результат**: Вывод версии Docker и успешный запуск тестового контейнера.

## Дополнительные настройки

### Для Linux

```bash
# Проверка статуса службы
sudo systemctl status docker

# Просмотр логов
sudo journalctl -u docker.service
```

### Для Windows/macOS

- Откройте Docker Desktop Dashboard
- Настройте ресурсы в Settings → Resources
- Проверьте статус в системном трее

**Примечание**: На Linux рекомендуется использовать Docker без sudo через добавление пользователя в группу docker.
