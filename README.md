# 🐳 Docker Шпаргалки и Руководства

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub last commit](https://img.shields.io/github/last-commit/Kotrecon/docker-cheatsheets)](https://github.com/Kotrecon/docker-cheatsheets)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

Полный набор профессиональных шпаргалок, руководств и **best practices** по Docker для разработчиков .NET и JavaScript/TypeScript на русском языке.

## 📚 Содержание

### 🏗️ [Архитектура Docker](docs/0-architecture/docker-architecture.md)

- [Слои файловой системы](docs/0-architecture/0-docker-architecture-layers.md)
- [Образы и контейнеры](docs/0-architecture/1-docker-architecture-images.md)
- [Сетевая инфраструктура](docs/0-architecture/4-docker-architecture-networking.md)
- [Драйверы хранения](docs/0-architecture/3-docker-architecture-storage-drivers.md)
- [Процесс сборки](docs/0-architecture/7-docker-architecture-build-process.md)

### ⚙️ [Установка и настройка](docs/1-installation/docker-installation-configuration.md)

- [Multi-OS установка](docs/1-installation/0-docker-installation-multi-os.md)
- [Docker в WSL](docs/1-installation/1-docker-installation-wsl.md)
- [Настройка демона](docs/1-installation/2-docker-daemon-configuration.md)
- [CI/CD интеграция](docs/1-installation/3-docker-in-ci-cd-pipelines.md)
- [Автоматизация с Ansible](docs/1-installation/4-automated-docker-deployment-with-ansible.md)

### 🛠️ [Команды и операции](docs/2-commands/docker-commands-operations.md)

- [Полный CLI справочник](docs/2-commands/0-docker-cli-reference.md)
- [Docker Compose гайды](docs/2-commands/1-docker-compose-guide.md)
- [Управление системой](docs/2-commands/2-docker-system-management.md)
- [Продвинутые команды](docs/2-commands/3-advanced-docker-commands.md)
- [Dockerfile reference](docs/2-commands/4-dockerfile-reference.md)

### 🔐 [Безопасность](docs/3-security/docker-security.md)

- [Best practices](docs/3-security/0-docker-security-best-practices.md)
- [Content Trust](docs/3-security/1-docker-content-trust.md)
- [Управление секретами](docs/3-security/2-docker-secrets-management.md)

### 🚀 [Практические сценарии](docs/4-practical-scenarios/docker-practical-scenarios.md)

- [Разработка с Docker](docs/4-practical-scenarios/0-docker-for-development.md)
- [Production best practices](docs/4-practical-scenarios/1-docker-in-production.md)
- [Миграция приложений](docs/4-practical-scenarios/2-migrating-applications-to-docker.md)

### 📋 [Готовые шаблоны](docs/4-practical-scenarios/templates/)

- [.NET Console App](docs/4-practical-scenarios/templates/net-console-app-in-docker.md)
- [ASP.NET Core](docs/4-practical-scenarios/templates/aspnet-core-in-docker.md)
- [Node.js + Nginx](docs/4-practical-scenarios/templates/nodejs-nginx-in-docker.md)
- [PostgreSQL](docs/4-practical-scenarios/templates/postgresql-in-docker.md)
- [Elasticsearch](docs/4-practical-scenarios/templates/elasticsearch-in-docker.md)
- [Keycloak](docs/4-practical-scenarios/templates/keycloak-in-docker.md)

## 🎯 Технологический фокус

### .NET 7/8

- Многостадийные сборки
- Entity Framework Core миграции
- Health Checks и мониторинг
- Оптимизированные образы

### JavaScript/TypeScript

- Hot reload разработка
- Alpine-based образы Node.js
- Nginx для статики
- TypeScript поддержка

### Универсальные шаблоны

- Production-ready конфигурации
- Best practices безопасности
- Мониторинг и логирование
- Масштабируемые архитектуры

## 📥 Быстрый старт

```bash
# Клонировать репозиторий
git clone https://github.com/Kotrecon/docker-cheatsheets.git

# Перейти в папку проекта
cd docker-cheatsheets

# Открыть в VS Code
code .
```

## 🏃‍♂️ Как использовать

### Для изучения Docker

1. Начните с раздела **Архитектура**
2. Перейдите к **Командам и операциям**
3. Изучите **Безопасность**
4. Примените знания в **Практических сценариях**

### Для работы с проектами

- Используйте готовые шаблоны из папки `templates/`
- Копируйте команды из справочников
- Следуйте best practices из руководств

### Для .NET разработчиков

```dockerfile
# Используйте оптимизированные многостадийные сборки
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
# ... смотрите примеры в шаблонах
```

### Для Node.js разработчиков

```dockerfile
# Используйте Alpine образы для минимального размера
FROM node:18-alpine
# ... смотрите примеры в шаблонах
```

## 🛠️ Инструменты и технологии

- **Docker & Docker Compose**
- **Trivy** - сканирование уязвимостей
- **Dockle** - проверка best practices
- **WSL 2** - разработка на Windows
- **GitHub Actions** - CI/CD пайплайны

## 🤝 Участие в развитии

Приветствуются улучшения и дополнения! См. [CONTRIBUTING.md](CONTRIBUTING.md) для подробностей.

### Как помочь

- Исправление опечаток и ошибок
- Добавление новых примеров
- Создание дополнительных шаблонов
- Улучшение документации

## 📊 Статистика проекта

- **📁 5 разделов** полного цикла изучения
- **📄 33 файла** документации
- **🎯 2 основных стека** (.NET + JS/TS)
- **🚀 9 готовых шаблонов**
- **🔐 Полное руководство по безопасности**

---

## 👨‍💻 Автор

**`@Kotrecon`**

Архитектор решений из Санкт-Петербурга. Специализация: .NET, C#, JS, AI/ML, RAG, DevOps, DB, промышленная автоматизация.  
[Telegram](https://t.me/Kotrecon) | [Email](mailto:ermakov_k@mail.ru)

---

## 🏷️ Версия

| Версия    | Дата         | Изменения            |
| --------- | ------------ | -------------------- |
| **1.0.0** | Декабрь 2025 | Начальная реализация |

---

## 📄 Лицензия

Этот проект распространяется под лицензией MIT - см. файл [LICENSE](LICENSE).

---
