# Keycloak в Docker

Полное руководство по развертыванию и управлению Keycloak в Docker для production-окружений с фокусом на безопасность, масштабируемость и интеграцию.

## 🎯 Задача

Развернуть отказоустойчивый кластер Keycloak в Docker с:

- Высокой доступностью и репликацией сессий
- Безопасной конфигурацией и SSL/TLS
- Интеграцией с внешними базами данных
- Мониторингом и бэкапами

## 📁 Структура проекта

```bash
keycloak-cluster/
├── docker-compose.yml              # Базовая установка
├── docker-compose.prod.yml         # Production кластер
├── docker-compose.db.yml           # Внешняя БД
├── .env.example                    # Переменные окружения
├── config/
│   ├── keycloak.conf              # Основная конфигурация
│   ├── cache-ispn.xml             # Кэширование
│   ├── themes/                    # Кастомные темы
│   └── ssl/                       # SSL сертификаты
├── scripts/
│   ├── setup-realm.sh             # Настройка realm
│   ├── create-users.sh            # Создание пользователей
│   └── export-import.sh           # Экспорт/импорт
└── monitoring/
    ├── prometheus.yml
    └── grafana-dashboards/
```

## 🐳 Docker конфигурация

### Базовая установка (Development)

```yaml
# docker-compose.yml
version: "3.8"

services:
  keycloak:
    image: quay.io/keycloak/keycloak:23.0.0
    container_name: keycloak
    command:
      [
        "start-dev",
        "--http-enabled=true",
        "--hostname-strict=false",
        "--hostname-strict-https=false",
      ]
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=admin
      - KC_HEALTH_ENABLED=true
      - KC_METRICS_ENABLED=true
    ports:
      - "8080:8080"
    networks:
      - keycloak-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/ready"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  keycloak-network:
    driver: bridge
```

### Production с PostgreSQL

```yaml
# docker-compose.prod.yml
version: "3.8"

services:
  keycloak:
    image: quay.io/keycloak/keycloak:23.0.0
    container_name: keycloak
    command:
      [
        "start",
        "--auto-build",
        "--http-enabled=true",
        "--https-port=8443",
        "--hostname-strict=false",
        "--hostname-strict-https=false",
        "--db=postgres",
        "--db-url=jdbc:postgresql://keycloak-db:5432/keycloak",
        "--db-username=keycloak",
        "--db-password-file=/run/secrets/db_password",
        "--cache=ispn",
        "--cache-config-file=/opt/keycloak/conf/cache-ispn.xml",
        "--features=token-exchange,admin-fine-grained-authz",
      ]
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD_FILE=/run/secrets/admin_password
      - KC_PROXY=edge
      - KC_HOSTNAME=keycloak.example.com
      - KC_HTTP_ENABLED=true
      - KC_HTTPS_PORT=8443
      - KC_HEALTH_ENABLED=true
      - KC_METRICS_ENABLED=true
      - JAVA_OPTS=-Xms512m -Xmx1024m -XX:MetaspaceSize=96M -XX:MaxMetaspaceSize=256m -Djava.net.preferIPv4Stack=true -Djava.awt.headless=true
    ports:
      - "8080:8080"
      - "8443:8443"
    volumes:
      - ./config/keycloak.conf:/opt/keycloak/conf/keycloak.conf
      - ./config/cache-ispn.xml:/opt/keycloak/conf/cache-ispn.xml
      - ./config/themes:/opt/keycloak/themes
      - ./config/ssl:/etc/ssl/keycloak
      - keycloak_data:/opt/keycloak/data
    depends_on:
      keycloak-db:
        condition: service_healthy
    secrets:
      - admin_password
      - db_password
    networks:
      - keycloak-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/ready"]
      interval: 30s
      timeout: 10s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: "2.0"
        reservations:
          memory: 1G
          cpus: "1.0"

  keycloak-db:
    image: postgres:15-alpine
    container_name: keycloak-db
    environment:
      - POSTGRES_DB=keycloak
      - POSTGRES_USER=keycloak
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    volumes:
      - keycloak_db_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
    secrets:
      - db_password
    networks:
      - keycloak-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U keycloak -d keycloak"]
      interval: 30s
      timeout: 10s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: "1.0"

secrets:
  admin_password:
    file: ./secrets/admin_password.txt
  db_password:
    file: ./secrets/db_password.txt

volumes:
  keycloak_data:
    driver: local
  keycloak_db_data:
    driver: local

networks:
  keycloak-network:
    driver: bridge
```

