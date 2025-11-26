# ASP.NET Core в Docker

Полное руководство по контейнеризации ASP.NET Core приложений с оптимизациями для production, health checks и мониторингом.

## 🎯 Задача

Создать production-готовый Docker образ для ASP.NET Core приложения с:

- Многостадийной сборкой для минимального размера
- Health checks и мониторингом
- Безопасной конфигурацией
- Оптимизацией производительности

## 📁 Структура проекта

```
aspnet-core-app/
├── Dockerfile                    # Production Dockerfile
├── Dockerfile.dev               # Development Dockerfile
├── docker-compose.yml           # Локальная разработка
├── docker-compose.prod.yml      # Production stack
├── src/
│   ├── MyApp.Api/              # Web API проект
│   │   ├── Controllers/
│   │   ├── Program.cs
│   │   ├── appsettings.json
│   │   └── MyApp.Api.csproj
│   └── MyApp.Domain/           # Domain layer
├── tests/
│   └── MyApp.Api.Tests/
└── config/
    ├── nlog.config             # Structured logging
    └── prometheus.yml          # Metrics config
```

## 🐳 Docker конфигурация

### Dockerfile (Production)

```dockerfile
# Multi-stage build для ASP.NET Core
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src

# Копирование и восстановление зависимостей
COPY ["src/MyApp.Api/MyApp.Api.csproj", "src/MyApp.Api/"]
COPY ["src/MyApp.Domain/MyApp.Domain.csproj", "src/MyApp.Domain/"]
RUN dotnet restore "src/MyApp.Api/MyApp.Api.csproj"

# Копирование исходного кода и сборка
COPY . .
WORKDIR "/src/src/MyApp.Api"
RUN dotnet build "MyApp.Api.csproj" -c Release -o /app/build

# Публикация приложения
FROM build AS publish
RUN dotnet publish "MyApp.Api.csproj" -c Release -o /app/publish \
    --runtime linux-x64 \
    --self-contained false \
    --no-restore

# Final stage
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS final
WORKDIR /app

# Создание непривилегированного пользователя
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Установка curl для health checks
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Копирование приложения
COPY --from=publish --chown=appuser:appuser /app/publish .
USER appuser

# Настройка health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

### Dockerfile (Development)

```dockerfile
# Development образ с hot reload
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS development

# Установка инструментов для отладки
RUN apt-get update && apt-get install -y \
    curl \
    procps \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Копирование проектов
