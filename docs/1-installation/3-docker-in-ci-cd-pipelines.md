# Docker in CI-CD Pipelines

**Docker** — это фундаментальный компонент современных CI/CD пайплайнов, обеспечивающий воспроизводимость, изоляцию и переносимость процессов сборки и развертывания.

## Архитектура Docker в CI/CD

### Общая схема пайплайна

```bash
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Source Code   │    │   CI Pipeline   │    │   CD Pipeline   │
│                 │    │                 │    │                 │
│  ┌─────────────┐│    │  ┌─────────────┐│    │  ┌─────────────┐│
│  │   Git       ││    │  │   Docker    ││    │  │ Production  ││
│  │ Repository  │├───►│  │   Build     │├───►│  │  Deployment ││
│  └─────────────┘│    │  └─────────────┘│    │  └─────────────┘│
│                 │    │         │       │    │                 │
│                 │    │         ▼       │    │                 │
│                 │    │  ┌─────────────┐│    │  ┌─────────────┐│
│                 │    │  │ Container   ││    │  │   Docker    ││
│                 │    │  │  Registry   ││    │  │   Swarm /   ││
│                 │    │  │             ││    │  │ Kubernetes  ││
│                 │    │  └─────────────┘│    │  └─────────────┘│
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Модели интеграции Docker в CI/CD

### 1. Docker как среда выполнения (Docker-in-Docker)

```bash
CI Runner → Docker Container → Build & Test → Push Image
    ↑              ↑
Docker Socket   Docker Images
```

### 2. Docker как артефакт (Build and Push)

```bash
Source → Build Image → Test Image → Push to Registry → Deploy
```

### 3. Многоступенчатые сборки в CI

```bash
Build Stage → Test Stage → Security Scan → Push Stage
     ↓             ↓             ↓            ↓
Dockerfile    Test Runner    Trivy/Grype   Registry
```

## Реализация в популярных CI/CD системах

### GitHub Actions

```yaml
name: .NET 7 Docker CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: "7.0.x"

      - name: Build Docker image
        run: |
          docker build -t $IMAGE_NAME:latest -f Dockerfile .

      - name: Run tests in container
        run: |
          docker run --rm $IMAGE_NAME:latest dotnet test

      - name: Run security scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: $IMAGE_NAME:latest
          format: "sarif"
          output: "trivy-results.sarif"

      - name: Log in to Container Registry
        if: github.ref == 'refs/heads/main'
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push Docker image
        if: github.ref == 'refs/heads/main'
        run: |
          docker tag $IMAGE_NAME:latest $REGISTRY/$IMAGE_NAME:${{ github.sha }}
          docker push $REGISTRY/$IMAGE_NAME:${{ github.sha }}
          docker tag $IMAGE_NAME:latest $REGISTRY/$IMAGE_NAME:latest
          docker push $REGISTRY/$IMAGE_NAME:latest
```

### GitLab CI

```yaml
stages:
  - build
  - test
  - security
  - deploy

variables:
  DOCKER_REGISTRY: registry.gitlab.com
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE

build:
  stage: build
  image: docker:24.0
  services:
    - docker:24.0-dind
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE:latest -t $DOCKER_IMAGE:$CI_COMMIT_SHA .
    - docker push $DOCKER_IMAGE:latest
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHA
  only:
    - main

test:
  stage: test
  image: $DOCKER_IMAGE:latest
  services:
    - postgres:13
    - redis:7
  variables:
    POSTGRES_DB: test_db
    REDIS_URL: redis://redis:6379
  script:
    - dotnet test --verbosity normal
    - dotnet ef database update
  dependencies:
    - build

security-scan:
  stage: security
  image:
    name: aquasec/trivy:0.45
    entrypoint: [""]
  variables:
    TRIVY_USERNAME: $CI_REGISTRY_USER
    TRIVY_PASSWORD: $CI_REGISTRY_PASSWORD
    TRIVY_REGISTRY: $CI_REGISTRY
  script:
    - trivy image --exit-code 0 --severity HIGH,CRITICAL $DOCKER_IMAGE:latest
    - trivy image --exit-code 1 --severity CRITICAL $DOCKER_IMAGE:latest
  dependencies:
    - build

deploy:
  stage: deploy
  image: alpine:3.18
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
  script:
    - ssh -o StrictHostKeyChecking=no $DEPLOY_USER@$DEPLOY_SERVER "
      docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY &&
      docker pull $DOCKER_IMAGE:$CI_COMMIT_SHA &&
      docker service update --image $DOCKER_IMAGE:$CI_COMMIT_SHA my_app_service"
  environment:
    name: production
    url: https://myapp.example.com
  only:
    - main
```

### Azure DevOps

```yaml
trigger:
  branches:
    include:
      - main
      - develop

variables:
  dockerRegistry: "my-docker-registry"
  imageRepository: "my-dotnet-app"
  dockerfilePath: "**/Dockerfile"
  tag: "$(Build.BuildId)"

