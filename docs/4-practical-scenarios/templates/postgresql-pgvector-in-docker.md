# PostgreSQL + pgvector в Docker

Простая настройка PostgreSQL с расширением pgvector для работы с векторными данными.

## 🎯 Задача

Запустить PostgreSQL с поддержкой векторных операций через расширение pgvector.

## 🐳 Базовая команда запуска

```bash
docker run -d \
  --name postgres-vector \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -p 5432:5432 \
  pgvector/pgvector:pg15
```

## 📁 Структура проекта

```
postgres-vector/
├── docker-compose.yml
└── init.sql
```

## 🚀 Простой docker-compose.yml

```yaml
version: "3.8"

services:
  postgres:
    image: pgvector/pgvector:pg15
    environment:
      POSTGRES_DB: vectordb
      POSTGRES_USER: vectoruser
      POSTGRES_PASSWORD: vectorpass
    ports:
      - "5432:5432"
    volumes:
      - vector_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

volumes:
  vector_data:
```

## 💡 Инициализация с pgvector

### init.sql

```sql
-- Включаем расширение pgvector
CREATE EXTENSION IF NOT EXISTS vector;

-- Создаем таблицу для векторных embeddings
CREATE TABLE IF NOT EXISTS documents (
    id BIGSERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding vector(1536), -- размерность векторов (например, для OpenAI)
    metadata JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Создаем индекс для поиска по схожести
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Пример данных
INSERT INTO documents (content, embedding, metadata) VALUES
('Перший документ', '[0.1, 0.2, 0.3]', '{"category": "docs"}'),
('Другий документ', '[0.4, 0.5, 0.6]', '{"category": "news"}');
```

## 🔍 Примеры использования

### Подключение и тестирование

```bash
# Подключение к базе
docker exec -it postgres-vector psql -U vectoruser -d vectordb
```

### SQL запросы с векторами

```sql
-- Поиск похожих векторов
SELECT
    id,
    content,
    embedding <=> '[0.1, 0.2, 0.3]' as distance
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, 0.3]'
LIMIT 5;

-- Создание индекса для больших datasets
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);
```

## 🛠️ Полезные команды

```bash
# Запуск
docker-compose up -d

# Просмотр логов
docker logs postgres-vector

# Бэкап
docker exec postgres-vector pg_dump -U vectoruser vectordb > vector_backup.sql

# Остановка
docker-compose down
```

## 🔧 Кастомный Dockerfile (опционально)

```dockerfile
FROM postgres:15-alpine

# Устанавливаем pgvector
RUN apk add --no-cache git build-base postgresql-dev
RUN git clone https://github.com/pgvector/pgvector.git /tmp/pgvector
RUN cd /tmp/pgvector && make && make install

# Очистка
RUN apk del git build-base
RUN rm -rf /tmp/pgvector
```

**Готово!** PostgreSQL с pgvector готов для работы с векторными embedding.
