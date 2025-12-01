# PostgreSQL в Docker

Простая настройка PostgreSQL в Docker контейнере.

## 🎯 Задача

Запустить PostgreSQL в Docker с минимальной конфигурацией.

## 🐳 Базовая команда запуска

```bash
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -p 5432:5432 \
  postgres:15-alpine
```

## 📁 Структура проекта (для постоянного хранения)

```bash
postgres-docker/
├── docker-compose.yml
└── init.sql (опционально)
```

## 🚀 Простой docker-compose.yml

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

## 💡 Подключение к базе

```bash
# Подключение из контейнера
docker exec -it postgres psql -U myuser -d myapp

# Подключение извне
psql -h localhost -p 5432 -U myuser -d myapp
```

## 🔧 С инициализацией базы

### docker-compose с init скриптом

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

volumes:
  postgres_data:
```

### init.sql

```sql
-- Создание таблиц при первом запуске
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Начальные данные
INSERT INTO users (email, name) VALUES
('admin@example.com', 'Admin User'),
('user@example.com', 'Regular User');
```

## 🛠️ Полезные команды

```bash
# Просмотр логов
docker logs postgres

# Бэкап базы
docker exec postgres pg_dump -U myuser myapp > backup.sql

# Восстановление из бэкапа
cat backup.sql | docker exec -i postgres psql -U myuser myapp

# Остановка и удаление
docker-compose down

# Остановка с удалением данных
docker-compose down -v
```

## 🔒 Безопасность (опционально)

```yaml
environment:
  POSTGRES_DB: myapp
  POSTGRES_USER: myuser
  POSTGRES_PASSWORD: mypassword
  POSTGRES_INITDB_ARGS: "--encoding=UTF8 --lc-collate=C --lc-ctype=C"
```

**Готово!** PostgreSQL работает в Docker с сохранением данных.