stages:
  - stage: Build
    displayName: Build and Test
    jobs:
      - job: Build
        displayName: Build Docker Image
        pool:
          vmImage: "ubuntu-latest"

        steps:
          - task: Docker@2
            displayName: Build Docker Image
            inputs:
              command: build
              repository: $(imageRepository)
              dockerfile: $(dockerfilePath)
              containerRegistry: $(dockerRegistry)
              tags: |
                $(tag)
                latest

          - task: Docker@2
            displayName: Run Tests in Container
            inputs:
              command: run
              containerRegistry: $(dockerRegistry)
              repository: $(imageRepository)
              tags: $(tag)
              arguments: "dotnet test --verbosity normal"

  - stage: Security
    displayName: Security Scan
    dependsOn: Build
    jobs:
      - job: Scan
        pool:
          vmImage: "ubuntu-latest"

        steps:
          - task: Docker@2
            displayName: Pull Image for Scanning
            inputs:
              command: pull
              containerRegistry: $(dockerRegistry)
              repository: $(imageRepository)
              tags: $(tag)

          - script: |
              docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                aquasec/trivy:latest image \
                $(dockerRegistry)/$(imageRepository):$(tag) \
                --format table --exit-code 0 --severity HIGH,CRITICAL
            displayName: "Security Scan with Trivy"

  - stage: Deploy
    displayName: Deploy to Production
    dependsOn: Security
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))

    jobs:
      - deployment: Deploy
        environment: "production"
        pool:
          vmImage: "ubuntu-latest"

        strategy:
          runOnce:
            deploy:
              steps:
                - task: Docker@2
                  displayName: Deploy to Swarm
                  inputs:
                    command: deploy
                    containerRegistry: $(dockerRegistry)
                    repository: $(imageRepository)
                    tags: $(tag)
                    arguments: "stack deploy --with-registry-auth -c docker-compose.prod.yml myapp"
```

## Многоступенчатые Dockerfile для CI/CD

### Оптимизированный Dockerfile для .NET 7

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src

# Copy project files
COPY ["MyApp/MyApp.csproj", "MyApp/"]
COPY ["MyApp.Tests/MyApp.Tests.csproj", "MyApp.Tests/"]
RUN dotnet restore "MyApp/MyApp.csproj"
RUN dotnet restore "MyApp.Tests/MyApp.Tests.csproj"

# Copy everything else and build
COPY . .
WORKDIR "/src/MyApp"
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

# Test stage
FROM build AS test
WORKDIR "/src/MyApp.Tests"
RUN dotnet test --verbosity normal --logger "trx;LogFileName=testresults.trx"

# Publish stage
FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

## Best Practices для Docker в CI/CD

### 1. Кэширование слоев

```yaml
# GitHub Actions с кэшированием
- name: Cache Docker layers
  uses: actions/cache@v3
  with:
    path: /tmp/.buildx-cache
    key: ${{ runner.os }}-buildx-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-buildx-

- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v2
  with:
    cache-to: type=local,dest=/tmp/.buildx-cache-new
    cache-from: type=local,src=/tmp/.buildx-cache
```

### 2. Сканирование безопасности

```yaml
- name: Run comprehensive security scan
  run: |
    docker run --rm \
      -v /var/run/docker.sock:/var/run/docker.sock \
      aquasec/trivy:latest \
      image --format sarif \
      --output trivy-results.sarif \
      --exit-code 0 \
      ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
```

### 3. Тегирование образов

```bash
# Semantic versioning
docker tag myapp:latest myregistry.com/myapp:1.2.3
docker tag myapp:latest myregistry.com/myapp:1.2
docker tag myapp:latest myregistry.com/myapp:1

# Git-based tagging
docker tag myapp:latest myregistry.com/myapp:${CI_COMMIT_SHA:0:8}
docker tag myapp:latest myregistry.com/myapp:${CI_COMMIT_REF_SLUG}
```

## Мониторинг и метрики в CI/CD

### Health checks в Docker

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

### Сбор метрик в пайплайне

```yaml
- name: Collect build metrics
  run: |
    echo "IMAGE_SIZE=$(docker image inspect myapp:latest --format='{{.Size}}')" >> $GITHUB_ENV
    echo "BUILD_TIME=${SECONDS}s" >> $GITHUB_ENV
    echo "LAYER_COUNT=$(docker image history myapp:latest --format='{{.ID}}' | wc -l)" >> $GITHUB_ENV
```

## Продвинутые сценарии

### Canary deployments

```yaml
- name: Canary deployment
  run: |
    # Deploy to 10% of instances
    docker service update --image myapp:new-version --update-parallelism 1 --update-delay 30s my_app_service

    # Wait and check metrics
    sleep 300

    # If healthy, deploy to 100%
    docker service update --image myapp:new-version my_app_service
```

### Blue-Green deployment

```yaml
- name: Blue-Green deployment
  run: |
    # Deploy green environment
    docker stack deploy -c docker-compose.green.yml myapp-green

    # Test green environment
    curl -f http://green.myapp.com/health

    # Switch traffic
    docker service update --image myapp:new-version myapp-blue

    # Remove old environment
    docker stack rm myapp-green
```

**Архитектурный принцип**: Docker в CI/CD пайплайнах обеспечивает детерминистичные сборки, изолированное тестирование и надежное развертывание, превращая процесс доставки программного обеспечения в предсказуемый и воспроизводимый конвейер.
