# Docker Architecture: Volumes

**Docker Volumes** — это механизм persistence (сохранения данных), позволяющий сохранять и разделять данные между контейнерами и хост-системой, независимо от жизненного цикла контейнеров.

## Архитектура томов Docker

### Модель хранения данных

```cli
┌─────────────────────────────────────────────────┐
│               Docker Host                       │
│                                                 │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │ Container A │        │   Docker Volume     │ │
│  │             │        │                     │ │
│  │ /app/data   │───────►│  /var/lib/docker/   │ │
│  │             │        │   volumes/my-data   │ │
│  └─────────────┘        └─────────────────────┘ │
│                                                 │
│  ┌─────────────┐                 │              │
│  │ Container B │                 │              │
│  │             │                 │              │
│  │ /app/data   │─────────────────┘              │
│  │             │                                │
│  └─────────────┘                                │
└─────────────────────────────────────────────────┘
```

## Типы томов Docker

### 1. Named Volumes (именованные тома)

```cli
┌─────────────────────────────────────────────────┐
│               Docker Host                       │
│                                                 │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │ Container   │        │   Named Volume      │ │
│  │             │        │                     │ │
│  │ /data       │───────►│    db-data          │ │
│  │             │        │                     │ │
│  └─────────────┘        └─────────────────────┘ │
│                          /var/lib/docker/       │
│                          volumes/db-data/_data  │
└─────────────────────────────────────────────────┘
```

### 2. Bind Mounts (bind-монтирования)

```cli
┌─────────────────────────────────────────────────┐
│               Docker Host                       │
│                                                 │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │ Container   │        │   Host Directory    │ │
│  │             │        │                     │ │
│  │ /app/logs   │───────►│  /home/user/logs    │ │
│  │             │        │                     │ │
│  └─────────────┘        └─────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### 3. tmpfs Mounts (временные файловые системы)

```cli
┌─────────────────────────────────────────────────┐
│               Docker Host                       │
│                                                 │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │ Container   │        │   tmpfs in RAM      │ │
│  │             │        │                     │ │
│  │ /tmp        │───────►│  Memory Storage     │ │
│  │             │        │                     │ │
│  └─────────────┘        └─────────────────────┘ │
└─────────────────────────────────────────────────┘
```

## Архитектура Named Volumes

### Структура хранения

```bash
/var/lib/docker/volumes/
├── my-volume/
│   ├── _data/
│   │   ├── database.db
│   │   └── config.json
│   └── metadata.db
├── db-data/
│   └── _data/
│       └── mysql/
└── app-logs/
    └── _data/
        └── application.log
```

### Компоненты volume-системы

- **Volume Driver** (драйвер томов) - управляет созданием и работой томов
- **Volume Plugin** (плагины томов) - расширения для внешних систем хранения
- **Metadata Database** (база метаданных) - хранит информацию о томах
- **Mount Point** (точка монтирования) - связь между контейнером и томом

## Volume Drivers (драйверы томов)

### Встроенные драйверы

- **local** (локальный) - хранение на локальной файловой системе
- **tmpfs** - хранение в оперативной памяти

### Подключаемые драйверы

```cli
┌─────────────────────────────────────────────────┐
│               Docker Host                       │
│                                                 │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │ Container   │        │   Volume Plugin     │ │
│  │             │        │                     │ │
│  │ /data       │───────►│   (NFS, Ceph, AWS)  │─────► External Storage
│  │             │        │                     │ │
│  └─────────────┘        └─────────────────────┘ │
└─────────────────────────────────────────────────┘
```

**Примеры внешних драйверов:**

- **nfs** - Network File System
- **cifs** - Common Internet File System
- **aws** - Amazon EBS/EFS
- **ceph** - распределённое хранилище
- **portworx** - облачное нативное хранилище

## Механизм работы томов

### Процесс монтирования

1. **Volume Creation** (создание тома)

   ```bash
   docker volume create my-data
   ```

2. **Mount Propagation** (распространение монтирования):

   ```cli
   Container Process → Mount Namespace → Host Filesystem → Volume Storage
   ```

3. **Data Flow** (поток данных)

   ```cli
   Application → Container FS → VFS → Volume Driver → Storage Backend
   ```

### Copy-on-Write с томами

```cli
Контейнер запускается с томом:
[Image Layers] + [Writable Layer] + [Volume Mount]

Чтение из тома: прямое обращение к volume storage
Запись в том: прямое изменение в volume storage
Изменения в томе: не затрагивают образ контейнера
```

## Volume Lifecycle (жизненный цикл томов)

### States (состояния)

```cli
created → mounted → in-use → unmounted → removed
    ↑         │         │         │         │
    └─────────┴─────────┴─────────┴─────────┘
           возможные переходы между состояниями
```

### Операции управления

- **Create** - создание тома
- **Inspect** - просмотр информации
- **Prune** - удаление неиспользуемых томов
- **Remove** - удаление тома
- **Backup** - резервное копирование данных

## Использование в Docker Compose

### Конфигурация томов

```yaml
version: "3.8"
services:
  database:
    image: postgres:13
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./backups:/backups # bind mount

  application:
    image: my-app:latest
    volumes:
      - app-logs:/app/logs
    tmpfs:
      - /tmp

volumes:
  db-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /opt/db-data

  app-logs:
    driver: local
```

## Volume Security (безопасность томов)

### Модель разрешений

- **File Ownership** (владение файлами) - UID/GID маппинг между контейнером и хостом
- **SELinux/AppArmor** - мандатный контроль доступа
- **Read-only Mounts** (монтирование только для чтения) - защита от изменений

### Безопасные практики

```bash
# Read-only монтирование
docker run -v my-data:/app/data:ro my-app

# Настройка владельца
docker run -v my-data:/app/data -u 1000:1000 my-app

# Приватные тома
docker volume create --driver local \
  --opt type=tmpfs \
  --opt device=tmpfs \
  private-temp
```

## Производительность и оптимизация

### Выбор типа монтирования

| Тип              | Производительность | Persistence | Изоляция |
| ---------------- | ------------------ | ----------- | -------- |
| **Named Volume** | Высокая            | ✅          | ✅       |
| **Bind Mount**   | Зависит от FS      | ✅          | ❌       |
| **tmpfs**        | Максимальная       | ❌          | ✅       |

### Оптимизации

- **Volume Drivers** - выбор подходящего драйвера для workload
- **Caching Strategies** - стратегии кэширования для часто читаемых данных
- **Storage Backend** - выбор бэкенда хранения (SSD, NVMe, network storage)

## Распределённые тома

### Multi-host volumes

```cli
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Node 1        │    │   Node 2        │    │   Storage       │
│                 │    │                 │    │   Backend       │
│  ┌─────────────┐│    │  ┌─────────────┐│    │                 │
│  │ Container   ││    │  │ Container   ││    │  ┌─────────────┐│
│  │             ││    │  │             ││    │  │ Distributed ││
│  │ /shared/data│├────┼──┤ /shared/data│├────┼──┤   Volume    ││
│  │             ││    │  │             ││    │  │             ││
│  └─────────────┘│    │  └─────────────┘│    │  └─────────────┘│
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

**Архитектурный принцип**: Docker Volumes обеспечивают отделение данных от контейнеров, позволяя сохранять состояние приложений, разделять данные между сервисами и интегрироваться с различными системами хранения, обеспечивая гибкость и надёжность persistence layer.
