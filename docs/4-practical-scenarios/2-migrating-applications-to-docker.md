# Миграция приложений в Docker

Практическое руководство по переносу legacy и монолитных приложений в Docker-контейнеры с минимальными изменениями кода и максимальной эффективностью.

## 🎯 Задача

Обеспечить плавную миграцию существующих приложений в Docker-контейнеры:

- Минимизация изменений в коде
- Сохранение существующей логики приложений
- Обеспечение обратной совместимости
- Постепенный переход на containerized инфраструктуру

## 📁 Стратегия миграции

### Этапы миграции

```
1. Анализ и подготовка
   ├── Аудит приложения
   ├── Выявление зависимостей
   └── Подготовка Dockerfile

2. Контейнеризация
   ├── Создание базового образа
   ├── Настройка конфигурации
   └── Тестирование в изоляции

3. Интеграция
   ├── Docker Compose настройка
   ├── Миграция данных
   └── CI/CD адаптация

4. Продакшен
   ├── Постепенный rollout
   ├── Мониторинг
   └── Оптимизация
```

## 🐳 Миграция .NET приложений

### Legacy ASP.NET (Framework 4.x) → Docker

```dockerfile
# Dockerfile для ASP.NET Framework
FROM mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2019

# Установка дополнительных компонентов
RUN powershell -Command \
    Add-WindowsFeature Web-ASP-Net45; \
    Add-WindowsFeature Web-Asp-Net45

# Копирование приложения
COPY ./publish/ /inetpub/wwwroot/

# Настройка IIS
RUN powershell -Command \
    Remove-Website -Name 'Default Web Site'; \
    New-Website -Name 'MyApp' -Port 80 -PhysicalPath 'C:\inetpub\wwwroot' -ApplicationPool '.NET v4.5'

EXPOSE 80
```

### ASP.NET Core Migration с поддержкой legacy

```dockerfile
# Multi-stage для mixed environment
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS migration-build
WORKDIR /src

# Копирование legacy конфигов
COPY legacy-app/web.config .
COPY legacy-app/Global.asax .

# Modern .NET Core app
COPY modern-app/*.csproj ./modern-app/
RUN dotnet restore modern-app/*.csproj

COPY modern-app/ ./modern-app/
RUN dotnet publish modern-app -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS final
WORKDIR /app

# Совместимость с legacy путями
RUN mkdir -p /inetpub/wwwroot
COPY --from=migration-build /app/publish ./modern
COPY legacy-assets/ ./legacy/

# Adapter для legacy routes
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost/health || exit 1

EXPOSE 80
ENTRYPOINT ["dotnet", "modern/MyApp.dll"]
```

### Конфигурация для mixed environment

```yaml
# docker-compose.migration.yml
version: "3.8"

services:
  legacy-adapter:
    build:
      context: .
      dockerfile: Dockerfile.legacy-adapter
    ports:
      - "8080:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Migration
      - LEGACY_API_URL=http://old-system-api
      - MODERN_API_URL=http://new-backend
    volumes:
      - legacy-data:/legacy/data
    networks:
      - migration-network

  new-backend:
    build: ./modern-backend
    environment:
      - ConnectionStrings__LegacyDatabase=${LEGACY_DB_CONNECTION}
      - ConnectionStrings__ModernDatabase=${MODERN_DB_CONNECTION}
    networks:
      - migration-network

  migration-proxy:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx-migration.conf:/etc/nginx/nginx.conf
    depends_on:
      - legacy-adapter
      - new-backend
    networks:
      - migration-network

volumes:
  legacy-data:

networks:
  migration-network:
    driver: bridge
```

## 🔄 Миграция Node.js приложений

### Legacy Node.js + Express Migration

```dockerfile
# Dockerfile для legacy Node.js app
FROM node:14-alpine AS legacy-base

# Установка legacy зависимостей
WORKDIR /app
COPY package-old.json ./package.json
RUN npm install --production --legacy-peer-deps

# Копирование legacy кода
COPY legacy-src/ ./src/
COPY old-config/ ./config/

# Modern wrapper
FROM node:18-alpine AS modern-wrapper
WORKDIR /app

# Legacy app
COPY --from=legacy-base /app ./legacy

# Modern adapter
COPY modern-src/ ./modern/
COPY package.json ./
RUN npm install

# Совместимость версий
ENV NODE_ENV=production
ENV LEGACY_APP_PORT=3001
ENV MODERN_APP_PORT=3000

EXPOSE 3000
CMD ["node", "modern/server.js"]
```

### Adapter для legacy API

```javascript
// legacy-adapter.js
const express = require("express");
const { createProxyMiddleware } = require("http-proxy-middleware");
const legacyApp = require("./legacy/app"); // Старое приложение

const app = express();

// Прокси для legacy routes
app.use(
  "/old-api",
  createProxyMiddleware({
    target: `http://localhost:${process.env.LEGACY_APP_PORT}`,
    changeOrigin: true,
    pathRewrite: {
      "^/old-api": "/api",
    },
  })
);

