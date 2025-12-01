# Docker в продакшене

Промышленное развертывание Docker-контейнеров с фокусом на надежность, безопасность и масштабируемость для .NET 7 и JavaScript/TypeScript приложений.

## 🎯 Задача

Обеспечить production-готовую Docker-инфраструктуру с:

- Высокой доступностью и отказоустойчивостью
- Автоматическим масштабированием
- Безопасной конфигурацией
- Мониторингом и логированием
- CI/CD интеграцией

## 📁 Структура production проекта

```bash
production-project/
├── docker-compose.prod.yml          # Production stack
├── docker-stack.yml                 # Swarm deployment
├── .env.production                  # Production variables
├── backend/
│   ├── Dockerfile.prod              # Optimized .NET 7 image
│   ├── appsettings.Production.json
│   └── nlog.config                  # Structured logging
├── frontend/
│   ├── Dockerfile.prod              # Optimized Node.js image
│   └── nginx.conf                   # Production nginx config
├── monitoring/
│   ├── prometheus.yml
│   ├── alertmanager.yml
│   └── grafana/dashboards/
├── security/
│   ├── ssl/
│   └── security-scans/
└── ci-cd/
    ├── github-actions/
    └── gitlab-ci.yml
```

## 🐳 Production Docker конфигурация

### Dockerfile для .NET 7 (Production)

```dockerfile
# Multi-stage build для .NET 7
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src

# Копирование и восстановление зависимостей
COPY *.sln .
COPY backend/*.csproj ./backend/
RUN dotnet restore

# Сборка приложения
COPY backend/ ./backend/
WORKDIR /src/backend
RUN dotnet publish -c Release -o /app/publish \
    --runtime linux-x64 \
    --self-contained false \
    --no-restore

# Final stage
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS final
WORKDIR /app

# Создание непривилегированного пользователя
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Установка безопасности
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Копирование приложения
COPY --from=build --chown=appuser:appuser /app/publish .
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Dockerfile для Node.js (Production)

```dockerfile
# Multi-stage build для Node.js
FROM node:18-alpine AS builder
WORKDIR /app

# Установка зависимостей
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Final stage
FROM node:18-alpine AS final
RUN apk add --no-cache curl

# Создание непривилегированного пользователя
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

WORKDIR /app

# Копирование приложения
COPY --from=builder --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --chown=nextjs:nodejs . .

USER nextjs

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/api/health || exit 1

EXPOSE 3000
CMD ["node", "server.js"]
```

### docker-compose.prod.yml

```yaml
version: "3.8"

services:
  backend:
    image: ${REGISTRY:-myregistry}/backend:${TAG:-latest}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 5s
        order: stop-first
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 3
        window: 120s
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
        reservations:
          memory: 256M
          cpus: "0.5"
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=${DB_CONNECTION_STRING}
    configs:
      - source: appsettings_production
        target: /app/appsettings.Production.json
    secrets:
      - jwt_secret
    networks:
      - backend-network
      - monitoring-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.backend.rule=Host(`api.example.com`)"
      - "prometheus.scrape=true"
      - "prometheus.port=8080"
      - "prometheus.path=/metrics"

  frontend:
    image: ${REGISTRY:-myregistry}/frontend:${TAG:-latest}
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 256M
          cpus: "0.5"
    ports:
      - "3000:3000"
    networks:
      - frontend-network
      - monitoring-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.frontend.rule=Host(`app.example.com`)"

  database:
    image: postgres:15-alpine
    deploy:
      placement:
        constraints:
          - node.role == manager
      resources:
        limits:
          memory: 1G
          cpus: "1.0"
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/backups:/backups
    networks:
      - backend-network
    labels:
      - "backup.schedule=daily"

  redis:
    image: redis:7-alpine
    deploy:
      replicas: 1
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - backend-network

  # Reverse proxy
  traefik:
    image: traefik:v2.10
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./security/ssl:/etc/ssl
    deploy:
      placement:
        constraints:
          - node.role == manager
    command:
      - --api.dashboard=true
      - --providers.docker=true
      - --providers.docker.swarmmode=true
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.letsencrypt.acme.email=admin@example.com
      - --certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web
    networks:
      - frontend-network
      - backend-network

  # Monitoring stack
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - prometheus_data:/prometheus
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    deploy:
      placement:
        constraints:
          - node.role == manager
    networks:
      - monitoring-network

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
    deploy:
      placement:
        constraints:
          - node.role == manager
    networks:
      - monitoring-network

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./monitoring/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    networks:
      - monitoring-network

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  prometheus_data:
    driver: local
  grafana_data:
    driver: local

configs:
  appsettings_production:
    file: ./backend/appsettings.Production.json

secrets:
  db_password:
    external: true
  jwt_secret:
    external: true

networks:
  frontend-network:
    driver: overlay
  backend-network:
    driver: overlay
  monitoring-network:
    driver: overlay
```

## 🚀 Развертывание в Swarm

### docker-stack.yml

```yaml
version: "3.8"

