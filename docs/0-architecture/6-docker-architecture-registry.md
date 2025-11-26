# Docker Architecture: Registry

**Docker Registry** — это хранилище для распределения Docker-образов, позволяющее хранить, управлять и распространять образы между разными окружениями и пользователями.

## Архитектура Docker Registry

### Компоненты registry-системы

```cli
┌─────────────────────────────────────────────────┐
│                 Docker Client                   │
│                                                 │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │   docker    │        │   Docker Registry   │ │
│  │   pull      │───────►│                     │ │
│  │   push      │◄───────│  Storage Backend    │ │
│  │   login     │        │                     │ │
│  └─────────────┘        └─────────────────────┘ │
│                          │                      │
│                          ▼                      │
│                 ┌─────────────────┐             │
│                 │   Auth Server   │             │
│                 │  (optional)     │             │
│                 └─────────────────┘             │
└─────────────────────────────────────────────────┘
```

## Типы Docker Registry

### 1. Docker Hub (публичный registry)

```cli
┌─────────────────────────────────────────────────┐
│               Global Ecosystem                  │
│                                                 │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │   Client A  │        │    Docker Hub       │ │
│  │             │───────►│                     │ │
│  └─────────────┘        │  ┌─────────────┐    │ │
│                         │  │  Official   │    │ │
│  ┌─────────────┐        │  │  Images     │    │ │
│  │   Client B  │───────►│  └─────────────┘    │ │
│  │             │        │  ┌─────────────┐    │ │
│  └─────────────┘        │  │ User Images │    │ │
│                         │  └─────────────┘    │ │
│  ┌─────────────┐        │  ┌─────────────┐    │ │
│  │   Client C  │───────►│  │ Org Images  │    │ │
│  │             │        │  └─────────────┘    │ │
│  └─────────────┘        └─────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### 2. Private Registry (приватный registry)

```cli
┌─────────────────────────────────────────────────┐
│                Organization                     │
│                                                 │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │  Developer  │        │  Private Registry   │ │
│  │             │───────►│                     │ │
│  └─────────────┘        │  ┌─────────────┐    │ │
│                         │  │  Staging    │    │ │
│  ┌─────────────┐        │  │   Images    │    │ │
│  │    CI/CD    │───────►│  └─────────────┘    │ │
│  │  Pipeline   │        │  ┌─────────────┐    │ │
│  └─────────────┘        │  │ Production  │    │ │
│                         │  │   Images    │    │ │
│  ┌─────────────┐        │  └─────────────┘    │ │
│  │ Production  │───────►│                     │ │
│  │  Servers    │        └─────────────────────┘ │
│  └─────────────┘                 │              │
│                                  ▼              │
│                         ┌─────────────────┐     │
│                         │  Storage Backend│     │
│                         │   (S3, NFS...)  │     │
│                         └─────────────────┘     │
└─────────────────────────────────────────────────┘
```

### 3. Registry Mirror (зеркало registry)

```cli
┌─────────────────────────────────────────────────┐
│                 Local Network                   │
│                                                 │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │   Docker    │        │  Registry Mirror    │ │
│  │   Clients   │───────►│                     │ │
│  │             │        │  ┌─────────────┐    │ │
│  └─────────────┘        │  │   Cache     │    │ │
│                         │  │   Layer     │    │ │
│                         │  └─────────────┘    │ │
│                         └─────────┬───────────┘ │
│                                   │             │
│                                   ▼             │
│                         ┌─────────────────────┐ │
│                         │   Docker Hub        │ │
│                         │   (upstream)        │ │
│                         └─────────────────────┘ │
└─────────────────────────────────────────────────┘
```

## Компоненты Registry

### Storage Backend (бэкенд хранения)

```bash
/var/lib/registry/
├── docker/
│   └── registry/
│       └── v2/
│           ├── blobs/
│           │   └── sha256/
│           │       ├── aa/
│           │       │   └── aa256... (layer data)
│           │       └── bb/
│           │           └── bb498... (layer data)
│           └── repositories/
│               └── myapp/
│                   ├── _layers/
│                   ├── _manifests/
│                   └── _uploads/
```

### Manifest System (система манифестов)

```json
Image Manifest:
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "size": 1469,
    "digest": "sha256:e6...",
    "layers": [
      {
        "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
        "size": 2813313,
        "digest": "sha256:aa..."
      }
    ]
  }
}
```

## Протокол взаимодействия

### Docker Registry API v2

```cli
Client ↔ Registry Communication:
1. Authentication (optional)
2. Image Push/Pull
3. Manifest Operations
4. Blob Upload/Download
```

### Push Process (процесс загрузки)

```cli
1. POST /v2/<name>/blobs/uploads/    # Initiate upload
2. PATCH /v2/<name>/blobs/uploads/<uuid>  # Upload data
3. PUT /v2/<name>/blobs/uploads/<uuid>    # Complete upload
4. PUT /v2/<name>/manifests/<reference>   # Upload manifest
```

### Pull Process (процесс скачивания)

```cli
1. GET /v2/<name>/manifests/<reference>   # Get manifest
2. GET /v2/<name>/blobs/<digest>          # Download layers
3. Image assembly                         # Build image locally
```

## Аутентификация и авторизация

### Token-based Authentication

```cli
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│   Docker     │        │   Registry   │        │   Auth       │
│   Client     │        │              │        │   Server     │
│              │        │              │        │              │
│  1. Request  │        │              │        │              │
│   image pull │───────►│  2. WWW-     │───────►│  3. Validate │
│              │        │  Authenticate│        │  credentials │
│              │        │              │        │              │
│  6. Download │        │  5. Return   │        │  4. Issue    │
│    image     │◄───────│   image      │◄───────│   token      │
│              │        │              │        │              │
└──────────────┘        └──────────────┘        └──────────────┘
```

### Registry Security (безопасность)

- **TLS/SSL** - шифрование трафика
- **JWT Tokens** - JSON Web Tokens для аутентификации
- **Role-based Access Control** - контроль доступа на основе ролей
- **Content Trust** - цифровая подпись образов (DCT)

## Хранение и оптимизация

### Layer Deduplication (дедупликация слоёв)

```cli
Image A: [base-layer][app-layer][config-A]
Image B: [base-layer][app-layer][config-B]
           ↑           ↑
       одинаковые    одинаковые
        слои         слои
