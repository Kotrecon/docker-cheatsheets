# Docker для разработки

Практическое руководство по настройке эффективной Docker-среды для разработки .NET 7 и JavaScript/TypeScript приложений с hot reload, отладкой и оптимизированными workflow.

## 🎯 Задача

Создать локальную среду разработки с Docker, которая обеспечивает:

- Hot reload для быстрой разработки
- Отладку кода внутри контейнеров
- Эмуляцию продакшен-окружения
- Интеграцию с IDE и инструментами разработки

## 📁 Структура проекта разработки

```
dev-project/
├── docker-compose.dev.yml          # Dev-специфичная конфигурация
├── docker-compose.override.yml     # Переопределения для разработки
├── .dockerignore                   # Исключения для разработки
├── backend/                        # .NET 7 Backend
│   ├── Dockerfile.dev
│   ├── src/
│   └── appsettings.Development.json
├── frontend/                       # JavaScript/TypeScript Frontend
│   ├── Dockerfile.dev
│   ├── src/
│   └── package.json
├── database/                       # Локальные БД
│   └── init-scripts/
└── monitoring/                     # Dev-мониторинг
    ├── prometheus.yml
    └── grafana/
```

## 🐳 Docker конфигурация для разработки

### Dockerfile для .NET 7 (Development)

```dockerfile
# Development образ для .NET 7
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS development

# Установка инструментов для отладки
RUN apt-get update && apt-get install -y \
    curl \
    procps \
    && rm -rf /var/lib/apt/lists/*

# Настройка для отладки
ENV ASPNETCORE_ENVIRONMENT=Development
ENV DOTNET_USE_POLLING_FILE_WATCHER=1
ENV DOTNET_CLI_TELEMETRY_OPTOUT=1

WORKDIR /app

# Копирование проектов
COPY *.sln .
COPY backend/*.csproj ./backend/
RUN dotnet restore

# Точка входа для разработки
CMD ["dotnet", "watch", "run", "--urls", "http://0.0.0.0:5000"]
```

### Dockerfile для Node.js (Development)

```dockerfile
# Development образ для Node.js
FROM node:18-alpine AS development

# Установка дополнительных инструментов
RUN apk add --no-cache \
    curl \
    bash

WORKDIR /app

# Копирование package файлов
COPY package*.json ./
COPY frontend/package*.json ./frontend/

# Установка зависимостей
RUN npm install

# Копирование исходного кода
COPY frontend/ ./frontend/

# Настройка hot reload
ENV CHOKIDAR_USEPOLLING=true
ENV WATCHPACK_POLLING=true

CMD ["npm", "run", "dev"]
```

### docker-compose.dev.yml

```yaml
version: "3.8"

services:
  backend-dev:
    build:
      context: .
      dockerfile: backend/Dockerfile.dev
    ports:
      - "5000:5000"
      - "5001:5001" # HTTPS для .NET
    volumes:
      - ./backend:/app
      - /app/backend/bin
      - /app/backend/obj
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - DOTNET_WATCH_RESTART_ON_RUDE_EDIT=true
    networks:
      - dev-network

  frontend-dev:
    build:
      context: .
      dockerfile: frontend/Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app/frontend
      - /app/frontend/node_modules
    environment:
      - NODE_ENV=development
      - CHOKIDAR_USEPOLLING=true
    networks:
      - dev-network

  database-dev:
    image: postgres:15-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=devdb
      - POSTGRES_USER=developer
      - POSTGRES_PASSWORD=devpass
    volumes:
      - postgres_dev_data:/var/lib/postgresql/data
      - ./database/init-scripts:/docker-entrypoint-initdb.d
    networks:
      - dev-network

  redis-dev:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - dev-network

  # Dev-мониторинг
  prometheus-dev:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - dev-network

  grafana-dev:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_dev_data:/var/lib/grafana
    networks:
      - dev-network

volumes:
  postgres_dev_data:
  grafana_dev_data:

networks:
  dev-network:
    driver: bridge
```

## 🔧 Интеграция с IDE

### Visual Studio Code - .launch.json

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Docker .NET Debug",
      "type": "coreclr",
      "request": "attach",
      "processId": "1",
      "pipeTransport": {
        "pipeProgram": "docker",
        "pipeArgs": ["exec", "-i", "backend-dev"],
        "debuggerPath": "/vsdbg/vsdbg",
        "pipeCwd": "${workspaceFolder}",
        "quoteArgs": false
      },
      "sourceFileMap": {
        "/app": "${workspaceFolder}/backend"
      }
    },
    {
      "name": "Docker Node.js Debug",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "address": "localhost",
      "localRoot": "${workspaceFolder}/frontend",
      "remoteRoot": "/app/frontend",
      "restart": true
    }
  ]
}
```

### Visual Studio - Docker Compose поддержка

```xml
<!-- .csproj для Docker поддержки -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net7.0</TargetFramework>
    <DockerDefaultTargetOS>Linux</DockerDefaultTargetOS>
    <DockerComposeProjectPath>..\docker-compose.dev.yml</DockerComposeProjectPath>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.VisualStudio.Azure.Containers.Tools.Targets" Version="1.19.5" />
  </ItemGroup>
