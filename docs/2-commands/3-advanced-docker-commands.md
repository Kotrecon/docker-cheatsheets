# Advanced Docker Commands

Специализированные и редко используемые команды Docker для продвинутых сценариев и специфических задач.

## 🔧 Специализированные команды управления

### `docker update`

**Обновление конфигурации работающих контейнеров**

```bash
docker update [OPTIONS] CONTAINER [CONTAINER...]
```

**Опции:**

- `--cpus` - ограничение CPU
- `--memory` / `-m` - ограничение памяти
- `--restart` - политика перезапуска
- `--blkio-weight` - вес блокирующего I/O

**Примеры:**

```bash
# Обновление лимита памяти
docker update --memory 512m my_container

# Изменение ограничений CPU
docker update --cpus 1.5 web_app

# Смена политики перезапуска
docker update --restart unless-stopped database

# Обновление нескольких контейнеров
docker update --memory 1g container1 container2 container3
```

### `docker rename`

**Переименование контейнера**

```bash
docker rename CONTAINER NEW_NAME
```

**Пример:**

```bash
docker rename old_container_name new_container_name
```

### `docker pause/unpause`

**Приостановка и возобновление контейнеров**

```bash
docker pause CONTAINER [CONTAINER...]
docker unpause CONTAINER [CONTAINER...]
```

## 🔐 Команды безопасности

### `docker trust`

**Управление доверием к образам (Docker Content Trust)**

```bash
docker trust inspect IMAGE[:TAG]
docker trust sign IMAGE[:TAG]
docker trust revoke IMAGE[:TAG]
```

**Примеры:**

```bash
# Просмотр информации о доверии
docker trust inspect nginx:latest

# Подписание образа
docker trust sign myapp:production

# Отзыв подписи
docker trust revoke myapp:old-version
```

### `docker scan`

**Сканирование образов на уязвимости**

```bash
docker scan [OPTIONS] IMAGE
```

**Опции:**

- `--file` - указание Dockerfile для анализа
- `--exclude-base` - исключить базовый образ из сканирования
- `--json` - вывод в формате JSON
- `--severity` - фильтр по уровню серьезности

**Примеры:**

```bash
# Базовое сканирование
docker scan nginx:latest

# Сканирование с Dockerfile
docker scan --file Dockerfile myapp:latest

# Сканирование с фильтром
docker scan --severity=high myapp:production

# JSON вывод для автоматизации
docker scan --json myapp:latest > scan_report.json
```

## 📡 Сетевые команды

### `docker network connect/disconnect`

**Подключение и отключение контейнеров от сетей**

```bash
docker network connect [OPTIONS] NETWORK CONTAINER
docker network disconnect [OPTIONS] NETWORK CONTAINER
```

**Примеры:**

```bash
# Подключение контейнера к сети
docker network connect my_network web_container

# Отключение контейнера от сети
docker network disconnect bridge old_container

# Подключение с указанием IP-адреса
docker network connect --ip 172.20.0.100 my_network app
```

### `docker network create`

**Создание пользовательских сетей**

```bash
docker network create [OPTIONS] NETWORK
```

**Опции:**

- `--driver` / `-d` - драйвер сети (bridge, overlay, macvlan)
- `--subnet` - подсеть
- `--gateway` - шлюз
- `--ip-range` - диапазон IP-адресов
- `--label` - метки сети

**Примеры:**

```bash
# Создание bridge сети
docker network create --driver bridge my_bridge

# Создание сети с подсетью
docker network create --subnet 172.20.0.0/16 --gateway 172.20.0.1 my_network

# Создание overlay сети
docker network create --driver overlay my_overlay

# Сеть с метками
docker network create --label environment=production prod_network
```

## 💾 Команды томов

### `docker volume create`

**Создание томов с настройками**

```bash
docker volume create [OPTIONS] VOLUME
```

**Опции:**

- `--driver` / `-d` - драйвер тома
- `--opt` / `-o` - опции драйвера
- `--label` - метки тома

**Примеры:**

```bash
# Создание тома с драйвером local
docker volume create my_volume

# Том с опциями
docker volume create --opt type=nfs --opt device=:/nfs/share nfs_volume

# Том с метками
docker volume create --label backup=daily database_volume
```

### `docker volume inspect`

**Детальная информация о томе**

```bash
docker volume inspect [OPTIONS] VOLUME [VOLUME...]
```

**Пример:**

```bash
docker volume inspect my_volume
```

## 🔄 Команды для разработки

### `docker buildx`

**Расширенные возможности сборки**

```bash
docker buildx build [OPTIONS] PATH | URL | -
```

