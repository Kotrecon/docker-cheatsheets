# Docker Compose Guide

Полное руководство по работе с Docker Compose для управления multi-container приложениями.

## 🚀 Что такое Docker Compose?

Docker Compose - это инструмент для определения и запуска multi-container Docker приложений. С помощью YAML-файла вы можете настроить все сервисы вашего приложения и запустить их одной командой.

## 📁 Базовая структура compose.yaml

```yaml
version: "3.8"

services:
  web:
    build: .
    ports:
      - "8000:8000"
    depends_on:
      - db
      - redis

  db:
    image: postgres:13
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - db_data:/var/lib/postgresql/data

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

volumes:
  db_data:
```

## ⚙️ Основные команды Compose

### `docker compose up`

**Запуск сервисов**

```bash
docker compose up [OPTIONS] [SERVICE...]
```

**Опции:**

- `-d` - запуск в фоновом режиме
- `--build` - пересборка образов перед запуском
- `--force-recreate` - принудительное пересоздание контейнеров
- `--no-deps` - не запускать зависимости

**Примеры:**

```bash
docker compose up              # Запуск всех сервисов
docker compose up -d           # Запуск в фоне
docker compose up web db       # Запуск только web и db
docker compose up --build      # Пересборка и запуск
```

### `docker compose down`

**Остановка сервисов**

```bash
docker compose down [OPTIONS]
```

**Опции:**

- `-v` - удаление томов
- `--rmi all` - удаление всех образов
- `--remove-orphans` - удаление orphan-контейнеров

**Примеры:**

```bash
docker compose down           # Остановка с сохранением томов
docker compose down -v        # Остановка с удалением томов
docker compose down --rmi all # Остановка с удалением образов
```

### `docker compose ps`

**Просмотр статуса сервисов**

```bash
docker compose ps [SERVICE...]
```

### `docker compose logs`

**Просмотр логов**

```bash
docker compose logs [OPTIONS] [SERVICE...]
```

**Опции:**

- `-f` - слежение за логами в реальном времени
- `--tail=N` - показать последние N строк

**Примеры:**

```bash
docker compose logs          # Все логи
docker compose logs web      # Логи только web-сервиса
docker compose logs -f web   # Слежение за логами web
```

## 🛠️ Продвинутые команды

### `docker compose exec`

**Выполнение команд в контейнере сервиса**

```bash
docker compose exec [OPTIONS] SERVICE COMMAND [ARG...]
```

**Примеры:**

```bash
docker compose exec web bash
docker compose exec db psql -U user myapp
docker compose exec redis redis-cli
```

### `docker compose build`

**Сборка образов**

```bash
docker compose build [OPTIONS] [SERVICE...]
```

**Опции:**

- `--no-cache` - сборка без кеша
- `--pull` - принудительное скачивание базовых образов

### `docker compose restart`

**Перезапуск сервисов**

```bash
docker compose restart [OPTIONS] [SERVICE...]
```

### `docker compose pause/unpause`

**Приостановка и возобновление сервисов**

```bash
docker compose pause [SERVICE...]
docker compose unpause [SERVICE...]
```

## 📈 Масштабирование сервисов

### Базовое масштабирование

```bash
# Масштабирование веб-сервиса до 3 экземпляров
docker compose up --scale web=3
```

### Конфигурация для масштабирования

```yaml
services:
  web:
    build: .
    ports:
      - "8080-8090:80" # Диапазон портов для масштабирования
    environment:
      - LOAD_BALANCER_URL=haproxy:80

  haproxy:
    image: haproxy:latest
    ports:
      - "80:80"
    depends_on:
      - web
```

### Особенности масштабирования

- **Stateless сервисы**: Идеально подходят для масштабирования (веб-серверы, API)
- **Stateful сервисы**: Требуют дополнительной настройки (базы данных, кэши)
- **Балансировка нагрузки**: Используйте Nginx, HAProxy или встроенные решения

## 📋 Конфигурация сервисов

### Базовые настройки сервиса

```yaml
services:
  app:
    image: nginx:latest # Используемый образ
    build: . # Сборка из Dockerfile
    container_name: myapp # Имя контейнера

    ports:
      - "80:80" # Проброс портов
      - "443:443"

    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb

    env_file:
      - .env # Файл с переменными окружения
      - .env.db

    volumes:
      - ./app:/app # Bind mount
      - logs:/var/log # Named volume

    networks:
      - frontend
      - backend

    depends_on:
      - database
      - cache
```

### Настройки здоровья (Healthcheck)

```yaml
services:
  web:
    image: nginx
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### Ресурсы и ограничения

```yaml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
    restart: unless-stopped
```

## 🌐 Сети в Compose

### Настройка сетей

```yaml
services:
  proxy:
    image: nginx
    networks:
      - frontend

  app:
    image: node:16
    networks:
      - frontend
      - backend

  database:
    image: postgres
    networks:
      - backend

networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
  backend:
    driver: bridge
```

## 💾 Тома данных

### Типы томов

```yaml
services:
  database:
    image: postgres
    volumes:
      # Named volume
      - db_data:/var/lib/postgresql/data

      # Bind mount
      - ./postgres.conf:/etc/postgresql.conf

      # Anonymous volume
      - /tmp

      # Volume с настройками
      - logs:/var/log:ro # Только для чтения

volumes:
  db_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /path/on/host
```

## 🎯 Практические примеры для .NET 7 и JavaScript/TypeScript

### .NET 7 Web API с PostgreSQL

```yaml
version: "3.8"

services:
  api:
    build: .
    ports:
      - "5000:80"
      - "5001:443"
    environment:
      - ConnectionStrings__DefaultConnection=Host=db;Database=myapp;Username=postgres;Password=postgres
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=https://+:443;http://+:80
    volumes:
      - ~/.aspnet/https:/https:ro
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

### .NET 7 с SQL Server

```yaml
version: "3.8"

services:
  webapi:
    build: .
    ports:
      - "5000:80"
    environment:
      - ConnectionStrings__DefaultConnection=Server=db;Database=myapp;User=sa;Password=YourPassword123;
      - ASPNETCORE_ENVIRONMENT=Development
    depends_on:
      db:
        condition: service_healthy

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      SA_PASSWORD: "YourPassword123!"
      ACCEPT_EULA: "Y"
      MSSQL_PID: "Developer"
    ports:
      - "1433:1433"
    volumes:
      - sql_data:/var/opt/mssql
    healthcheck:
      test: /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "$$SA_PASSWORD" -Q "SELECT 1" || exit 1
      interval: 10s
      timeout: 3s
      retries: 10
      start_period: 30s

volumes:
  sql_data:
```

### Node.js/TypeScript Backend с Hot Reload

```yaml
version: "3.8"

services:
  backend:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/myapp
      - CHOKIDAR_USEPOLLING=true
    command: npm run dev:watch
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

### Full-Stack .NET 7 API + React/TypeScript Frontend

```yaml
version: "3.8"

services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - ./frontend/src:/app/src
      - ./frontend/public:/app/public
      - /app/node_modules
    environment:
      - REACT_APP_API_URL=http://localhost:5000
      - CHOKIDAR_USEPOLLING=true
    command: npm start
    depends_on:
      - api

  api:
    build: ./api
    ports:
      - "5000:80"
    environment:
      - ConnectionStrings__DefaultConnection=Host=db;Database=myapp;Username=postgres;Password=postgres
      - ASPNETCORE_ENVIRONMENT=Development
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### .NET 7 с Redis для кэширования

```yaml
version: "3.8"

services:
  webapp:
    build: .
    ports:
      - "5000:80"
    environment:
      - ConnectionStrings__DefaultConnection=Host=db;Database=myapp;Username=postgres;Password=postgres
      - RedisConnection=redis:6379
      - ASPNETCORE_ENVIRONMENT=Development
    depends_on:
      - db
      - redis

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

## 🔧 Полезные сценарии

### Разработка

```bash
# Запуск для разработки
docker compose up

# Пересборка и запуск
docker compose up --build

# Запуск только определенных сервисов
docker compose up frontend backend

# Просмотр логов в реальном времени
docker compose logs -f
```

### Продакшен

```bash
# Запуск в фоне
docker compose up -d

# Проверка статуса
docker compose ps

# Остановка
docker compose down

# Обновление
docker compose pull
docker compose up -d
```

### Отладка

```bash
# Выполнение команд в контейнере .NET
docker compose exec api bash

# Проверка подключения к БД
docker compose exec db psql -U postgres -d myapp

# Просмотр логов Node.js приложения
docker compose logs -f backend
```

## ⚠️ Лучшие практики для .NET и Node.js

### Для .NET:

- Используйте многостадийные сборки для уменьшения размера образов
- Настройте healthchecks для проверки готовности приложения
- Используйте переменные окружения для конфигурации
- Настройте HTTPS в продакшене

### Для Node.js:

- Используйте .dockerignore для исключения node_modules
- Настройте hot reload для разработки
- Используйте alpine образы для уменьшения размера
- Настройте правильные рабочие директории

## 📝 Примечания

- Compose V2 теперь часть Docker CLI (`docker compose` вместо `docker-compose`)
- Файлы по умолчанию: `compose.yaml`, `docker-compose.yml`
- Используйте `--env-file` для указания файла с переменными
- Профили позволяют запускать группы сервисов: `docker compose --profile debug up`

Этот гайд покрывает основные сценарии использования Docker Compose для .NET 7 и JavaScript/TypeScript приложений.
[file content end]

---

Теперь только релевантные примеры для твоего стека! 🎯 .NET 7 и JavaScript/TypeScript готовы к использованию.
