# Nginx Reverse Proxy в Docker

Простая и эффективная настройка Nginx как reverse proxy в Docker.

## 🎯 Задача

Настроить Nginx в качестве reverse proxy для перенаправления трафика на backend-приложения.

## 📁 Структура проекта

```bash
nginx-proxy/
├── nginx.conf
├── Dockerfile
└── docker-compose.yml
```

## 🐳 Минимальный Dockerfile

```dockerfile
FROM nginx:alpine

# Копируем конфиг
COPY nginx.conf /etc/nginx/nginx.conf

# Экспортируем порт
EXPOSE 80

# Запускаем Nginx
CMD ["nginx", "-g", "daemon off;"]
```

## ⚙️ Базовый nginx.conf

```nginx
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server api1:8080;
        server api2:8080;
    }

    upstream frontend {
        server webapp:3000;
    }

    server {
        listen 80;
        server_name localhost;

        # Frontend
        location / {
            proxy_pass http://frontend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # API
        location /api/ {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # Health check
        location /health {
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
```

## 🚀 Запуск с Docker Compose

```yaml
# docker-compose.yml
version: "3.8"

services:
  nginx:
    build: .
    ports:
      - "80:80"
    depends_on:
      - api1
      - api2
      - webapp
    networks:
      - app-network

  api1:
    image: my-api:latest
    networks:
      - app-network

  api2:
    image: my-api:latest
    networks:
      - app-network

  webapp:
    image: my-frontend:latest
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

## 💡 Простой запуск

```bash
# Сборка
docker build -t nginx-proxy .

# Запуск
docker run -d -p 80:80 --name nginx-proxy nginx-proxy
```

## 🔧 Дополнительные настройки (по необходимости)

### С SSL/TLS

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/certs/example.com.key;

    location / {
        proxy_pass http://frontend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

### С load balancing

```nginx
upstream backend {
    least_conn;
    server api1:8080 weight=3;
    server api2:8080 weight=1;
    server api3:8080 backup;
}
```

### С кэшированием

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m;

location /api/ {
    proxy_cache my_cache;
    proxy_pass http://backend;
}
```

**Всё!** Простой reverse proxy готов к работе.