services:
  backend:
    image: myregistry/backend:${TAG}
    deploy:
      mode: replicated
      replicas: 3
      placement:
        constraints:
          - node.labels.environment == production
      update_config:
        parallelism: 1
        delay: 30s
      rollback_config:
        parallelism: 1
        delay: 10s
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
    networks:
      - prod-network

  # ... другие сервисы аналогично

networks:
  prod-network:
    driver: overlay
    attachable: true
```

### Команды развертывания

```bash
# Инициализация Swarm
docker swarm init

# Создание секретов
echo "supersecretdbpassword" | docker secret create db_password -
echo "jwtsecretkey" | docker secret create jwt_secret -

# Развертывание stack
docker stack deploy -c docker-stack.yml myapp

# Мониторинг развертывания
docker stack ps myapp
docker service ls
docker service logs myapp_backend -f
```

## 🔒 Безопасность в production

### Security-ориентированные настройки

```yaml
# security-config.yml
services:
  backend:
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid
    labels:
      - "com.example.security.scan=true"
```

### SSL конфигурация

```bash
# Генерация SSL сертификатов
docker run -it --rm -v ssl_data:/ssl -v /var/run/docker.sock:/var/run/docker.sock \
  traefik cert generate --domain=*.example.com
```

## 📊 Мониторинг и метрики

### Prometheus конфигурация

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: "backend"
    static_configs:
      - targets: ["backend:8080"]
    metrics_path: "/metrics"
    scrape_interval: 10s

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]

  - job_name: "docker"
    static_configs:
      - targets: ["docker-exporter:9323"]

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
```

### Alert rules

```yaml
# monitoring/alert_rules.yml
groups:
  - name: backend
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on backend"
          description: "Error rate is {{ $value }} per second"

      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
```

## 🔧 Настройка для разных окружений

### Production-specific переменные

```bash
# .env.production
REGISTRY=myregistry.azurecr.io
TAG=1.0.0
DB_CONNECTION_STRING=Server=database;Database=prod;User Id=produser;Password=...
REDIS_PASSWORD=secure_redis_password
GRAFANA_PASSWORD=secure_grafana_password
```

## 🛠️ CI/CD интеграция

### GitHub Actions workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    tags: ["v*"]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ./backend/Dockerfile.prod
          push: true
          tags: ${{ secrets.REGISTRY }}/backend:${{ github.ref_name }}

      - name: Deploy to Swarm
        run: |
          echo "${{ secrets.SSH_KEY }}" > key.pem
          chmod 600 key.pem
          ssh -i key.pem ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }} \
            "docker stack deploy -c docker-stack.yml myapp"
```

## 📈 Масштабирование и оптимизация

### Автоскейлинг на основе метрик

```bash
# Автоматическое масштабирование
docker service update --replicas 5 myapp_backend

# На основе CPU
docker service update \
  --replicas $(docker service inspect myapp_backend --format '{{.Spec.Mode.Replicated.Replicas}}') \
  --update-parallelism 2 \
  myapp_backend
```

### Resource ограничения

```yaml
resources:
  limits:
    memory: 1G
    cpus: "2.0"
  reservations:
    memory: 512M
    cpus: "0.5"
```

## 🛠️ Troubleshooting в production

### Экстренные команды

```bash
# Экстренное масштабирование
docker service scale myapp_backend=10

# Быстрый rollback
docker service rollback myapp_backend

# Аварийное обновление
docker service update --image myregistry/backend:emergency-fix myapp_backend

# Диагностика сети
docker network inspect myapp_prod-network
docker service logs myapp_backend --tail 100 -f

# Мониторинг ресурсов
docker stats $(docker ps -q)
```

### Disaster recovery

```bash
# Backup базы данных
docker exec $(docker ps -q -f name=myapp_database) \
  pg_dump -U $DB_USER $DB_NAME > backup.sql

# Восстановление из backup
docker exec -i $(docker ps -q -f name=myapp_database) \
  psql -U $DB_USER $DB_NAME < backup.sql
```

## 💡 Production Best Practices

### Безопасность

- Используйте только официальные образы
- Регулярно обновляйте базовые образы
- Сканируйте образы на уязвимости (Trivy, Dockle)
- Используйте secrets для конфиденциальных данных

### Надежность

- Настройте health checks для всех сервисов
- Используйте restart policies
- Реализуйте circuit breaker паттерны
- Настройте мониторинг и алертинг

### Производительность

- Оптимизируйте размер образов (multi-stage builds)
- Настройте кэширование на всех уровнях
- Используйте CDN для статики
- Оптимизируйте настройки базы данных

### Операционная готовность

- Автоматизируйте развертывание
- Вести centralized логирование
- Регулярно тестируйте backup/restore
- Документируйте процедуры аварийного восстановления

**Ключевой вывод**: Production-развертывание Docker требует комплексного подхода, сочетающего безопасность, надежность и наблюдаемость. Правильно настроенная инфраструктура обеспечивает высокую доступность приложений и упрощает операционное управление.
