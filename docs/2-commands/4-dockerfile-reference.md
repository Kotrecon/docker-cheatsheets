# Dockerfile Reference

Полный справочник по инструкциям Dockerfile с примерами для .NET 7 и JavaScript/TypeScript.

## 📝 Базовые инструкции

### `FROM`

**Задает базовый (родительский) образ**

```dockerfile
FROM <image>[:<tag>] [AS <name>]
```

**Примеры для разных технологий:**

```dockerfile
# .NET 7
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS runtime
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build

# Node.js
FROM node:18-alpine AS development
FROM node:18-alpine AS builder

# Универсальные
FROM alpine:latest
FROM ubuntu:20.04
```

### `LABEL`

**Добавляет метаданные к образу**

```dockerfile
LABEL <key>=<value> <key>=<value> ...
```

**Примеры:**

```dockerfile
LABEL maintainer="dev@example.com"
LABEL version="1.0" description="My application"
LABEL com.example.technology="dotnet-7.0"
LABEL com.example.framework="node-18"
```

### `ENV`

**Устанавливает переменные окружения**

```dockerfile
ENV <key> <value>
ENV <key>=<value> ...
```

**Примеры для разных технологий:**

```dockerfile
# .NET
ENV ASPNETCORE_ENVIRONMENT=Production
ENV DOTNET_CLI_TELEMETRY_OPTOUT=1
ENV ASPNETCORE_URLS=http://+:80

# Node.js
ENV NODE_ENV=production
ENV NPM_CONFIG_LOGLEVEL=warn
ENV HOST=0.0.0.0

# Общие
ENV PATH="/app/bin:${PATH}"
```

## 🛠️ Инструкции сборки

### `RUN`

**Выполняет команды и создает слой образа**

```dockerfile
RUN <command>                         # shell форма
RUN ["executable", "param1", "param2"] # exec форма
```

**Примеры для разных технологий:**

```dockerfile
# .NET
RUN dotnet restore
RUN dotnet build -c Release -o /app/build
RUN dotnet publish -c Release -o /app/publish

# Node.js
RUN npm ci --only=production
RUN npm run build
RUN npm cache clean --force

# Системные пакеты
RUN apt-get update && apt-get install -y curl
RUN apk add --no-cache git
```

### `COPY`

**Копирует файлы и папки в контейнер**

```dockerfile
COPY [--chown=<user>:<group>] <src>... <dest>
```

**Примеры:**

```dockerfile
# .NET - копирование проектов отдельно для кеширования
COPY ["MyApp.csproj", "nuget.config", "./"]
COPY . .

# Node.js - копирование package файлов отдельно
COPY package*.json ./
COPY tsconfig.json ./
COPY src/ ./src/

# С изменением владельца
COPY --chown=node:node . /app
```

### `ADD`

**Копирует файлы с дополнительными возможностями**

```dockerfile
ADD [--chown=<user>:<group>] <src>... <dest>
```

**Особенности:**

- Может распаковывать локальные tar-архивы
- Может скачивать URL-ы (не рекомендуется)

**Примеры:**

```dockerfile
ADD https://example.com/file.tar.gz /tmp/
ADD data.tar.gz /data/
```

## ⚙️ Конфигурация контейнера

### `WORKDIR`

**Задает рабочую директорию**

```dockerfile
WORKDIR /path/to/workdir
```

**Пример:**

```dockerfile
WORKDIR /app
COPY . .          # Копирует в /app
WORKDIR /app/src
RUN pwd           # /app/src
```

### `USER`

**Указывает пользователя для следующих инструкций**

```dockerfile
USER <user>[:<group>]
```

**Примеры:**

```dockerfile
# Создание и использование непривилегированного пользователя
RUN adduser -u 1000 --disabled-password --gecos "" appuser
USER appuser

# Для Node.js
USER node

# По UID/GID
USER 1000:1000
```

### `ARG`

**Задает переменные для передачи во время сборки**

```dockerfile
ARG <name>[=<default value>]
```

**Примеры:**

```dockerfile
ARG RUNTIME_VERSION=7.0
FROM mcr.microsoft.com/dotnet/aspnet:${RUNTIME_VERSION}

ARG NODE_ENV=production
ENV NODE_ENV=${NODE_ENV}

ARG BUILD_NUMBER=1
LABEL build=${BUILD_NUMBER}
```

## 🚀 Инструкции запуска

### `CMD`

**Команда по умолчанию при запуске контейнера**

```dockerfile
CMD ["executable","param1","param2"]  # exec форма (рекомендуется)
CMD command param1 param2             # shell форма
```

**Примеры для разных технологий:**

```dockerfile
# .NET
CMD ["dotnet", "MyApp.dll"]
CMD dotnet MyApp.dll

# Node.js
CMD ["node", "dist/index.js"]
CMD ["npm", "start"]

# Разработка
CMD ["npm", "run", "dev"]
```

### `ENTRYPOINT`

**Настраивает контейнер как исполняемый файл**

```dockerfile
ENTRYPOINT ["executable", "param1", "param2"]
ENTRYPOINT command param1 param2
```

**Взаимодействие с CMD:**

```dockerfile
# .NET с аргументами
ENTRYPOINT ["dotnet", "MyApp.dll"]
CMD ["--urls", "http://0.0.0.0:80"]

# Node.js с обработчиком сигналов
ENTRYPOINT ["node", "dist/index.js"]
```

## 🌐 Сетевые инструкции

### `EXPOSE`

**Информирует о портах, которые прослушивает контейнер**

