Отлично! Создам файл **0 - Docker Security Best Practices.md** в точном соответствии с оформлением из примера про слои:

---

# Docker Security Best Practices: основные принципы и инструменты

Безопасность Docker-окружения начинается с соблюдения базовых принципов и использования специализированных инструментов для автоматического сканирования и аудита.

## Что такое безопасность Docker?

**Безопасность Docker** — это комплекс мер, направленных на защиту контейнеров, образов и всей containerized инфраструктуры от угроз и уязвимостей.

## Ключевые принципы безопасности

### 1. 🚫 Принцип наименьших привилегий

```dockerfile
# Запуск от непривилегированного пользователя
FROM mcr.microsoft.com/dotnet/aspnet:7.0
RUN useradd -m appuser
USER appuser
COPY --chown=appuser:appuser . /app
```

### 2. 🔒 Минимальная поверхность атаки

```dockerfile
# Многостадийная сборка для .NET 7
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=build /app .
USER 1000
```

### 3. 📦 Безопасные базовые образы

- Используйте официальные образы с регулярными обновлениями
- Предпочитайте минимальные дистрибутивы (Alpine, distroless)
- Регулярно обновляйте базовые образы

## Инструменты автоматического сканирования

### 🔍 Trivy — сканирование на уязвимости (CVE)

```bash
# Базовая проверка образа
trivy image your-app:latest

# Проверка с выходным кодом для CI/CD
trivy image --exit-code 1 --severity CRITICAL,HIGH your-app:latest

# Проверка только фиксированных уязвимостей
trivy image --ignore-unfixed your-app:latest
```

**Что проверяет Trivy:**

- Уязвимости в установленных пакетах
- Конфигурационные файлы на безопасность
- Секреты в коде и переменных окружения

### 📋 Dockle — проверка на best practices

```bash
# Базовая проверка
docker run --rm goodwithtech/dockle:v0.4.14 your-app:latest

# Проверка с JSON выводом
docker run --rm goodwithtech/dockle:v0.4.14 -f json your-app:latest

# Тестовый образ с замечаниями
docker run --rm goodwithtech/dockle:v0.4.14 goodwithtech/dockle-test:v2
```

**Что проверяет Dockle:**

- Соответствие CIS Docker Benchmark
- Соблюдение Dockerfile best practices
- Настройки безопасности runtime

## Практические примеры для вашего стека

### .NET 7 приложения

```dockerfile
# Безопасный Dockerfile для .NET 7
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:7.0
RUN adduser --disabled-password --gecos '' appuser
WORKDIR /app
COPY --from=build --chown=appuser:appuser /app .
USER appuser
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:8080/health || exit 1
```

### JavaScript/TypeScript приложения

```dockerfile
# Безопасный Dockerfile для Node.js
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001
WORKDIR /app
COPY --from=build --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --chown=nextjs:nodejs . .
USER nextjs
EXPOSE 3000
CMD ["npm", "start"]
```

## Интеграция в CI/CD пайплайны

### GitHub Actions пример

```yaml
name: Security Scan
on: [push]
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Build image
        run: docker build -t my-app .

      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "my-app"
          format: "sarif"
          output: "trivy-results.sarif"

      - name: Scan with Dockle
        run: |
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
          goodwithtech/dockle:v0.4.14 my-app
```

## Ограничение ресурсов и capabilities

```yaml
# docker-compose.yml
version: "3.8"
services:
  app:
    image: my-app:latest
    user: "1000:1000"
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
```

## Мониторинг и аудит

```bash
# Проверка запущенных контейнеров
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.RunningFor}}"

# Аудит безопасности
docker bench-security

# Проверка логов
docker logs [container_id]
```

**Ключевой вывод**: Безопасность Docker требует комплексного подхода — от правильного написания Dockerfile до автоматического сканирования в CI/CD. Инструменты Trivy и Dockle позволяют автоматизировать проверки и предотвращать развертывание уязвимых образов в production.

---
