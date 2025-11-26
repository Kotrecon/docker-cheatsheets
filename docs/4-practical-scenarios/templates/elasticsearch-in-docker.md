# Elasticsearch в Docker

Полное руководство по развертыванию и управлению Elasticsearch в Docker для production-окружений с фокусом на производительность, безопасность и мониторинг.

## 🎯 Задача

Развернуть отказоустойчивый кластер Elasticsearch в Docker с:

- Высокой доступностью и репликацией данных
- Безопасной конфигурацией и аутентификацией
- Мониторингом и алертингом
- Резервным копированием и восстановлением

## 📁 Структура проекта

```
elasticsearch-cluster/
├── docker-compose.yml              # Базовый кластер
├── docker-compose.prod.yml         # Production кластер
├── docker-compose.monitoring.yml   # Мониторинг
├── config/
│   ├── elasticsearch.yml           # Основная конфигурация
│   ├── jvm.options                 # JVM настройки
│   ├── log4j2.properties           # Логирование
│   └── certificates/               # SSL сертификаты
├── data/                           # Volume mounts
├── scripts/
│   ├── setup-users.sh              # Настройка пользователей
│   ├── create-index-templates.sh   # Шаблоны индексов
│   └── backup-restore.sh           # Бэкапы
└── monitoring/
    ├── prometheus.yml
    ├── alert-rules.yml
    └── grafana-dashboards/
```

## 🐳 Docker конфигурация

### Базовый Docker Compose (Single Node)

```yaml
# docker-compose.yml
version: "3.8"

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - node.name=elasticsearch
      - cluster.name=es-docker-cluster
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false # Для development
      - xpack.security.http.ssl.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es_data:/usr/share/elasticsearch/data
      - ./config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./config/jvm.options:/usr/share/elasticsearch/config/jvm.options
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - elastic
    healthcheck:
      test:
        [
          "CMD-SHELL",
          'curl -s http://localhost:9200/_cluster/health | grep -q ''"status":"green"''',
        ]
      interval: 30s
      timeout: 10s
      retries: 3

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    networks:
      - elastic
    depends_on:
      - elasticsearch
    healthcheck:
      test:
        [
          "CMD-SHELL",
          'curl -s -f http://localhost:5601/api/status | grep -q ''"overall":{"level":"available"}''',
        ]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  es_data:
    driver: local

networks:
  elastic:
    driver: bridge
```

### Production Cluster (3 Node)

```yaml
# docker-compose.prod.yml
version: "3.8"

services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-production-cluster
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.keystore.password=${ES_KEYSTORE_PASSWORD}
      - xpack.security.http.ssl.truststore.password=${ES_TRUSTSTORE_PASSWORD}
      - xpack.security.transport.ssl.enabled=true
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es01_data:/usr/share/elasticsearch/data
      - ./config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./config/jvm.options:/usr/share/elasticsearch/config/jvm.options
      - ./config/certificates:/usr/share/elasticsearch/config/certs
    ports:
      - "9200:9200"
    networks:
      - elastic-prod
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: "2.0"
        reservations:
          memory: 2G
          cpus: "1.0"
    healthcheck:
      test:
        [
          "CMD-SHELL",
          'curl -s -u elastic:${ELASTIC_PASSWORD} --insecure https://localhost:9200/_cluster/health | grep -q ''"status":"green"''',
        ]
      interval: 30s
      timeout: 10s
      retries: 5

  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-production-cluster
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es01,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es02_data:/usr/share/elasticsearch/data
      - ./config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./config/jvm.options:/usr/share/elasticsearch/config/jvm.options
      - ./config/certificates:/usr/share/elasticsearch/config/certs
    networks:
      - elastic-prod
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: "2.0"

  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-production-cluster
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es01,es02
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es03_data:/usr/share/elasticsearch/data
      - ./config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./config/jvm.options:/usr/share/elasticsearch/config/jvm.options
      - ./config/certificates:/usr/share/elasticsearch/config/certs
    networks:
      - elastic-prod
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: "2.0"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=https://es01:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD}
      - ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=/usr/share/kibana/config/certs/ca/ca.crt
    volumes:
      - ./config/certificates:/usr/share/kibana/config/certs
    ports:
      - "5601:5601"
    networks:
      - elastic-prod
    depends_on:
      - es01
      - es02
      - es03
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: "0.5"

volumes:
  es01_data:
    driver: local
  es02_data:
    driver: local
  es03_data:
    driver: local

networks:
  elastic-prod:
    driver: bridge
```

## ⚙️ Конфигурационные файлы

### elasticsearch.yml