// Новые endpoints
app.use("/api", require("./modern/routes"));

// Health check для migration monitoring
app.get("/migration-health", (req, res) => {
  const legacyHealth = checkLegacyHealth();
  const modernHealth = checkModernHealth();

  res.json({
    status: legacyHealth && modernHealth ? "healthy" : "degraded",
    legacy: legacyHealth,
    modern: modernHealth,
    migration_progress: calculateMigrationProgress(),
  });
});

// Запуск legacy app в background
legacyApp.listen(process.env.LEGACY_APP_PORT, () => {
  console.log(`Legacy app running on port ${process.env.LEGACY_APP_PORT}`);
});

// Запуск modern app
app.listen(process.env.MODERN_APP_PORT, () => {
  console.log(
    `Migration adapter running on port ${process.env.MODERN_APP_PORT}`
  );
});
```

## 🗄️ Миграция баз данных

### Постепенная миграция данных

```yaml
# docker-compose.database-migration.yml
version: "3.8"

services:
  legacy-database:
    image: postgres:12-alpine
    environment:
      - POSTGRES_DB=legacy_app
      - POSTGRES_USER=legacy_user
      - POSTGRES_PASSWORD=legacy_pass
    volumes:
      - legacy_db_data:/var/lib/postgresql/data
      - ./migration-scripts/legacy-init.sql:/docker-entrypoint-initdb.d/legacy-init.sql
    networks:
      - migration-network

  modern-database:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=modern_app
      - POSTGRES_USER=modern_user
      - POSTGRES_PASSWORD=modern_pass
    volumes:
      - modern_db_data:/var/lib/postgresql/data
    networks:
      - migration-network

  migration-service:
    build: ./database-migration
    environment:
      - SOURCE_DB_URL=postgresql://legacy_user:legacy_pass@legacy-database:5432/legacy_app
      - TARGET_DB_URL=postgresql://modern_user:modern_pass@modern-database:5432/modern_app
      - MIGRATION_BATCH_SIZE=1000
    depends_on:
      - legacy-database
      - modern-database
    networks:
      - migration-network

  dual-write-proxy:
    image: ghcr.io/migration-proxy:latest
    environment:
      - READ_SOURCE=legacy-database
      - WRITE_TARGETS=legacy-database,modern-database
      - MIGRATION_PHASE=dual_write
    depends_on:
      - legacy-database
      - modern-database
    networks:
      - migration-network

volumes:
  legacy_db_data:
  modern_db_data:

networks:
  migration-network:
    driver: bridge
```

### Database migration service

```javascript
// database-migration/service.js
class DatabaseMigration {
  constructor(sourceConfig, targetConfig) {
    this.sourcePool = new Pool(sourceConfig);
    this.targetPool = new Pool(targetConfig);
    this.migrationState = new MigrationState();
  }

  async startIncrementalMigration() {
    while (this.migrationState.getProgress() < 100) {
      const batch = await this.extractBatch();
      await this.transformAndLoad(batch);
      await this.updateMigrationState(batch);

      // Dual-write для новых данных
      await this.enableDualWrite();
    }

    await this.switchToModern();
  }

  async enableDualWrite() {
    // Прокси для записи в обе БД
    app.post("/api/data", async (req, res) => {
      try {
        await Promise.all([
          this.sourcePool.query("INSERT INTO legacy_table ...", req.body),
          this.targetPool.query("INSERT INTO modern_table ...", req.body),
        ]);
        res.status(200).json({ success: true });
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });
  }
}
```

## 🔧 Миграция конфигурации

### Конфигурационный адаптер

```yaml
# config/migration-config.yml
services:
  config-transformer:
    image: config-migration:latest
    environment:
      - LEGACY_CONFIG_PATH=/config/old-app.config
      - MODERN_CONFIG_PATH=/config/new-app.json
      - TRANSFORMATION_RULES=/rules/legacy-to-modern.yaml
    volumes:
      - ./legacy-config:/config/old-app.config
      - ./modern-config:/config/new-app.json
      - ./transformation-rules:/rules
    networks:
      - migration-network

  config-server:
    image: consul:latest
    ports:
      - "8500:8500"
    volumes:
      - ./consul-config:/consul/config
    networks:
      - migration-network
```

### Transformation rules

```yaml
# transformation-rules/legacy-to-modern.yaml
database:
  legacy: "Data Source=server;Initial Catalog=db"
  modern:
    connectionString: "Server=server;Database=db"
    provider: "SqlServer"

logging:
  legacy: "LogLevel=Debug;LogFile=/logs/app.log"
  modern:
    level: "Debug"
    outputs:
      - type: "file"
        path: "/logs/app.log"
      - type: "console"

features:
  legacy: "FeatureA=true;FeatureB=false"
  modern:
    features:
      FeatureA: true
      FeatureB: false
```

## 🚀 Постепенный rollout

### Blue-Green миграция

```yaml
# docker-compose.blue-green.yml
version: "3.8"

services:
  blue-app:
    image: myapp:blue
    ports:
      - "8080:80"
    environment:
      - DEPLOYMENT_COLOR=blue
    networks:
      - migration-network

