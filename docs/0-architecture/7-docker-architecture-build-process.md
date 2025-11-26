# Docker Architecture: Build Process

**Docker Build Process** — это многоступенчатый процесс преобразования Dockerfile в готовый к использованию Docker-образ, использующий кэширование слоёв и оптимизацию для эффективной сборки.

## Архитектура процесса сборки

### Общая схема сборки

```cli
┌─────────────────────────────────────────────────┐
│               Docker Build Process              │
│                                                 │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │  Dockerfile │        │   Build Context     │ │
│  │             │        │                     │ │
│  │ Instructions│        │  Source Code        │ │
│  │             │        │  Dependencies       │ │
│  │             │        │  Resources          │ │
│  └─────────────┘        └─────────────────────┘ │
│           │                      │              │
│           ▼                      ▼              │
│  ┌─────────────────────────────────────────────┐│
│  │            Docker Builder                   ││
│  │                                             ││
│  │  ┌─────────────┐    ┌─────────────────────┐ ││
│  │  │  Parser     │    │   Executor          │ ││
│  │  │             │    │                     │ ││
│  │  │ Parse       │    │ Execute instructions│ ││
│  │  │ Dockerfile  │    │  & create layers    │ ││
│  │  └─────────────┘    └─────────────────────┘ ││
│  └─────────────────────────────────────────────┘│
│                           │                     │
│                           ▼                     │
│                  ┌─────────────────┐            │
│                  │  Docker Image   │            │
│                  │                 │            │
│                  │  Immutable      │            │
│                  │  Layered        │            │
│                  └─────────────────┘            │
└─────────────────────────────────────────────────┘
```

## Компоненты системы сборки

### 1. Build Context (контекст сборки)

```bash
Build Context Structure:
/app/
├── Dockerfile
├── .dockerignore
├── src/
│   ├── main.py
│   └── utils.py
├── requirements.txt
├── package.json
└── config/
    └── settings.yaml
```

### 2. Dockerfile Parser (парсер)

```dockerfile
# Dockerfile Instructions:
FROM ubuntu:22.04              # Base image
COPY . /app                    # File operations
RUN apt-get update && \        # Command execution
    apt-get install -y python3
ENV PYTHONPATH=/app           # Environment variables
EXPOSE 8000                   # Network configuration
CMD ["python3", "app.py"]     # Runtime command
```

### 3. Layer Builder (построитель слоёв)

```bash
Layer Creation Process:
Instruction → Temporary Container → Filesystem Snapshot → New Layer
     ↓               ↓                   ↓               ↓
  FROM ubuntu    Create container    Take snapshot    Save as layer
    COPY         Copy files          Take snapshot    Save as layer
    RUN          Execute command     Take snapshot    Save as layer
```

## Детальный процесс сборки

### Фазы сборки

```bash
1. Context Preparation
   ↓
2. Dockerfile Parsing
   ↓
3. Base Image Pull
   ↓
4. Layer-by-Layer Execution
   ↓
5. Image Configuration
   ↓
6. Final Image Creation
```

### 1. Context Preparation (подготовка контекста)

```bash
# Build context отправляется Docker daemon
docker build -t my-app .
# ↑                     ↑
# Image tag          Build context directory
```

### 2. Dockerfile Parsing (парсинг Dockerfile)

```bash
Dockerfile Analysis:
┌──────────────┬─────────────────┬──────────────┐
│  Instruction │     Arguments   │  Layer Type  │
├──────────────┼─────────────────┼──────────────┤
│     FROM     │   ubuntu:22.04  │   Base Layer │
│     COPY     │     . /app      │   File Layer │
│     RUN      │  apt-get update │   Exec Layer │
│     ENV      │  PATH=/usr/bin  │  Metadata    │
└──────────────┴─────────────────┴──────────────┘
```

### 3. Layer Execution (выполнение по слоям)

```bash
Step 1/6 : FROM ubuntu:22.04
  → Pull base image
  → Create read-only layer (Layer 1)

Step 2/6 : RUN apt-get update
  → Create temporary container from Layer 1
  → Execute command in container
  → Create filesystem snapshot
  → Save as new read-only layer (Layer 2)

Step 3/6 : COPY . /app
  → Add files from build context
  → Create new layer with files (Layer 3)
```

## Кэширование слоёв

### Механизм кэширования

```bash
Layer Cache Key = Instruction + Context Hash
     ↓
Cache Hit → Reuse existing layer
Cache Miss → Execute instruction and create new layer
```

### Пример кэширования