```yaml
# config/elasticsearch.yml
cluster.name: "es-production-cluster"
node.name: ${HOSTNAME}
network.host: 0.0.0.0

# Discovery and cluster formation
discovery.seed_hosts: ["es01", "es02", "es03"]
cluster.initial_master_nodes: ["es01", "es02", "es03"]

# Security
xpack.security.enabled: true
xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/elastic-certificates.p12
  truststore.path: certs/elastic-certificates.p12
xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/elastic-certificates.p12
  truststore.path: certs/elastic-certificates.p12

# Memory and performance
bootstrap.memory_lock: true
thread_pool.write.queue_size: 1000

# Index management
action.auto_create_index: ".monitoring*,.watches,.triggered_watches,.watcher-history*,.ml*"

# Monitoring
xpack.monitoring.collection.enabled: true

# Snapshot and restore
path.repo: ["/usr/share/elasticsearch/snapshots"]
```

### jvm.options

```yaml
# config/jvm.options
-Xms2g
-Xmx2g

# G1GC for better performance
-XX:+UseG1GC
-XX:MaxGCPauseMillis=400
-XX:G1ReservePercent=25

# Heap dump on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/usr/share/elasticsearch/heapdump.hprof

# Logging
-Dlog4j2.formatMsgNoLookups=true

# Security
-Djava.security.manager=allow
```

## 🔒 Безопасность и SSL

### Генерация сертификатов

```bash
#!/bin/bash
# scripts/generate-certificates.sh

# Создание CA
docker exec -it es01 \
  elasticsearch-certutil ca --pem --out /usr/share/elasticsearch/config/certs/ca.zip

# Распаковка CA
unzip /path/to/ca.zip -d /path/to/certs/

# Создание сертификатов для узлов
docker exec -it es01 \
  elasticsearch-certutil cert --pem --ca-cert /usr/share/elasticsearch/config/certs/ca/ca.crt --ca-key /usr/share/elasticsearch/config/certs/ca/ca.key --out /usr/share/elasticsearch/config/certs/certs.zip

# Распаковка сертификатов
unzip /path/to/certs.zip -d /path/to/certs/

# Создание PKCS12 хранилища
docker exec -it es01 \
  elasticsearch-certutil http --silent --pem --in /usr/share/elasticsearch/config/certs/ca.zip --out /usr/share/elasticsearch/config/certs/elastic-certificates.p12
```

### Настройка пользователей

```bash
#!/bin/bash
# scripts/setup-users.sh

# Установка паролей для встроенных пользователей
until curl -s -u elastic:${ELASTIC_PASSWORD} --insecure https://es01:9200; do
  sleep 5
done

# Создание custom пользователей
curl -X POST -u elastic:${ELASTIC_PASSWORD} --insecure https://es01:9200/_security/user/app_user -H 'Content-Type: application/json' -d'
{
  "password": "${APP_USER_PASSWORD}",
  "roles": ["app_role"],
  "full_name": "Application User"
}'

# Создание ролей
curl -X POST -u elastic:${ELASTIC_PASSWORD} --insecure https://es01:9200/_security/role/app_role -H 'Content-Type: application/json' -d'
{
  "indices": [
    {
      "names": ["app-*"],
      "privileges": ["read", "write", "create_index"]
    }
  ]
}'
```

## 🚀 Запуск и управление

### Инициализация кластера

```bash
# Запуск production кластера
docker-compose -f docker-compose.prod.yml up -d

# Проверка статуса кластера
curl -u elastic:${ELASTIC_PASSWORD} --insecure https://localhost:9200/_cluster/health?pretty

# Проверка узлов
curl -u elastic:${ELASTIC_PASSWORD} --insecure https://localhost:9200/_cat/nodes?v

# Мониторинг индексов
curl -u elastic:${ELASTIC_PASSWORD} --insecure https://localhost:9200/_cat/indices?v
```

### Управление индексами

```bash
#!/bin/bash
# scripts/create-index-templates.sh

# Создание template для логов приложения
curl -X PUT -u elastic:${ELASTIC_PASSWORD} --insecure https://es01:9200/_index_template/app-logs -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["app-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "refresh_interval": "30s"
    },
    "mappings": {
      "properties": {
        "timestamp": { "type": "date" },
        "level": { "type": "keyword" },
        "message": { "type": "text" },
        "service": { "type": "keyword" },
        "environment": { "type": "keyword" }
      }
    }
  }
}'
```

## 💾 Резервное копирование и восстановление

### Настройка snapshot repository

```bash
#!/bin/bash
# scripts/configure-backup.sh

# Создание директории для снапшотов
mkdir -p ./snapshots

# Настройка snapshot repository
curl -X PUT -u elastic:${ELASTIC_PASSWORD} --insecure https://es01:9200/_snapshot/backup_repository -H 'Content-Type: application/json' -d'
{
  "type": "fs",
  "settings": {
    "location": "/usr/share/elasticsearch/snapshots/backup_repository",
    "compress": true
  }
}'

# Создание снапшота
curl -X PUT -u elastic:${ELASTIC_PASSWORD} --insecure https://es01:9200/_snapshot/backup_repository/snapshot_$(date +%Y%m%d_%H%M%S)?wait_for_completion=true
```

### Docker Compose с backup сервисом