### High Availability Cluster

```yaml
# docker-compose.ha.yml
version: "3.8"

services:
  keycloak-node1:
    image: quay.io/keycloak/keycloak:23.0.0
    container_name: keycloak-node1
    command:
      [
        "start",
        "--auto-build",
        "--db=postgres",
        "--db-url=jdbc:postgresql://keycloak-db:5432/keycloak",
        "--db-username=keycloak",
        "--db-password-file=/run/secrets/db_password",
        "--cache=ispn",
        "--cache-config-file=/opt/keycloak/conf/cache-ispn.xml",
        "--hostname-strict=false",
        "--hostname-strict-https=false",
      ]
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD_FILE=/run/secrets/admin_password
      - KC_PROXY=edge
      - KC_HOSTNAME=keycloak.example.com
      - KC_HTTP_ENABLED=true
      - KC_HTTPS_PORT=8443
      - KC_HEALTH_ENABLED=true
      - KC_METRICS_ENABLED=true
      - JAVA_OPTS=-Xms512m -Xmx1024m -Djboss.node.name=node1 -Djava.net.preferIPv4Stack=true -Djgroups.tcp.address=keycloak-node1 -Djgroups.tcp.port=7800 -Djgroups.bind_addr=keycloak-node1
    ports:
      - "8081:8080"
      - "8444:8443"
    volumes:
      - ./config/keycloak.conf:/opt/keycloak/conf/keycloak.conf
      - ./config/cache-ispn.xml:/opt/keycloak/conf/cache-ispn.xml
    depends_on:
      keycloak-db:
        condition: service_healthy
    secrets:
      - admin_password
      - db_password
    networks:
      - keycloak-ha-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/ready"]
      interval: 30s
      timeout: 10s
      retries: 3

  keycloak-node2:
    image: quay.io/keycloak/keycloak:23.0.0
    container_name: keycloak-node2
    command:
      [
        "start",
        "--auto-build",
        "--db=postgres",
        "--db-url=jdbc:postgresql://keycloak-db:5432/keycloak",
        "--db-username=keycloak",
        "--db-password-file=/run/secrets/db_password",
        "--cache=ispn",
        "--cache-config-file=/opt/keycloak/conf/cache-ispn.xml",
        "--hostname-strict=false",
        "--hostname-strict-https=false",
      ]
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD_FILE=/run/secrets/admin_password
      - KC_PROXY=edge
      - KC_HOSTNAME=keycloak.example.com
      - KC_HTTP_ENABLED=true
      - KC_HTTPS_PORT=8443
      - KC_HEALTH_ENABLED=true
      - KC_METRICS_ENABLED=true
      - JAVA_OPTS=-Xms512m -Xmx1024m -Djboss.node.name=node2 -Djava.net.preferIPv4Stack=true -Djgroups.tcp.address=keycloak-node2 -Djgroups.tcp.port=7800 -Djgroups.bind_addr=keycloak-node2
    ports:
      - "8082:8080"
      - "8445:8443"
    volumes:
      - ./config/keycloak.conf:/opt/keycloak/conf/keycloak.conf
      - ./config/cache-ispn.xml:/opt/keycloak/conf/cache-ispn.xml
    depends_on:
      keycloak-db:
        condition: service_healthy
    secrets:
      - admin_password
      - db_password
    networks:
      - keycloak-ha-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/ready"]
      interval: 30s
      timeout: 10s
      retries: 3

  keycloak-db:
    image: postgres:15-alpine
    container_name: keycloak-db
    environment:
      - POSTGRES_DB=keycloak
      - POSTGRES_USER=keycloak
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    volumes:
      - keycloak_db_data:/var/lib/postgresql/data
    secrets:
      - db_password
    networks:
      - keycloak-ha-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U keycloak -d keycloak"]
      interval: 30s
      timeout: 10s
      retries: 5

  load-balancer:
    image: nginx:alpine
    container_name: keycloak-lb
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./config/nginx.conf:/etc/nginx/nginx.conf
      - ./config/ssl:/etc/ssl/certs
    depends_on:
      - keycloak-node1
      - keycloak-node2
    networks:
      - keycloak-ha-network

secrets:
  admin_password:
    file: ./secrets/admin_password.txt
  db_password:
    file: ./secrets/db_password.txt

volumes:
  keycloak_db_data:
    driver: local

networks:
  keycloak-ha-network:
    driver: bridge
```

