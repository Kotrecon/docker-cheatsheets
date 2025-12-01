# Docker Content Trust: цифровые подписи и доверенные образы

Docker Content Trust (DCT) — это система цифровых подписей, обеспечивающая целостность и аутентичность Docker-образов. Она гарантирует, что вы получаете именно те образы, которые были подписаны издателем.

## Что такое Docker Content Trust?

**Docker Content Trust** — механизм цифровой подписи, основанный на The Update Framework (TUF), который защищает от:

- Модификации образов при передаче
- Подмены образов злоумышленниками
- Распространения неподписанных образов

## Архитектура DCT

```text
Publisher → Signs Image → Push to Registry
                          ↓
User → Verifies Signature → Pull Trusted Image
```

## Настройка Docker Content Trust

### 🔧 Базовая настройка

```bash
# Включение DCT для всех операций
export DOCKER_CONTENT_TRUST=1

# Или только для конкретного реестра
export DOCKER_CONTENT_TRUST_SERVER=https://myregistry:4443
```

### 🗝️ Генерация ключей

```bash
# Генерация корневого ключа (требует парольную фразу)
docker trust key generate my_publisher

# Генерация ключа репозитория
docker trust signer add --key publisher.pub my_publisher myregistry/my-app
```

## Работа с подписями образов

### ✍️ Подписание образов

```bash
# Сборка и подписание образа
docker build -t myregistry/my-app:1.0 .
docker trust sign myregistry/my-app:1.0

# Подписание с конкретным ключом
docker trust sign myregistry/my-app:1.0 --local my_publisher
```

### 🔍 Проверка подписей

```bash
# Просмотр информации о подписи
docker trust inspect myregistry/my-app:1.0 --pretty

# Проверка подписей для всех тегов
docker trust inspect myregistry/my-app
```

### Пример вывода проверки

```json
{
  "Name": "myregistry/my-app:1.0",
  "SignedTags": [
    {
      "SignedTag": "1.0",
      "Digest": "sha256:abc123...",
      "Signers": ["my_publisher"]
    }
  ]
}
```

## Интеграция с CI/CD пайплайнами

### GitHub Actions с DCT

```yaml
name: Build and Sign
on:
  push:
    tags: ["v*"]
jobs:
  build-and-sign:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ secrets.REGISTRY_URL }}
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - name: Import Signing Key
        run: |
          echo "${{ secrets.DCT_PRIVATE_KEY }}" > private.key
          docker trust key load private.key --name ci_signer

      - name: Build and Sign
        run: |
          export DOCKER_CONTENT_TRUST=1
          docker build -t ${{ secrets.REGISTRY_URL }}/my-app:${{ github.ref_name }} .
          docker trust sign ${{ secrets.REGISTRY_URL }}/my-app:${{ github.ref_name }}
```

## Практические примеры для вашего стека

### .NET 7 приложения с подписью

```dockerfile
# Dockerfile для подписываемого .NET 7 приложения
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=build /app .
USER 1000
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

```bash
# Сборка и подписание
docker build -t myregistry/net7-app:1.2.0 .
docker trust sign myregistry/net7-app:1.2.0
```

### JavaScript/TypeScript приложения с подписью

```dockerfile
# Dockerfile для подписываемого Node.js приложения
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
RUN addgroup -g 1001 -S nodejs && adduser -S nextjs -u 1001
WORKDIR /app
COPY --from=build --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --chown=nextjs:nodejs . .
USER nextjs
CMD ["npm", "start"]
```

```bash
# Подписание разных тегов
docker trust sign myregistry/node-app:1.0.0
docker trust sign myregistry/node-app:latest
```

## Работа с приватными реестрами

### Harbor с DCT

```bash
# Настройка для Harbor registry
export DOCKER_CONTENT_TRUST_SERVER=https://harbor.example.com:4443
export DOCKER_CONTENT_TRUST_ROOT_PASSPHRASE=my_root_pass
export DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE=my_repo_pass

# Подписание образа в Harbor
docker trust sign harbor.example.com/my-project/my-app:1.0
```

### GitLab Container Registry

```bash
# Подписание образов в GitLab
docker trust sign registry.gitlab.com/my-group/my-project:latest
```

## Мониторинг и управление подписями

### 📊 Проверка статуса подписей

```bash
# Список всех подписанных образов в реестре
docker trust inspect --pretty myregistry/my-app

# Проверка конкретного тега
docker trust inspect myregistry/my-app:1.0

# История подписей
docker trust inspect myregistry/my-app --show-signatures
```

### 🗑️ Управление ключами

```bash
# Удаление подписанта
docker trust signer remove my_publisher myregistry/my-app

# Ротация ключей
docker trust key rotate myregistry/my-app
```

## Best Practices для DCT

### ✅ Рекомендации по использованию

```bash
# 1. Всегда используйте конкретные теги версий
docker trust sign myregistry/my-app:1.2.3

# 2. Избегайте подписи тега 'latest'
# 3. Храните ключи в безопасном месте
# 4. Регулярно обновляйте ключи
```

### 🔒 Безопасное хранение ключей

```bash
# Использование аппаратных токенов
docker trust key generate my_publisher --hardware

# Шифрование ключей в репозитории
gpg --symmetric --cipher-algo AES256 private.key
```

## Отладка проблем с DCT

### 🔍 Распространенные ошибки

```bash
# Проверка настроек DCT
echo $DOCKER_CONTENT_TRUST

# Просмотр логов доверия
docker trust inspect myregistry/my-app --debug

# Сброс локальных метаданных доверия
docker trust revoke myregistry/my-app:1.0
```

### Пример отладки

```bash
# Если образ не подписан
Error: remote trust data does not exist for myregistry/my-app:1.0

# Решение: подписать образ или отключить DCT для этого образа
export DOCKER_CONTENT_TRUST=0
docker pull myregistry/my-app:1.0
```

**Ключевой вывод**: Docker Content Trust обеспечивает критически важный уровень безопасности, гарантируя целостность и аутентичность Docker-образов. Интеграция DCT в процессы сборки и развертывания защищает от атак на цепочку поставок и обеспечивает доверенную доставку контейнеров в production-окружения.