```yaml
# docker-compose.backup.yml
version: "3.8"

services:
  # ... существующие сервисы elasticsearch

  elasticsearch-curator:
    image: bitnami/elasticsearch-curator:5.8
    volumes:
      - ./scripts/curator-actions.yml:/curator-actions.yml
      - ./snapshots:/snapshots
    environment:
      - ELASTICSEARCH_HOST=es01
      - ELASTICSEARCH_PORT=9200
      - ELASTICSEARCH_USER=elastic
      - ELASTICSEARCH_PASSWORD=${ELASTIC_PASSWORD}
      - ELASTICSEARCH_USE_SSL=true
    command: ["--config", "/curator-actions.yml"]
    networks:
      - elastic-prod
    deploy:
      restart_policy:
        condition: on-failure

  # Планировщик backup
  backup-scheduler:
    image: alpine:latest
    command: |
      sh -c "
        echo '0 2 * * * /usr/bin/curl -X PUT -u elastic:$${ELASTIC_PASSWORD} --insecure https://es01:9200/_snapshot/backup_repository/snapshot_$$(date +%Y%m%d)?wait_for_completion=true' >> /etc/crontabs/root &&
        crond -f"
    volumes:
      - ./scripts:/scripts
    environment:
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
    networks:
      - elastic-prod
```

## 📊 Мониторинг и алертинг

### Docker Compose для мониторинга

```yaml
# docker-compose.monitoring.yml
version: "3.8"

services:
  elasticsearch-exporter:
    image: justwatch/elasticsearch_exporter:1.6.0
    environment:
      - ES.URI=https://elastic:${ELASTIC_PASSWORD}@es01:9200
      - ES.ALL=true
      - ES.TLS-SKIP-VERIFY=true
    ports:
      - "9114:9114"
    networks:
      - elastic-prod
      - monitoring

  metricbeat:
    image: docker.elastic.co/beats/metricbeat:8.11.0
    user: root
    command: metricbeat -e -strict.perms=false
    volumes:
      - ./monitoring/metricbeat.yml:/usr/share/metricbeat/metricbeat.yml:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - elastic-prod
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--web.console.libraries=/etc/prometheus/console_libraries"
      - "--web.console.templates=/etc/prometheus/consoles"
      - "--storage.tsdb.retention.time=200h"
      - "--web.enable-lifecycle"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana-dashboards:/etc/grafana/provisioning/dashboards
    networks:
      - monitoring

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

### Prometheus конфигурация

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "elasticsearch"
    static_configs:
      - targets: ["elasticsearch-exporter:9114"]
    scrape_interval: 30s
    metrics_path: /metrics

  - job_name: "elasticsearch-nodes"
    static_configs:
      - targets: ["es01:9200", "es02:9200", "es03:9200"]
    metrics_path: /_prometheus/metrics
    basic_auth:
      username: "elastic"
      password: "${ELASTIC_PASSWORD}"
    scheme: https
    tls_config:
      insecure_skip_verify: true

rule_files:
  - "alert-rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
```

## 🛠️ Troubleshooting

### Частые проблемы и решения

**Проблема**: Elasticsearch не запускается из-за memory lock  
**Решение**:

```yaml
# В docker-compose.yml
ulimits:
  memlock:
    soft: -1
    hard: -1
```

**Проблема**: High memory usage  
**Решение**:

```yaml
environment:
  - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
```

**Проблема**: Slow cluster formation  
**Решение**:

```yaml
environment:
  - discovery.seed_hosts=es01,es02,es03
  - cluster.initial_master_nodes=es01,es02,es03
```

### Команды диагностики

```bash
# Проверка здоровья кластера
curl -u elastic:${ELASTIC_PASSWORD} --insecure https://localhost:9200/_cluster/health?pretty

# Проверка распределения шардов
curl -u elastic:${ELASTIC_PASSWORD} --insecure https://localhost:9200/_cat/shards?v

# Мониторинг производительности
curl -u elastic:${ELASTIC_PASSWORD} --insecure https://localhost:9200/_nodes/stats?pretty

# Проверка индексов
curl -u elastic:${ELASTIC_PASSWORD} --insecure https://localhost:9200/_cat/indices?v
```

## 💡 Best Practices для Elasticsearch в Docker

### Производительность

- Используйте отдельные volumes для данных
- Настройте оптимальные значения JVM heap
- Включите memory locking
- Используйте G1GC garbage collector

### Безопасность

- Всегда включайте xpack.security в production
- Используйте SSL/TLS для коммуникации между узлами
- Регулярно ротируйте пароли и сертификаты
- Настройте fine-grained access control

### Надежность

- Используйте минимум 3 узла для production
- Настройте replica shards для отказоустойчивости
- Реализуйте регулярное резервное копирование
- Мониторьте disk space и performance metrics

### Операционное управление

- Настройте centralized logging
- Используйте index templates для consistency
- Реализуйте index lifecycle management
- Настройте алертинг на critical events

**Ключевой вывод**: Elasticsearch в Docker обеспечивает гибкость и масштабируемость, но требует тщательной настройки для production использования. Правильно настроенный кластер обеспечивает высокую производительность, безопасность и отказоустойчивость.