```dockerfile
EXPOSE <port> [<port>/<protocol>...]
```

**Примеры:**

```dockerfile
EXPOSE 80
EXPOSE 443/tcp
EXPOSE 3000 5000 8080
```

## 💾 Инструкции данных

### `VOLUME`

**Создает точку монтирования**

```dockerfile
VOLUME ["/data"]
VOLUME /var/log /var/db
```

**Пример:**

```dockerfile
VOLUME /var/lib/mysql
VOLUME ["/app/data", "/app/logs"]
```

## 📚 Многостадийные сборки

### Базовый шаблон многостадийной сборки

```dockerfile
# Стадия сборки → Финальный образ
FROM base-image AS builder
# ... сборка
FROM runtime-image
COPY --from=builder /app/build /app
```

## 🎯 Специфичные примеры для технологий

### Dockerfile для .NET 7 приложения

```dockerfile
# Многостадийная сборка для .NET 7
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src

# Копирование файлов проектов отдельно для кеширования
COPY ["MyApp/MyApp.csproj", "MyApp/"]
COPY ["MyApp.Tests/MyApp.Tests.csproj", "MyApp.Tests/"]
RUN dotnet restore "MyApp/MyApp.csproj"

# Копирование исходного кода и сборка
COPY . .
WORKDIR "/src/MyApp"
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish

# Финальный образ
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS final
WORKDIR /app

# Создание непривилегированного пользователя
RUN adduser -u 1000 --disabled-password --gecos "" appuser

# Копирование собранного приложения
COPY --from=publish /app/publish .
RUN chown -R appuser:appuser /app

USER appuser

EXPOSE 80
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Dockerfile для Node.js/TypeScript приложения

```dockerfile
# Стадия сборки
FROM node:18-alpine AS builder

WORKDIR /app

# Копирование package файлов отдельно для кеширования
COPY package*.json ./
COPY tsconfig*.json ./

# Установка зависимостей
RUN npm ci --only=production

# Копирование исходного кода
COPY src/ ./src/
COPY public/ ./public/

# Сборка TypeScript
RUN npm run build

# Финальный образ
FROM node:18-alpine AS runtime

WORKDIR /app

# Создание непривилегированного пользователя
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Копирование собранного приложения и node_modules
COPY --from=builder --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nextjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nextjs:nodejs /app/package.json ./

USER nextjs

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

### Dockerfile для разработки Node.js с Hot Reload

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Копирование package файлов
COPY package*.json ./
COPY tsconfig.json ./

# Установка всех зависимостей (включая dev)
RUN npm install

# Копирование исходного кода
COPY . .

# Установка nodemon для hot reload (глобально или локально)
RUN npm install -g nodemon

EXPOSE 3000

# Команда для разработки с hot reload
CMD ["nodemon", "--exec", "ts-node", "src/index.ts"]
```

### Dockerfile для .NET 7 с миграциями базы данных

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src

COPY ["MyApp.csproj", "./"]
RUN dotnet restore "MyApp.csproj"

COPY . .
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish

# Отдельная стадия для миграций
FROM build AS migrations
RUN dotnet tool install --global dotnet-ef
ENV PATH="$PATH:/root/.dotnet/tools"
RUN dotnet ef migrations bundle -o /app/migrations

FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS final
WORKDIR /app

COPY --from=publish /app/publish .
COPY --from=migrations /app/migrations ./migrations

RUN adduser -u 1000 --disabled-password --gecos "" appuser
USER appuser

EXPOSE 80

# Запуск миграций при старте
CMD ["sh", "-c", "dotnet ef migrations script --idempotent --output /tmp/migrations.sql && psql $$DATABASE_URL -f /tmp/migrations.sql && dotnet MyApp.dll"]
```

### Dockerfile для React/TypeScript приложения

```dockerfile
# Стадия сборки
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
COPY tsconfig.json ./

RUN npm ci

COPY public/ ./public/
COPY src/ ./src/

RUN npm run build

# Финальный образ с nginx
FROM nginx:alpine

# Копирование собранного приложения
COPY --from=builder /app/build /usr/share/nginx/html

# Копирование конфигурации nginx
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

## 🔧 Лучшие практики для .NET и Node.js

### Для .NET 7:

```dockerfile
# Используйте многостадийные сборки
# Копируйте .csproj файлы отдельно для кеширования
# Используйте непривилегированных пользователей
# Настройте правильные переменные окружения
```

### Для Node.js:

```dockerfile
# Используйте alpine образы для уменьшения размера
# Разделяйте зависимости разработки и продакшена
# Используйте npm ci вместо npm install
# Настройте правильные рабочие директории
```

### Универсальные лучшие практики:

```dockerfile
# Минимизируйте количество слоев
# Используйте .dockerignore
# Сортируйте инструкции от редко меняющихся к часто меняющимся
# Используйте конкретные теги образов
# Настройте healthchecks
```

## 📋 Пример .dockerignore

```bash
# .NET
**/bin/
**/obj/
**/.vs/
**/*.user
**/*.lock.json

# Node.js
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.npm
.eslintcache

# Общие
.git/
.gitignore
README.md
Dockerfile*
.dockerignore
.env
.nyc_output
coverage/
.nyc_output
.coverage
```

## 🚀 Быстрые шаблоны

### Быстрый старт .NET 7 API

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=build /app .
EXPOSE 80
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

### Быстрый старт Node.js API

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "src/index.js"]
```

Этот справочник покрывает все основные инструкции Dockerfile с фокусом на .NET 7 и JavaScript/TypeScript приложениях.

---