  green-app:
    image: myapp:green
    ports:
      - "8081:80"
    environment:
      - DEPLOYMENT_COLOR=green
    networks:
      - migration-network

  migration-router:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./blue-green-router.conf:/etc/nginx/nginx.conf
    environment:
      - ACTIVE_COLOR=blue
      - TRAFFIC_PERCENTAGE=10
    depends_on:
      - blue-app
      - green-app
    networks:
      - migration-network

networks:
  migration-network:
    driver: bridge
```

### Traffic management configuration

```nginx
# blue-green-router.conf
http {
    upstream blue {
        server blue-app:80;
    }

    upstream green {
        server green-app:80;
    }

    split_clients $request_id $deployment {
        10% green;
        90% blue;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://$deployment;
            proxy_set_header X-Deployment-Color $deployment;
        }

        # Health checks для мониторинга миграции
        location /migration-status {
            access_log off;
            add_header Content-Type application/json;

            return 200 '{
                "active": "${ACTIVE_COLOR}",
                "traffic_split": "${TRAFFIC_PERCENTAGE}",
                "blue_health": "$$(check_health blue)",
                "green_health": "$$(check_health green)"
            }';
        }
    }
}
```

## 📊 Мониторинг миграции

### Миграционные метрики

```yaml
# monitoring/migration-dashboard.yml
services:
  migration-monitor:
    image: custom-migration-monitor:latest
    environment:
      - METRICS_ENDPOINT=http://prometheus:9090
      - ALERT_THRESHOLD=0.95
    volumes:
      - ./migration-metrics.json:/etc/monitoring/metrics.json
    networks:
      - monitoring-network

  migration-dashboard:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    volumes:
      - ./migration-dashboards:/var/lib/grafana/dashboards
    networks:
      - monitoring-network
```

### Key metrics для миграции

```json
{
  "migration_metrics": {
    "legacy_traffic_percentage": "gauge",
    "modern_traffic_percentage": "gauge",
    "data_sync_lag_seconds": "gauge",
    "error_rate_comparison": "gauge",
    "feature_adoption_rate": "counter",
    "rollback_requests": "counter"
  }
}
```

## 🛠️ Troubleshooting миграции

### Common issues and solutions

**Проблема**: Legacy dependencies not available in containers  
**Решение**:

```dockerfile
# Custom base image with legacy dependencies
FROM ubuntu:18.04 AS legacy-base
RUN apt-get update && apt-get install -y \
    libssl1.0.0 \
    legacy-package=1.2.3
```

**Проблема**: File paths different in containers  
**Решение**:

```dockerfile
# Symlink legacy paths
RUN ln -s /app /inetpub/wwwroot && \
    ln -s /app/logs /var/log/legacy-app
```

**Проблема**: Windows services in Linux containers  
**Решение**:

```dockerfile
# Service wrapper
FROM mcr.microsoft.com/dotnet/runtime:7.0
COPY windows-service-wrapper.sh /
RUN chmod +x /windows-service-wrapper.sh
CMD ["/windows-service-wrapper.sh"]
```

### Rollback procedures

```bash
#!/bin/bash
# emergency-rollback.sh

# Быстрый rollback к legacy системе
docker-compose -f docker-compose.legacy.yml up -d

# Переключение трафика
curl -X POST http://router/switch-traffic -d '{"target":"legacy"}'

# Оповещение команды
send_alert "Migration rolled back - check logs"
```

## 💡 Best Practices миграции

### Стратегические рекомендации

- **Начинайте с non-critical services** - снижает риски
- **Используйте feature flags** - для контроля rollout
- **Подготовьте rollback plan** - на каждый этап
- **Мониторьте производительность** - сравнивайте до/после

### Технические рекомендации

- **Сохраняйте backward compatibility** - минимум 2 версии
- **Используйте миграционные прокси** - для плавного перехода
- **Тестируйте в staging** - полная копия production
- **Документируйте процесс** - для повторного использования

### Операционные рекомендации

- **Планируйте maintenance windows** - для критичных изменений
- **Коммуницируйте с stakeholders** - о статусе миграции
- **Проводите post-mortem** - после каждого этапа
- **Измеряйте успешность** - по бизнес-метрикам

**Ключевой вывод**: Успешная миграция в Docker требует тщательного планирования, постепенного подхода и надежных механизмов отката. Фокус на обратной совместимости и минимальных изменениях кода обеспечивает плавный переход без disruption для пользователей.