## ⚙️ Конфигурационные файлы

### keycloak.conf

```properties
# config/keycloak.conf
# Database Configuration
db=postgres
db-url=jdbc:postgresql://keycloak-db:5432/keycloak
db-username=keycloak
db-password=${KC_DB_PASSWORD}

# HTTP Configuration
http-enabled=true
http-port=8080
https-port=8443
hostname=keycloak.example.com
hostname-strict=false
hostname-strict-https=false

# Proxy Configuration
proxy=edge

# Features
features=token-exchange,admin-fine-grained-authz,scripts,account-api,account2,admin-api,authorization

# Health and Metrics
health-enabled=true
metrics-enabled=true

# Logging
log-level=INFO
log-console-output=json

# Cache Configuration
cache=ispn
cache-config-file=conf/cache-ispn.xml

# Cluster Configuration
jgroups-tcp-address=keycloak-node1
jgroups-tcp-port=7800
```

### cache-ispn.xml

```xml
<!-- config/cache-ispn.xml -->
<infinispan
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="urn:infinispan:config:11.0 http://www.infinispan.org/schemas/infinispan-config-11.0.xsd"
    xmlns="urn:infinispan:config:11.0">

    <cache-container name="keycloak">
        <transport cluster="keycloak-cluster" stack="tcp"/>

        <replicated-cache name="realms">
            <expiration interval="300000"/>
        </replicated-cache>

        <replicated-cache name="users">
            <expiration interval="300000"/>
        </replicated-cache>

        <distributed-cache name="sessions" owners="2">
            <expiration lifespan="3600000"/>
        </distributed-cache>

        <distributed-cache name="authenticationSessions" owners="2">
            <expiration lifespan="3600000"/>
        </distributed-cache>

        <distributed-cache name="offlineSessions" owners="2">
            <expiration lifespan="2592000"/>
        </distributed-cache>

        <distributed-cache name="clientSessions" owners="2">
            <expiration lifespan="3600000"/>
        </distributed-cache>

        <distributed-cache name="offlineClientSessions" owners="2">
            <expiration lifespan="2592000"/>
        </distributed-cache>

        <distributed-cache name="loginFailures" owners="2">
            <expiration lifespan="900000"/>
        </distributed-cache>

        <replicated-cache name="work"/>

        <local-cache name="authorization">
            <expiration interval="300000"/>
        </local-cache>

        <local-cache name="keys">
            <expiration interval="3600000"/>
        </local-cache>
    </cache-container>

    <jgroups>
        <stack name="tcp" extends="tcp">
            <TCP bind_addr="global"
                 bind_port="7800"
                 port_range="30"
                 recv_buf_size="20000000"
                 send_buf_size="640000"
                 sock_conn_timeout="300"
                 thread_naming_pattern="pl"
                 />
            <MPING bind_addr="global"
                   break_on_coord_rsp="true"
                   mcast_addr="${jgroups.mping.mcast_addr:239.6.6.6}"
                   mcast_port="${jgroups.mping.mcast_port:46655}"
                   num_initial_members="3"
                   />
            <MERGE3 min_interval="10000"
                    max_interval="30000"
                    />
            <FD_SOCK />
            <FD_ALL timeout="60000"
                    interval="15000"
                    timeout_check_interval="5000"
                    />
            <VERIFY_SUSPECT timeout="5000" />
            <pbcast.NAKACK2 use_mcast_xmit="false"
                            xmit_interval="1000"
                            xmit_table_num_rows="100"
                            xmit_table_msgs_per_row="2000"
                            xmit_table_max_compaction_time="30000"
                            resend_last_seqno="true"
                            />
            <UNICAST3 xmit_interval="500"
                      xmit_table_num_rows="100"
                      xmit_table_msgs_per_row="2000"
                      xmit_table_max_compaction_time="60000"
                      conn_close_timeout="0"
                      conn_expiry_timeout="0"
                      />
            <pbcast.STABLE desired_avg_gossip="50000"
                           max_bytes="400000"
                           />
            <pbcast.GMS print_local_addr="false"
                       join_timeout="3000"
                       />
            <UFC max_credits="2000000"
                 min_threshold="0.4"
                 />
            <MFC max_credits="2000000"
                 min_threshold="0.4"
                 />
            <FRAG3 />
        </stack>
    </jgroups>
</infinispan>
```