</Project>
```

## 🚀 Запуск и использование

### Базовый запуск

```bash
# Запуск всей dev-среды
docker-compose -f docker-compose.dev.yml up -d

# Просмотр логов
docker-compose -f docker-compose.dev.yml logs -f backend-dev

# Остановка
docker-compose -f docker-compose.dev.yml down

# Пересборка и запуск
docker-compose -f docker-compose.dev.yml up -d --build
```

### Индивидуальный запуск сервисов

```bash
# Только backend с hot reload
docker-compose -f docker-compose.dev.yml up backend-dev

# Только frontend с hot reload
docker-compose -f docker-compose.dev.yml up frontend-dev

# Запуск с кастомными портами
docker-compose -f docker-compose.dev.yml up -p 3002:3000 frontend-dev
```

## ⚙️ Конфигурационные файлы

### .dockerignore для разработки

```dockerignore
**/node_modules
**/bin
**/obj
**/.git
**/Dockerfile*
**/docker-compose*
**/.env
**/npm-debug.log
**/yarn-error.log
**/.vscode
**/.idea
```

### prometheus.yml для dev-мониторинга

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "backend-dev"
    static_configs:
      - targets: ["backend-dev:5000"]
    metrics_path: "/metrics"

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

## 🔧 Настройка для разных окружений

### Development-специфичные настройки

```yaml
# docker-compose.override.yml
version: "3.8"

services:
  backend-dev:
    environment:
      - ConnectionStrings__DefaultConnection=Host=database-dev;Database=devdb;Username=developer;Password=devpass
      - Logging__LogLevel__Default=Debug
    labels:
      - "dev.monitoring.enabled=true"

  frontend-dev:
    environment:
      - API_URL=http://localhost:5000
      - ENABLE_SOURCE_MAPS=true
    labels:
      - "dev.monitoring.enabled=true"
```

## 📊 Мониторинг и логи в разработке

### Просмотр логов в реальном времени

```bash
# Все логи
docker-compose -f docker-compose.dev.yml logs -f

# Фильтрация по сервису
docker-compose -f docker-compose.dev.yml logs -f backend-dev

# Логи с временными метками
docker-compose -f docker-compose.dev.yml logs -f --timestamps

# Только ошибки
docker-compose -f docker-compose.dev.yml logs --tail=100 | grep -i error
```

### Мониторинг ресурсов

```bash
# Использование ресурсов
docker stats

# Инспекция контейнеров
docker inspect backend-dev

# Проверка сетей
docker network ls
docker network inspect dev-project_dev-network
```

## 🛠️ Troubleshooting

### Частые проблемы и решения

1. **Проблема**: Hot reload не работает в Docker
   **Решение**:

   ```yaml
   environment:
     - CHOKIDAR_USEPOLLING=true # Для Node.js
     - DOTNET_USE_POLLING_FILE_WATCHER=1 # Для .NET
   volumes:
     - ./src:/app/src # Правильное маппинг томов
   ```

2. **Проблема**: Отладка не подключается
   **Решение**:

   ```bash
   # Проверка отладочных портов
   docker port backend-dev
   # Перезапуск с отладочными флагами
   docker-compose -f docker-compose.dev.yml up --force-recreate backend-dev
   ```

3. **Проблема**: Медленная работа в Windows/WSL2
   **Решение**:
   ```yaml
   # В WSL2 использовать дефолтный драйвер
   volumes:
     - /c/Dev/project:/app # Прямой путь для производительности
   ```

## 💡 Best Practices для разработки

### Производительность

- Используйте `.dockerignore` для исключения ненужных файлов
- Настройте кэширование зависимостей в Dockerfile
- Используйте named volumes для node_modules и пакетов

### Отладка

- Настройте source maps для фронтенда
- Используйте remote debugging в IDE
- Добавьте health checks для мониторинга

### Безопасность

- Не используйте root пользователя в dev-контейнерах
- Отдельные пароли для dev-баз данных
- Ограничьте экспортируемые порты

### Оптимизация workflow

- Автоматический restart при изменении кода
- Интеграция с pre-commit hooks
- Локальное тестирование в изоляции

**Ключевой вывод**: Правильно настроенная Docker-среда разработки ускоряет iteration cycle, обеспечивает консистентность окружений и позволяет отлаживать приложения в условиях, близких к продакшену.