COPY *.sln .
COPY src/MyApp.Api/*.csproj ./src/MyApp.Api/
COPY src/MyApp.Domain/*.csproj ./src/MyApp.Domain/
RUN dotnet restore

# Настройка для разработки
ENV ASPNETCORE_ENVIRONMENT=Development
ENV DOTNET_USE_POLLING_FILE_WATCHER=1
ENV DOTNET_CLI_TELEMETRY_OPTOUT=1

CMD ["dotnet", "watch", "run", "--urls", "http://0.0.0.0:5000"]
```

## 🚀 Запуск и использование

### Базовый запуск

```bash
# Сборка образа
docker build -t myapp-api:latest .

# Запуск контейнера
docker run -d -p 8080:8080 \
  -e ASPNETCORE_ENVIRONMENT=Production \
  -e ConnectionStrings__DefaultConnection="Server=db;Database=myapp;User=sa;Password=..." \
  myapp-api:latest

# Просмотр логов
docker logs -f <container_id>

# Health check
curl http://localhost:8080/health
```

### Docker Compose для разработки

```yaml
# docker-compose.yml
version: "3.8"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "5000:5000"
      - "5001:5001"
    volumes:
      - .:/app
      - /app/src/MyApp.Api/bin
      - /app/src/MyApp.Api/obj
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Host=db;Database=myapp_dev;Username=postgres;Password=postgres
    depends_on:
      - db
    networks:
      - app-network

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=myapp_dev
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - app-network

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

### Production Docker Compose

```yaml
# docker-compose.prod.yml
version: "3.8"

services:
  api:
    image: myregistry/myapp-api:${TAG:-latest}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: any
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
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
      - prod-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.example.com`)"
      - "prometheus.scrape=true"
      - "prometheus.port=8080"
      - "prometheus.path=/metrics"

  db:
    image: postgres:15-alpine
    deploy:
      placement:
        constraints: [node.role == manager]
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - prod-network

configs:
  appsettings_production:
    file: ./src/MyApp.Api/appsettings.Production.json

secrets:
  db_password:
    external: true
  jwt_secret:
    external: true

volumes:
  postgres_data:

networks:
  prod-network:
    driver: overlay
```

## ⚙️ Конфигурационные файлы

### appsettings.Production.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore.Database.Command": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": "Server=db;Database=myapp;User Id=appuser;Password=..."
  },
  "Jwt": {
    "SecretKey": "",
    "Issuer": "myapp-api",
    "Audience": "myapp-users"
  },
  "HealthChecks": {
    "Enabled": true,
    "Endpoint": "/health"
  },
  "Metrics": {
    "Enabled": true,
    "Endpoint": "/metrics"
  }
}
```

### nlog.config для structured logging

```xml
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <targets>
    <target name="jsonFile" xsi:type="File" fileName="/var/log/myapp/api.log">
      <layout xsi:type="JsonLayout">
        <attribute name="timestamp" layout="${date:format=yyyy-MM-ddTHH\:mm\:ss.fffZ}" />
        <attribute name="level" layout="${level:upperCase=true}" />
        <attribute name="logger" layout="${logger}" />
        <attribute name="message" layout="${message}" />
        <attribute name="exception" layout="${exception:format=toString}" />
        <attribute name="properties" encode="false">
          <layout type="JsonLayout" includeAllProperties="true" />
        </attribute>
      </layout>
    </target>

    <target name="console" xsi:type="Console">
      <layout xsi:type="JsonLayout" includeAllProperties="true">
        <attribute name="timestamp" layout="${date:format=yyyy-MM-ddTHH\:mm\:ss.fffZ}" />
        <attribute name="level" layout="${level:upperCase=true}" />
        <attribute name="message" layout="${message}" />
      </layout>
    </target>
  </targets>

  <rules>
    <logger name="*" minlevel="Info" writeTo="jsonFile,console" />
  </rules>
</nlog>
```

## 🔧 Программная конфигурация

### Program.cs с Docker оптимизациями

```csharp
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Prometheus;

var builder = WebApplication.CreateBuilder(args);

// Конфигурация для Docker
builder.Configuration
    .AddJsonFile("appsettings.json", optional: false)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true)
    .AddEnvironmentVariables();

// Health Checks
builder.Services.AddHealthChecks()
    .AddNpgSql(builder.Configuration.GetConnectionString("DefaultConnection"))
    .AddRedis(builder.Configuration.GetConnectionString("Redis"))
    .ForwardToPrometheus();

// Metrics
builder.Services.AddMetrics();

// Logging
builder.Services.AddLogging(logging =>
{
    logging.ClearProviders();
    logging.AddConsole();
    logging.AddNLog();
});

// API Controllers
builder.Services.AddControllers();

// Swagger для development
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddEndpointsApiExplorer();
    builder.Services.AddSwaggerGen();
}

var app = builder.Build();

// Конфигурация middleware pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseRouting();

// Metrics endpoint
app.UseHttpMetrics();
app.MapMetrics();

// Health checks
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        var result = new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                duration = e.Value.Duration.TotalMilliseconds
            })
        };
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsJsonAsync(result);
    }
});

// Ready check для orchestration
app.MapGet("/ready", () => Results.Ok(new { status = "ready" }));

// API endpoints
app.MapControllers();

app.Run();
```

## 📊 Мониторинг и метрики

### Prometheus конфигурация

```yaml
# config/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "aspnet-core-api"
    static_configs:
      - targets: ["api:8080"]
    metrics_path: "/metrics"
    scrape_interval: 10s
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: "(.*):.*"
        replacement: "${1}"

  - job_name: "health-checks"
    static_configs:
      - targets: ["api:8080"]
    metrics_path: "/health"
    scrape_interval: 30s
```

### Health Checks Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class HealthController : ControllerBase
{
    private readonly ILogger<HealthController> _logger;
    private readonly ApplicationDbContext _dbContext;

    public HealthController(ILogger<HealthController> logger, ApplicationDbContext dbContext)
    {
        _logger = logger;
        _dbContext = dbContext;
    }

    [HttpGet("detailed")]
    public async Task<IActionResult> GetDetailedHealth()
    {
        var healthReport = new
        {
            status = "Healthy",
            timestamp = DateTime.UtcNow,
            checks = new[]
            {
                new { name = "database", status = await CheckDatabase() },
                new { name = "memory", status = CheckMemory() },
                new { name = "disk", status = CheckDisk() }
            }
        };

        _logger.LogInformation("Health check executed at {Timestamp}", DateTime.UtcNow);

        return Ok(healthReport);
    }

    private async Task<string> CheckDatabase()
    {
        try
        {
            await _dbContext.Database.ExecuteSqlRawAsync("SELECT 1");
            return "Healthy";
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Database health check failed");
            return "Unhealthy";
        }
    }

    private string CheckMemory()
    {
        var process = Process.GetCurrentProcess();
        var memoryMB = process.WorkingSet64 / 1024 / 1024;
        return memoryMB < 500 ? "Healthy" : "Degraded";
    }

    private string CheckDisk()
    {
        var drive = new DriveInfo("/app");
        var freePercentage = (double)drive.AvailableFreeSpace / drive.TotalSize * 100;
        return freePercentage > 10 ? "Healthy" : "Degraded";
    }
}
```

## 🔧 Настройка для разных окружений

### Development-specific настройки

```csharp
// Development конфигурация
if (builder.Environment.IsDevelopment())
{
    // Подробные ошибки
    builder.Services.AddDatabaseDeveloperPageExceptionFilter();

    // CORS для development
    builder.Services.AddCors(options =>
    {
        options.AddPolicy("Development", policy =>
        {
            policy.AllowAnyOrigin()
                  .AllowAnyMethod()
                  .AllowAnyHeader();
        });
    });
}
```

### Production-specific настройки

```csharp
// Production конфигурация
if (builder.Environment.IsProduction())
{
    // Security headers
    app.UseHsts();
    app.UseHttpsRedirection();

    // Rate limiting
    app.UseRateLimiting();

    // Production CORS
    app.UseCors("Production");
}
```

## 🛠️ Troubleshooting

### Частые проблемы и решения

**Проблема**: Миграции EF Core не применяются  
**Решение**:

```dockerfile
# Добавить в Dockerfile
RUN dotnet ef database update
```

Или при запуске:

```bash
docker exec <container_id> dotnet ef database update
```

**Проблема**: High memory usage  
**Решение**:

```yaml
# В docker-compose.prod.yml
resources:
  limits:
    memory: 512M
    cpus: "1.0"
```

**Проблема**: Slow startup in containers  
**Решение**:

```dockerfile
# Использовать ready-to-run компиляцию
RUN dotnet publish -c Release -o /app/publish \
    --runtime linux-x64 \
    --self-contained false \
    /p:PublishReadyToRun=true
```

### Команды диагностики

```bash
# Проверка логов
docker logs -f --tail 100 <container_id>

# Проверка ресурсов
docker stats <container_id>

# Вход в контейнер для диагностики
docker exec -it <container_id> /bin/bash

# Health check
curl http://localhost:8080/health

# Metrics
curl http://localhost:8080/metrics
```

## 💡 Best Practices для ASP.NET Core в Docker

### Безопасность

- Используйте непривилегированных пользователей
- Регулярно обновляйте базовые образы
- Сканируйте образы на уязвимости
- Используйте secrets для конфиденциальных данных

### Производительность

- Используйте multi-stage builds
- Оптимизируйте слои Dockerfile
- Настройте кэширование зависимостей
- Используйте alpine-based образы

### Наблюдаемость

- Настройте structured logging
- Добавьте comprehensive health checks
- Экспортируйте метрики для Prometheus
- Используйте distributed tracing

### Операционная готовность

- Настройте graceful shutdown
- Реализуйте readiness/liveness probes
- Добавьте circuit breaker паттерны
- Настройте автоматическое восстановление

**Ключевой вывод**: ASP.NET Core отлично подходит для контейнеризации благодаря кроссплатформенности и оптимизациям для cloud-native сред. Правильно настроенный Docker образ обеспечивает безопасность, производительность и удобство операционного управления.