```

### Garbage Collection (сборка мусора)

```yaml
# registry config.yml
storage:
  delete:
    enabled: true
  maintenance:
    uploadpurging:
      enabled: true
      age: 168h
      interval: 24h
    readonly:
      enabled: false
```

## Интеграция с CI/CD

### Pipeline Integration

```cli
Develop → Build → Test → Push to Registry → Deploy
    ↓        ↓        ↓         ↓             ↓
┌────────┐┌────────┐┌────────┐┌────────────┐┌────────┐
│  Code  ││ Docker ││  Test  ││  Registry  ││  Prod  │
│        ││ Build  ││        ││   Staging  ││        │
└────────┘└────────┘└────────┘└────────────┘└────────┘
```

### Tagging Strategy (стратегия тегирования)

```bash
registry.company.com/app:
├── latest          (development)
├── v1.2.3          (release)
├── v1.2.3-staging  (staging)
├── v1.2.3-prod     (production)
└── sha-a1b2c3d     (git commit)
```

## Производительность и масштабирование

### Caching Strategies (стратегии кэширования)

- **CDN Integration** - распределённое кэширование
- **Registry Mirror** - локальное зеркалирование
- **Storage Optimization** - оптимизация бэкенда хранения

### High Availability (высокая доступность)

```cli
┌─────────────────┐    ┌─────────────────┐
│   Registry      │    │   Registry      │
│   Instance A    │    │   Instance B    │
│                 │    │                 │
│  ┌─────────────┐│    │  ┌─────────────┐│
│  │   Load      │├────┼──┤   Shared    ││
│  │  Balancer   ││    │  │   Storage   ││
│  └─────────────┘│    │  └─────────────┘│
└─────────────────┘    └─────────────────┘
```

**Архитектурный принцип**: Docker Registry обеспечивает централизованное, безопасное и эффективное хранение и распределение Docker-образов, поддерживая различные сценарии использования от индивидуальной разработки до enterprise-инфраструктур с тысячами пользователей.