### Nginx Load Balancer Config

```nginx
# config/nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream keycloak_backend {
        least_conn;
        server keycloak-node1:8080 max_fails=3 fail_timeout=30s;
        server keycloak-node2:8080 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 80;
        server_name keycloak.example.com;

        location / {
            return 301 https://$server_name$request_uri;
        }
    }

    server {
        listen 443 ssl http2;
        server_name keycloak.example.com;

        ssl_certificate /etc/ssl/certs/keycloak.crt;
        ssl_certificate_key /etc/ssl/certs/keycloak.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;

        client_max_body_size 100M;

        location / {
            proxy_pass http://keycloak_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_connect_timeout 30s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
        }

        location /health {
            proxy_pass http://keycloak_backend;
            access_log off;
        }

        location /metrics {
            proxy_pass http://keycloak_backend;
            access_log off;
        }
    }
}
```

## 🚀 Запуск и управление

### Инициализация и настройка

```bash
# Запуск production установки
docker-compose -f docker-compose.prod.yml up -d

# Проверка статуса
docker-compose -f docker-compose.prod.yml logs -f keycloak

# Health check
curl http://localhost:8080/health/ready

# Вход в админку
open http://localhost:8080
```

### Автоматическая настройка realm

```bash
#!/bin/bash
# scripts/setup-realm.sh

# Ожидание запуска Keycloak
until curl -f -s http://localhost:8080/health/ready; do
    sleep 5
done

# Получение access token
ACCESS_TOKEN=$(curl -s -X POST \
  http://localhost:8080/realms/master/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin&password=${KEYCLOAK_ADMIN_PASSWORD}&grant_type=password&client_id=admin-cli" \
  | jq -r '.access_token')

# Создание realm
curl -s -X POST \
  http://localhost:8080/admin/realms \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "realm": "myapp",
    "enabled": true,
    "displayName": "My Application",
    "loginWithEmailAllowed": true,
    "duplicateEmailsAllowed": false
  }'

# Создание client
curl -s -X POST \
  http://localhost:8080/admin/realms/myapp/clients \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "clientId": "myapp-frontend",
    "enabled": true,
    "publicClient": true,
    "redirectUris": ["http://localhost:3000/*"],
    "webOrigins": ["http://localhost:3000"],
    "protocol": "openid-connect"
  }'
```

## 🔒 Безопасность

### Настройка SSL/TLS

```bash
#!/bin/bash
# scripts/generate-ssl.sh

# Создание приватного ключа
openssl genrsa -out ./config/ssl/keycloak.key 2048

# Создание CSR
openssl req -new -key ./config/ssl/keycloak.key -out ./config/ssl/keycloak.csr \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=keycloak.example.com"

# Создание самоподписанного сертификата (для тестирования)
openssl x509 -req -days 365 -in ./config/ssl/keycloak.csr \
  -signkey ./config/ssl/keycloak.key -out ./config/ssl/keycloak.crt

# Или использование Let's Encrypt (production)
certbot certonly --standalone -d keycloak.example.com
```

### Настройка безопасности

```bash
#!/bin/bash
# scripts/configure-security.sh

# Смена пароля администратора
docker exec -it keycloak /opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8080 \
  --realm master \
  --user admin \
  --password admin

docker exec -it keycloak /opt/keycloak/bin/kcadm.sh set-password \
  -r master \
  --username admin \
  --new-password ${NEW_ADMIN_PASSWORD}

# Настройка password policy
docker exec -it keycloak /opt/keycloak/bin/kcadm.sh update realms/master \
  -s 'passwordPolicy="length(8) and digits(1) and lowerCase(1) and upperCase(1)"'
```

