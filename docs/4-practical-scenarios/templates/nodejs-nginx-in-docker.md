# Node.js + Nginx в Docker

Простая настройка Node.js приложения с Nginx в качестве reverse proxy.

## 🎯 Задача

Запустить Node.js приложение с Nginx для раздачи статики и проксирования API.

## 📁 Структура проекта

```bash
node-nginx/
├── frontend/          # Статические файлы
├── src/               # Node.js приложение
├── nginx.conf
├── Dockerfile
└── docker-compose.yml
```

## 🐳 Простой Dockerfile

```dockerfile
FROM node:18-alpine as builder

WORKDIR /app
COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

# Финальный образ
FROM nginx:alpine

# Копируем статику
COPY --from=builder /app/frontend /usr/share/nginx/html

# Копируем конфиг Nginx
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
```

## ⚙️ Простой nginx.conf

```nginx
events {
    worker_connections 1024;
}

http {
    upstream nodeapp {
        server nodejs:3000;
    }

    server {
        listen 80;

        # Статические файлы
        location / {
            root /usr/share/nginx/html;
            try_files $uri $uri/ /index.html;
        }

        # API прокси
        location /api/ {
            proxy_pass http://nodeapp;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

## 🚀 Docker Compose

```yaml
version: "3.8"

services:
  nginx:
    build: .
    ports:
      - "80:80"
    depends_on:
      - nodejs
    networks:
      - app-network

  nodejs:
    image: node:18-alpine
    working_dir: /app
    volumes:
      - ./src:/app
    command: ["node", "server.js"]
    environment:
      - NODE_ENV=production
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

## 💡 Простое Node.js приложение

```javascript
// src/server.js
const express = require("express");
const app = express();

app.use(express.json());

app.get("/api/health", (req, res) => {
  res.json({ status: "OK", service: "Node.js API" });
});

app.get("/api/data", (req, res) => {
  res.json({ message: "Hello from Node.js API" });
});

app.listen(3000, "0.0.0.0", () => {
  console.log("Node.js server running on port 3000");
});
```

## 🚀 Запуск

```bash
# Сборка и запуск
docker-compose up -d --build

# Проверка
curl http://localhost/api/health
```

## 📦 Простой package.json

```json
{
  "name": "node-nginx-app",
  "scripts": {
    "start": "node src/server.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
```

**Готово!** Node.js приложение работает за Nginx proxy.
