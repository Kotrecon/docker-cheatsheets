Отлично! Обновляю файл с учетом использования только open-source решений:

---

# Docker Secrets Management: управление конфиденциальными данными

Управление секретами — критически важный аспект безопасности Docker, обеспечивающий защиту паролей, токенов, ключей API и других конфиденциальных данных в containerized средах.

## Что такое Docker Secrets?

**Docker Secrets** — это безопасный механизм для хранения и передачи конфиденциальной информации контейнерам без раскрытия данных в environment variables или файлах конфигурации.

## Архитектура управления секретами

```text
Secret Storage → Docker Swarm/External Vault → Container Runtime
       ↓                  ↓                          ↓
   Encrypted          Secure Transfer          In-Memory Filesystem
     Data              (/run/secrets/)           (/run/secrets/)
```

## Встроенные Docker Secrets (Swarm Mode)

### 🔧 Создание и управление секретами

```bash
# Создание секрета из файла
echo "my-secret-password" | docker secret create db_password -

# Создание секрета из stdin
cat config.json | docker secret create app_config -

# Просмотр списка секретов
docker secret ls

# Информация о секрете
docker secret inspect db_password
```

### 🐳 Использование в Docker Compose

```yaml
# docker-compose.secrets.yml
version: "3.8"
services:
  webapp:
    image: myregistry/my-app:latest
    secrets:
      - db_password
      - api_key
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password
      - API_KEY_FILE=/run/secrets/api_key

secrets:
  db_password:
    external: true
  api_key:
    file: ./api-key.txt
```

## Open-source системы управления секретами

### 🔑 HashiCorp Vault (Open Source)

```bash
# Запуск Vault в Docker
docker run -d --name=vault \
  -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=root \
  -e VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200 \
  vault:latest

# Инициализация Vault
docker exec vault vault secrets enable -path=secret kv-v2
```

### 🔐 SOPS (Secrets OPerationS) + Age

```bash
# Установка SOPS
curl -LO https://github.com/mozilla/sops/releases/download/v3.7.3/sops-v3.7.3.linux
sudo install sops-v3.7.3.linux /usr/local/bin/sops

# Генерация ключа Age
age-keygen -o age-key.txt
```

### 🗄️ Bitnami Sealed Secrets для Kubernetes

```bash
# Установка kubeseal
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.23.0/kubeseal-0.23.0-linux-amd64.tar.gz
tar -xzf kubeseal-0.23.0-linux-amd64.tar.gz
sudo install kubeseal /usr/local/bin/
```

## Практические примеры для вашего стека

### .NET 7 приложения с секретами

```dockerfile
# Dockerfile для .NET 7 с поддержкой секретов
FROM mcr.microsoft.com/dotnet/aspnet:7.0
RUN adduser --disabled-password --gecos '' appuser
WORKDIR /app
COPY --chown=appuser:appuser . .
USER appuser

# Чтение секретов из файлов
CMD ["dotnet", "MyApp.dll",
     "--DbPassword", "$(cat /run/secrets/db_password)",
     "--ApiKey", "$(cat /run/secrets/api_key)"]
```

```csharp
// Чтение секретов в .NET 7 приложении
public static void Main(string[] args)
{
    var dbPassword = File.ReadAllText("/run/secrets/db_password");
    var apiKey = File.ReadAllText("/run/secrets/api_key");

    var builder = WebApplication.CreateBuilder(args);
    builder.Services.AddDbContext<AppDbContext>(options =>
        options.UseSqlServer($"Server=db;Database=app;User=sa;Password={dbPassword};"));
}
```

### JavaScript/TypeScript приложения с секретами

```dockerfile
# Dockerfile для Node.js с поддержкой секретов
FROM node:18-alpine
RUN addgroup -g 1001 -S nodejs && adduser -S appuser -u 1001
WORKDIR /app
COPY --chown=appuser:nodejs package*.json ./
RUN npm ci --only=production
COPY --chown=appuser:nodejs . .
USER appuser

# Скрипт для загрузки секретов
COPY --chown=appuser:nodejs docker-entrypoint.sh ./
ENTRYPOINT ["./docker-entrypoint.sh"]
CMD ["npm", "start"]
```

```javascript
// docker-entrypoint.sh для Node.js
#!/bin/sh
set -e

# Загрузка секретов в environment variables
export DB_PASSWORD=$(cat /run/secrets/db_password)
export JWT_SECRET=$(cat /run/secrets/jwt_secret)
export API_KEY=$(cat /run/secrets/api_key)

exec "$@"
```

```typescript
// app.ts - использование секретов в TypeScript
const dbConfig = {
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || "5432"),
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD, // Из секрета
  database: process.env.DB_NAME,
};

const jwtSecret = process.env.JWT_SECRET; // Из секрета
```