## 📊 Мониторинг

### Docker Compose для мониторинга

```yaml
# docker-compose.monitoring.yml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    networks:
      - keycloak-network

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana-dashboards:/etc/grafana/provisioning/dashboards
    networks:
      - keycloak-network

  keycloak-metrics:
    image: quay.io/keycloak/keycloak:23.0.0
    command: ["start", "--metrics-enabled=true", "--health-enabled=true"]
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=${KEYCLOAK_ADMIN_PASSWORD}
    networks:
      - keycloak-network

volumes:
  prometheus_data:
  grafana_data:

networks:
  keycloak-network:
    external: true
    name: keycloak-cluster_keycloak-network
```

### Prometheus конфигурация

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "keycloak"
    static_configs:
      - targets: ["keycloak:8080"]
    metrics_path: "/metrics"
    scrape_interval: 30s
    basic_auth:
      username: "admin"
      password: "${KEYCLOAK_ADMIN_PASSWORD}"

  - job_name: "keycloak-health"
    static_configs:
      - targets: ["keycloak:8080"]
    metrics_path: "/health"
    scrape_interval: 30s
```

## 💾 Бэкапы и миграция

### Экспорт/импорт данных

```bash
#!/bin/bash
# scripts/export-import.sh

# Экспорт realm
docker exec -it keycloak /opt/keycloak/bin/kc.sh export \
  --file /tmp/myapp-realm.json \
  --realm myapp \
  --users realm_file

# Копирование файла с контейнера
docker cp keycloak:/tmp/myapp-realm.json ./backups/

# Импорт realm
docker cp ./backups/myapp-realm.json keycloak:/tmp/
docker exec -it keycloak /opt/keycloak/bin/kc.sh import \
  --file /tmp/myapp-realm.json
```

### Резервное копирование базы данных

```bash
#!/bin/bash
# scripts/backup-database.sh

# Бэкап PostgreSQL
docker exec keycloak-db pg_dump -U keycloak keycloak > ./backups/keycloak-$(date +%Y%m%d).sql

# Восстановление
cat ./backups/keycloak-20231201.sql | docker exec -i keycloak-db psql -U keycloak keycloak
```

## 🛠️ Troubleshooting

### Частые проблемы и решения

**Проблема**: Keycloak не запускается из-за БД  
**Решение**:

```bash
# Проверка подключения к БД
docker exec -it keycloak-db psql -U keycloak -d keycloak -c "SELECT 1;"

# Пересоздание БД
docker-compose down -v
docker-compose up -d
```

**Проблема**: High memory usage  
**Решение**:

```yaml
environment:
  - JAVA_OPTS=-Xms512m -Xmx1024m -XX:MetaspaceSize=96M -XX:MaxMetaspaceSize=256m
```

**Проблема**: Slow performance  
**Решение**:

```yaml
# Увеличение кэша
cache=ispn
cache-config-file=conf/cache-ispn.xml
```

### Команды диагностики

```bash
# Проверка логов
docker-compose logs -f keycloak

# Проверка здоровья
curl http://localhost:8080/health/ready

# Проверка метрик
curl http://localhost:8080/metrics

# Проверка подключения к БД
docker exec -it keycloak-db pg_isready -U keycloak -d keycloak
```

## 💡 Best Practices для Keycloak в Docker

### Безопасность

- Используйте сложные пароли для администратора
- Включите SSL/TLS в production
- Регулярно обновляйте Keycloak
- Настройте firewall rules

### Производительность

- Используйте внешнюю БД (PostgreSQL/MySQL)
- Настройте кэширование Infinispan
- Мониторьте memory usage
- Используйте connection pooling

### Надежность

- Используйте HA кластер для production
- Настройте регулярные бэкапы
- Мониторьте disk space
- Имейте disaster recovery plan

### Операционное управление

- Используйте configuration-as-code
- Автоматизируйте deployment
- Настройте мониторинг и алертинг
- Документируйте конфигурации

**Ключевой вывод**: Keycloak в Docker обеспечивает гибкость и масштабируемость для identity management. Правильно настроенный кластер с внешней БД, кэшированием и мониторингом обеспечивает высокую доступность и безопасность для production-окружений.
