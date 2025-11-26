# Docker Daemon Configuration

**Docker Daemon** (`dockerd`) — это фоновый процесс, который управляет Docker-контейнерами, образами, сетями и томами. Правильная настройка демона критически важна для производительности, безопасности и стабильности Docker-окружения.

## Архитектура Docker Daemon

### Компоненты системы

```bash
┌─────────────────────────────────────────────────┐
│               Docker Daemon (dockerd)           │
│                                                 │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │  API Server │        │   Container Runtime │ │
│  │             │        │                     │ │
│  │ REST API    │◄──────►│   (containerd)      │ │
│  │ Endpoints   │        │                     │ │
│  └─────────────┘        └─────────────────────┘ │
│           │                      │              │
│           ▼                      ▼              │
│  ┌─────────────┐        ┌─────────────────────┐ │
│  │  Storage    │        │     Networking      │ │
│  │  Drivers    │        │                     │ │
│  │             │        │   Network Drivers   │ │
│  └─────────────┘        └─────────────────────┘ │
│                                                 │
└─────────────────────────────────────────────────┘
```

## Конфигурационные файлы

### Основные расположения

```bash
# Основной конфигурационный файл
/etc/docker/daemon.json

# Systemd unit файл
/lib/systemd/system/docker.service
/etc/systemd/system/docker.service.d/override.conf

# Логи демона
/var/log/docker.log
journalctl -u docker.service
```

## Конфигурация через daemon.json

### Базовая конфигурация

```json
{
  "debug": false,
  "log-level": "info",
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"],
  "storage-driver": "overlay2",
  "data-root": "/var/lib/docker",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "iptables": true,
  "live-restore": true
}
```

### Расширенная конфигурация для продакшена

```json
{
  "debug": false,
  "log-level": "warn",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "labels": "production"
  },
  "storage-driver": "overlay2",
  "storage-opts": ["overlay2.override_kernel_check=true"],
  "data-root": "/opt/docker",
  "exec-opts": ["native.cgroupdriver=systemd", "native.cgroupdriver=cgroupfs"],
  "bip": "172.17.0.1/16",
  "fixed-cidr": "172.17.0.0/24",
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    },
    "nproc": {
      "Name": "nproc",
      "Hard": 64000,
      "Soft": 64000
    }
  },
  "live-restore": true,
  "max-concurrent-downloads": 3,
  "max-concurrent-uploads": 3,
  "registry-mirrors": [
    "https://mirror.gcr.io",
    "https://docker-mirror.example.com"
  ],
  "insecure-registries": ["myregistry.example.com:5000"],
  "metrics-addr": "0.0.0.0:9323",
  "experimental": false
}
```

## Настройки производительности

### Оптимизация для высоконагруженных систем

```json
{
  "storage-driver": "overlay2",
  "storage-opts": ["overlay2.size=20G", "overlay2.override_kernel_check=true"],
  "log-driver": "local",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5",
    "compress": "true"
  },
  "max-concurrent-downloads": 5,
  "max-concurrent-uploads": 5,
  "max-download-attempts": 5,
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  }
}
```

## Безопасность демона

### Безопасная конфигурация

```json
{
  "tls": true,
  "tlscacert": "/etc/docker/ca.pem",
  "tlscert": "/etc/docker/server-cert.pem",
  "tlskey": "/etc/docker/server-key.pem",
  "tlsverify": true,
  "userns-remap": "default",
  "no-new-privileges": true,
  "icc": false,
  "userland-proxy": false,
  "seccomp-profile": "/etc/docker/seccomp/default.json"
}
```

### Настройка пользовательских namespace

```json
{
  "userns-remap": "default",
  "userns-remap": "dockremap:dockremap"
}
```

## Сетевые настройки

### Конфигурация сети

```json
{
  "bip": "10.0.0.1/24",
  "fixed-cidr": "10.0.0.0/25",
  "default-gateway": "10.0.0.254",
  "mtu": 1500,
  "dns": ["8.8.8.8", "8.8.4.4"],
  "dns-opts": ["timeout:2", "attempts:3"],
  "bridge": "none",
  "ipv6": false,
  "ip6tables": false
}
```