**Опции:**

- `--platform` - сборка для нескольких платформ
- `--push` - автоматическая отправка в registry
- `--load` - загрузка образа в локальный docker
- `--progress` - тип отображения прогресса

**Примеры:**

```bash
# Сборка для нескольких архитектур
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:multiarch .

# Сборка и отправка в registry
docker buildx build --platform linux/amd64 --push -t myregistry/myapp:latest .

# Сборка с прогрессом в режиме TTY
docker buildx build --progress=tty -t myapp:latest .
```

### `docker checkpoint`

**Управление контрольными точками контейнеров**

```bash
docker checkpoint create [OPTIONS] CONTAINER CHECKPOINT
docker checkpoint ls CONTAINER
docker checkpoint rm CONTAINER CHECKPOINT
```

**Примеры:**

```bash
# Создание контрольной точки
docker checkpoint create my_container before_update

# Список контрольных точек
docker checkpoint ls my_container

# Удаление контрольной точки
docker checkpoint rm my_container old_checkpoint
```

## 🐳 Команды Swarm (для оркестрации)

### `docker node`

**Управление узлами Swarm**

```bash
docker node ls
docker node inspect NODE
docker node update [OPTIONS] NODE
docker node promote/demote NODE
```

### `docker service`

**Управление сервисами Swarm**

```bash
docker service create [OPTIONS] IMAGE [COMMAND] [ARG...]
docker service ls
docker service scale SERVICE=REPLICAS
docker service update [OPTIONS] SERVICE
```

**Примеры:**

```bash
# Создание сервиса
docker service create --name web --replicas 3 -p 80:80 nginx

# Масштабирование сервиса
docker service scale web=5

# Обновление сервиса
docker service update --image nginx:latest web
```

## 🔍 Команды отладки и инспекции

### `docker events`

**Мониторинг событий Docker в реальном времени**

```bash
docker events [OPTIONS]
```

**Примеры:**

```bash
# Все события
docker events

# События за определенный период
docker events --since "2024-01-01" --until "2024-01-02"

# Фильтрация по типу события
docker events --filter type=container --filter event=die
```

### `docker logs` с расширенными опциями

```bash
# Логи с временными метками
docker logs -t container_name

# Логи с определенного времени
docker logs --since 1h container_name

# Последние N строк
docker logs --tail 100 container_name

# Логи в реальном времени с фильтрацией
docker logs -f container_name | grep "ERROR"
```

## 🛠️ Специфические сценарии

### Миграция контейнеров

```bash
# Экспорт контейнера
docker export container_name > container_backup.tar

# Импорт контейнера
docker import container_backup.tar new_image:tag

# Сохранение образа
docker save -o image_backup.tar image_name:tag

# Загрузка образа
docker load -i image_backup.tar
```

### Работа с registry

```bash
# Логин в registry
docker login myregistry.example.com

# Просмотр образов в registry
docker search nginx

# Просмотр тегов образа
docker images --filter=reference='nginx:*'

# Очистка локального кеша registry
docker system prune -a --filter until=168h
```

### Управление контекстами

```bash
# Список контекстов
docker context ls

# Создание контекста
docker context create remote --docker host=ssh://user@server

# Переключение контекста
docker context use remote

# Просмотр текущего контекста
docker context inspect
```

## ⚡ Автоматизация и скрипты

### Полезные команды для скриптов

```bash
# Получить IP-адрес контейнера
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name

# Проверить здоровье контейнера
docker inspect --format='{{.State.Health.Status}}' container_name

# Получить порты контейнера
docker port container_name

# Проверить, запущен ли контейнер
docker inspect -f '{{.State.Running}}' container_name
```

### Мониторинг и алерты

```bash
# Скрипт для проверки состояния контейнеров
if [ $(docker ps -q | wc -l) -eq 0 ]; then
    echo "No containers running!" | mail -s "Docker Alert" admin@example.com
fi

# Проверка использования ресурсов
if docker stats --no-stream --format "{{.MemPerc}}" container_name | grep -q "95"; then
    echo "High memory usage!" >&2
fi
```

## 🎯 Когда использовать эти команды

- **`docker update`** - динамическое изменение ресурсов без перезапуска
- **`docker scan`** - безопасность в CI/CD пайплайнах
- **`docker buildx`** - мульти-архитектурные сборки
- **`docker trust`** - продакшен окружения с подписанными образами
- **`docker checkpoint`** - миграция stateful приложений
- **`docker events`** - мониторинг и аудит

Эти команды редко нужны в повседневной работе, но незаменимы в специфических сценариях и продвинутых конфигурациях.

---