## Docker Compose с секретами для разработки

### 🛠️ Development окружение

```yaml
# docker-compose.dev.yml
version: "3.8"
services:
  webapp:
    build: .
    secrets:
      - db_password
      - jwt_secret
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - DB_HOST=db
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=appuser
    secrets:
      - postgres_password
    command: >
      sh -c "
        echo \"POSTGRES_PASSWORD=$$(cat /run/secrets/postgres_password)\" >> /tmp/env &&
        docker-entrypoint.sh postgres
      "

secrets:
  db_password:
    file: ./secrets/db_password.txt
  jwt_secret:
    file: ./secrets/jwt_secret.txt
  postgres_password:
    file: ./secrets/postgres_password.txt
```

## Production окружение с Docker Swarm

### 🚀 Swarm Mode развертывание

```yaml
# docker-compose.prod.yml
version: "3.8"
services:
  webapp:
    image: myregistry/my-app:${TAG:-latest}
    deploy:
      replicas: 3
      restart_policy:
        condition: any
    secrets:
      - source: db_password_prod
        target: db_password
      - source: jwt_secret_prod
        target: jwt_secret

  database:
    image: postgres:15
    deploy:
      placement:
        constraints: [node.role == manager]
    secrets:
      - source: postgres_password_prod
        target: postgres_password

secrets:
  db_password_prod:
    external: true
  jwt_secret_prod:
    external: true
  postgres_password_prod:
    external: true
```

## Интеграция с CI/CD пайплайнами

### GitHub Actions с Docker Secrets

```yaml
name: Deploy with Secrets
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Deploy to Swarm
        run: |
          echo "${{ secrets.DB_PASSWORD }}" | docker secret create db_password_prod -
          echo "${{ secrets.JWT_SECRET }}" | docker secret create jwt_secret_prod -

          docker stack deploy -c docker-compose.prod.yml myapp
```

### GitLab CI с SOPS

```yaml
# .gitlab-ci.yml
deploy_production:
  stage: deploy
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - apk add --no-cache sops age
    - echo "$SOPS_AGE_KEY" > age-key.txt
  script:
    - |
      # Декрипт секретов
      sops --decrypt --age age-key.txt secrets.enc.yaml > secrets.yaml

      # Создание Docker secrets
      cat secrets.yaml | yq '.db_password' | docker secret create db_password_prod -
      cat secrets.yaml | yq '.jwt_secret' | docker secret create jwt_secret_prod -
```

## Open-source альтернативы для разных сценариев

### 🔧 Для простых проектов

- **Docker Swarm Secrets** - встроенное решение
- **.env файлы** с ограниченным доступом

### 🚀 Для средних проектов

- **HashiCorp Vault OSS** - полнофункциональное решение
- **SOPS + Git** - шифрование в репозитории

### 🏢 Для сложных инфраструктур

- **Bitnami Sealed Secrets** (для Kubernetes)
- **Vault OSS с консулом** - высокодоступная настройка

## Best Practices для управления секретами

### ✅ Рекомендации по безопасности

```bash
# 1. Никогда не храните секреты в Dockerfile
# 2. Используйте .dockerignore для исключения файлов с секретами
# 3. Регулярно ротируйте секреты
# 4. Используйте разные секреты для разных окружений
# 5. Аудит доступа к секретам
```

### 🔒 Шифрование и ротация

```bash
# Ротация секретов в Swarm
docker secret create new_db_password -
docker service update --secret-rm db_password --secret-add source=new_db_password,target=db_password myapp_web

# Просмотр истории секретов
docker service ps myapp_web --no-trunc
```

## Мониторинг и аудит секретов

### 📊 Отслеживание использования

```bash
# Какие сервисы используют секреты
docker service inspect myapp_web --format='{{.Spec.TaskTemplate.ContainerSpec.Secrets}}'

# Проверка доступа к секретам
docker exec <container_id> ls -la /run/secrets/

# Аудит изменений секретов
docker events --filter type=secret
```

### Пример мониторинга:

```bash
# Скрипт проверки целостности секретов
#!/bin/bash
for secret in $(docker secret ls -q); do
    echo "Checking secret: $secret"
    docker secret inspect $secret --format '{{.CreatedAt}} {{.UpdatedAt}}'
done
```

**Ключевой вывод**: Open-source решения предоставляют полный набор инструментов для безопасного управления секретами в Docker. От встроенных Docker Swarm Secrets до полнофункционального HashiCorp Vault — все необходимое доступно бесплатно для проектов любого масштаба.

---

Теперь раздел "Безопасность" полностью завершен с open-source фокусом! ✅ Переходим к **Практическим сценариям**?