### Multiple daemon hosts

```json
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://192.168.1.100:2376", "fd://"],
  "tls": true,
  "tlscacert": "/etc/docker/ca.pem",
  "tlscert": "/etc/docker/server-cert.pem",
  "tlskey": "/etc/docker/server-key.pem"
}
```

## Настройка через Systemd

### Docker service unit файл

```ini
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service containerd.service
Wants=network-online.target
Requires=docker.socket containerd.service

[Service]
Type=notify
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutStartSec=0
RestartSec=2
Restart=always
StartLimitBurst=3
StartLimitInterval=60s
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process
OOMScoreAdjust=-500

[Install]
WantedBy=multi-user.target
```

### Кастомные systemd настройки

```bash
# Создание override конфигурации
sudo systemctl edit docker.service

# Добавление в override.conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// --config-file=/etc/docker/daemon.json
Environment="HTTP_PROXY=http://proxy.example.com:3128"
Environment="HTTPS_PROXY=http://proxy.example.com:3128"
Environment="NO_PROXY=localhost,127.0.0.1,.example.com"
LimitNOFILE=65536
LimitMEMLOCK=infinity
```

## Мониторинг и метрики

### Включение метрик Prometheus

```json
{
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true
}
```

### Конфигурация логов

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "labels": "production",
    "env": "os,customer"
  }
}
```

## Настройки для различных storage драйверов

### Overlay2

```json
{
  "storage-driver": "overlay2",
  "storage-opts": ["overlay2.override_kernel_check=true", "overlay2.size=20G"]
}
```

### Btrfs

```json
{
  "storage-driver": "btrfs",
  "storage-opts": ["btrfs.min_space=1G"]
}
```

### ZFS

```json
{
  "storage-driver": "zfs",
  "storage-opts": ["zfs.fsname=zpool/docker"]
}
```

## Ресурсные ограничения

### Настройки ресурсов

```json
{
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    },
    "nproc": {
      "Name": "nproc",
      "Hard": 65536,
      "Soft": 65536
    },
    "core": {
      "Name": "core",
      "Hard": -1,
      "Soft": -1
    }
  },
  "cpu-rt-period": 1000000,
  "cpu-rt-runtime": 950000
}
```

## Реестры и mirrors

### Настройка registry mirrors

```json
{
  "registry-mirrors": [
    "https://mirror.gcr.io",
    "https://docker-mirror.company.com",
    "https://registry-1.docker.io"
  ],
  "insecure-registries": ["registry.company.com:5000", "192.168.1.100:5000"]
}
```

## Практические сценарии

### Разработка

```json
{
  "debug": true,
  "log-level": "debug",
  "storage-driver": "overlay2",
  "experimental": true,
  "features": {
    "buildkit": true
  }
}
```

### Продакшен

```json
{
  "debug": false,
  "log-level": "warn",
  "storage-driver": "overlay2",
  "live-restore": true,
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  },
  "max-concurrent-downloads": 3,
  "max-concurrent-uploads": 3
}
```

### High Availability

```json
{
  "cluster-store": "consul://192.168.1.100:8500",
  "cluster-advertise": "192.168.1.101:2376",
  "cluster-store-opts": {
    "kv.path": "docker/nodes"
  }
}
```

## Управление и обслуживание

### Команды управления демоном

```bash
# Перезагрузка демона с новой конфигурацией
sudo systemctl daemon-reload
sudo systemctl restart docker

# Проверка конфигурации
docker info
docker system info

# Просмотр логов
journalctl -u docker.service -f
sudo tail -f /var/log/docker.log
```

### Валидация конфигурации

```bash
# Проверка синтаксиса daemon.json
sudo docker --config /etc/docker daemon --validate

# Тестовый запуск
sudo dockerd --validate --config-file /etc/docker/daemon.json
```

**Архитектурный принцип**: Docker Daemon является центральным компонентом Docker-экосистемы, и его правильная настройка обеспечивает стабильность, безопасность и производительность всей контейнерной платформы, адаптируясь к требованиям различных окружений от разработки до высоконагруженного продакшена.