```dockerfile
# Layer 1: Кэшируется до изменения базового образа
FROM node:18-alpine

# Layer 2: Кэшируется до изменения package.json
COPY package*.json ./

# Layer 3: Кэшируется до изменения команды
RUN npm ci

# Layer 4: Кэш сбрасывается при любом изменении исходного кода
COPY . .

# Layer 5: Кэшируется до изменения команды
CMD ["node", "server.js"]
```

### Cache Invalidation (инвалидация кэша)

```bash
Instruction Change:    RUN apt-get install python3 → RUN apt-get install python3.9
Context File Change:   COPY requirements.txt ./     (файл изменён)
Base Image Update:     FROM node:16 → FROM node:18
Build Argument Change: --build-arg VERSION=1.0 → --build-arg VERSION=2.0
```

## Многоступенчатая сборка (Multi-stage Builds)

### Архитектура многоступенчатой сборки

```bash
┌─────────────────────────────────────────────────┐
│              Multi-stage Build                  │
│                                                 │
│  Stage 1: Builder                               │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │   FROM      │        │   Build Tools       │ │
│  │  node:18    │───────►│   Dependencies      │ │
│  │             │        │   Source Code       │ │
│  └─────────────┘        └─────────────────────┘ │
│           │                      │              │
│           ▼                      ▼              │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │   COPY      │        │   Build Artifacts   │ │
│  │  --from=    │        │   (dist/, target/)  │ │
│  │   builder   │        │                     │ │
│  └─────────────┘        └─────────────────────┘ │
│           │                                     │
│           ▼                                     │
│  Stage 2: Runtime                               │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │   FROM      │        │   Final Image       │ │
│  │ nginx:alpine│───────►│   Small Size        │ │
│  │             │        │   Production Ready  │ │
│  └─────────────┘        └─────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### Пример multi-stage Dockerfile

```dockerfile
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## BuildKit - современный движок сборки

### Архитектура BuildKit

```bash
┌─────────────────────────────────────────────────┐
│                 BuildKit                        │
│                                                 │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │  Frontend   │        │    LLB              │ │
│  │             │        │  (Low-Level Build)  │ │
│  │ Dockerfile  │───────►│                     │ │
│  │  Parser     │        │  Dependency Graph   │ │
│  └─────────────┘        └─────────────────────┘ │
│                           │                     │
│                           ▼                     │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │  Workers    │        │   Cache Manager     │ │
│  │             │        │                     │ │
│  │ Parallel    │        │  Local & Remote     │ │
│  │ Execution   │        │     Cache           │ │
│  └─────────────┘        └─────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### Преимущества BuildKit

- **Parallel Layer Building** (параллельная сборка слоёв)
- **Incremental Transfer** (инкрементальная передача данных)
- **Secret Management** (управление секретами)
- **Cache Import/Export** (импорт/экспорт кэша)

## Оптимизация процесса сборки

### Стратегии оптимизации

**1. Layer Ordering (упорядочивание слоёв):**

```dockerfile
# ❌ Плохо - часто меняющийся слой в начале
COPY . .
RUN npm install

# ✅ Хорошо - редко меняющиеся слои в начале
COPY package*.json ./
RUN npm install
COPY . .
```

**2. .dockerignore Optimization:**

```dockerignore
# Исключение ненужных файлов из контекста
node_modules/
.git/
*.log
.env
Dockerfile*
```

**3. Multi-stage для уменьшения размера:**

```dockerfile
FROM golang:1.19 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

FROM scratch
COPY --from=builder /app/myapp /
CMD ["/myapp"]
```

## Расширенные возможности сборки

### Build Arguments (аргументы сборки)

```dockerfile
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}

ARG BUILD_ENV=production
ENV NODE_ENV=${BUILD_ENV}

# Использование:
# docker build --build-arg NODE_VERSION=16 --build-arg BUILD_ENV=development .
```

### Secret Management (управление секретами)

```dockerfile
# Dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install

# Сборка:
# docker build --secret id=npmrc,src=.npmrc .
```

### Cache Mounts (кэш-монтирования)

```dockerfile
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y packages

RUN --mount=type=cache,target=/root/.npm \
    npm install
```

## Мониторинг и отладка сборки

### Build Insights (анализ сборки)

```bash
# Детальная информация о сборке
docker build --progress=plain .

# Анализ размера образа
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# История сборки
docker history my-app:latest
```

### Performance Metrics (метрики производительности)

```bash
Build Time: 2m 15s
Image Size: 245 MB
Layers: 8
Cache Efficiency: 85%
```

**Архитектурный принцип**: Docker Build Process преобразует декларативные инструкции Dockerfile в оптимизированные, многослойные образы, используя интеллектуальное кэширование и параллельное выполнение для обеспечения быстрой, воспроизводимой и эффективной сборки контейнеров.
