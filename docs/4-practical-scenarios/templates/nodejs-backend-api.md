# Node.js Backend API в Docker

Простая настройка Node.js backend API в Docker контейнере.

## 🎯 Задача

Запустить Node.js API в Docker с минимальной конфигурацией.

## 📁 Структура проекта

```
node-api/
├── src/
│   ├── app.js
│   └── routes/
├── package.json
├── Dockerfile
└── docker-compose.yml
```

## 🐳 Минимальный Dockerfile

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Копируем зависимости
COPY package*.json ./
RUN npm install

# Копируем исходный код
COPY . .

# Экспортируем порт
EXPOSE 3000

# Запускаем приложение
CMD ["node", "src/app.js"]
```

## 💡 Простой app.js

```javascript
// src/app.js
const express = require("express");
const app = express();

app.use(express.json());

// Health check
app.get("/health", (req, res) => {
  res.json({ status: "OK", timestamp: new Date().toISOString() });
});

// Простой API endpoint
app.get("/api/users", (req, res) => {
  res.json([
    { id: 1, name: "John" },
    { id: 2, name: "Jane" },
  ]);
});

app.post("/api/users", (req, res) => {
  const user = req.body;
  res.status(201).json({ id: Date.now(), ...user });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

## 📦 Простой package.json

```json
{
  "name": "node-api",
  "version": "1.0.0",
  "main": "src/app.js",
  "scripts": {
    "start": "node src/app.js",
    "dev": "nodemon src/app.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
```

## 🚀 Запуск с Docker Compose

```yaml
# docker-compose.yml
version: "3.8"

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

## 💡 Простой запуск

```bash
# Сборка
docker build -t node-api .

# Запуск
docker run -d -p 3000:3000 --name node-api node-api
```

## 🔧 Для разработки

### Dockerfile.dev

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "run", "dev"]
```

### docker-compose.dev.yml

```yaml
version: "3.8"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
```

## 🚀 Запуск

```bash
# Production
docker-compose up -d

# Development
docker-compose -f docker-compose.dev.yml up
```

**Готово!** Node.js API работает в Docker контейнере.
