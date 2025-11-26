# Docker Automated Deployment with Ansible for .NET 7

**Ansible** — это инструмент для автоматизации развертывания и управления конфигурациями, идеально подходящий для установки и настройки Docker окружения для .NET 7 приложений.

## Архитектура автоматизированного развертывания для .NET 7

### Схема развертывания

```bash
┌─────────────────┐    Ansible SSH     ┌─────────────────┐
│   Control Node  │───────────────────►│  Target Host    │
│                 │                    │                 │
│  ┌─────────────┐│                    │  ┌─────────────┐│
│  │ Ansible     ││                    │  │   Docker    ││
│  │ Playbooks   ││                    │  │   Engine    ││
│  └─────────────┘│                    │  └─────────────┘│
│                 │                    │         │       │
│  ┌─────────────┐│                    │  ┌─────────────┐│
│  │ Inventory   ││                    │  │  .NET 7     ││
│  │   File      ││                    │  │  Registry   ││
│  └─────────────┘│                    │  │  Container  ││
└─────────────────┘                    └─────────────────┘
```

## Установка Ansible

### Подготовка управляющего узла

```bash
# Обновление пакетного менеджера
sudo apt update

# Установка Ansible
sudo apt install -y ansible

# Проверка установки
ansible --version
```

## Структура проекта Ansible для .NET 7

### Организация директорий

```bash
/projects/ansible/
├── playbooks/
│   ├── dotnet-docker.yml
│   └── inventory
├── roles/
│   ├── docker/
│   ├── dotnet/
│   └── registry/
├── templates/
│   ├── docker-daemon.json
│   └── dotnet-dockerfile.j2
└── files/
    └── dotnet-app/
```

## Плейбук установки Docker и .NET 7 Registry

### Основной плейбук: `dotnet-docker.yml`

```yaml
---
- name: Установка Docker и локального registry для .NET 7
  hosts: localhost
  become: yes
  vars:
    registry_port: 5000
    registry_name: "localhost:{{ registry_port }}"
    dotnet_version: "7.0"
    test_image: "mcr.microsoft.com/dotnet/samples:aspnetapp"
    registry_data_dir: "/opt/registry-data"
    dotnet_app_dir: "/opt/dotnet-apps"

  tasks:
    # Подготовка системы
    - name: Обновление индекса пакетов
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Установка системных зависимостей для .NET
      apt:
        name:
          - wget
          - software-properties-common
          - python3-pip
          - net-tools
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present

    # Установка .NET 7 SDK
    - name: Добавление репозитория Microsoft
      shell: |
        wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
        dpkg -i packages-microsoft-prod.deb
        rm packages-microsoft-prod.deb

    - name: Установка .NET 7 SDK
      apt:
        name: dotnet-sdk-7.0
        state: present
        update_cache: yes

    - name: Проверка установки .NET
      command: dotnet --version
      register: dotnet_version
      changed_when: false

    - name: Отображение версии .NET
      debug:
        msg: "Установлен .NET {{ dotnet_version.stdout }}"

    # Установка Python модулей для Docker
    - name: Установка docker-py модуля
      pip:
        name:
          - docker
          - docker-compose
        state: present

    # Очистка конфликтующих пакетов
    - name: Удаление конфликтующих Docker пакетов
      apt:
        name:
          - docker.io
          - docker-doc
          - podman-docker
          - containerd
          - runc
        state: absent
        purge: yes
      ignore_errors: yes

    # Настройка репозитория Docker
    - name: Добавление GPG ключа Docker
      apt_key:
        url: "https://download.docker.com/linux/ubuntu/gpg"
        state: present

    - name: Добавление Docker APT репозитория
      apt_repository:
        repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        state: present
        filename: "docker"

    # Установка Docker Engine
    - name: Установка Docker компонентов
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present
        update_cache: yes

    # Настройка Docker сервиса
    - name: Запуск и включение Docker сервиса
      systemd:
        name: docker
        state: started
        enabled: yes
        daemon_reload: yes

    - name: Добавление пользователя в группу docker
      user:
        name: "{{ ansible_user_id }}"
        groups: docker
        append: yes

    - name: Ожидание готовности Docker демона
      wait_for:
        path: /var/run/docker.sock
        state: present
        timeout: 60

    # Подготовка директорий для .NET приложений
    - name: Создание директории для .NET приложений
      file:
        path: "{{ dotnet_app_dir }}"
        state: directory
        owner: "{{ ansible_user_id }}"
        group: docker
        mode: "0755"

    - name: Создание директории для данных registry
      file:
        path: "{{ registry_data_dir }}"
        state: directory
        owner: "{{ ansible_user_id }}"
        group: docker
        mode: "0755"

    # Создание тестового .NET 7 приложения
    - name: Создание тестового .NET 7 Web API
      command: "dotnet new webapi -n TestApi -o {{ dotnet_app_dir }}/test-api --force"
      args:
        creates: "{{ dotnet_app_dir }}/test-api/Program.cs"

    - name: Настройка Dockerfile для .NET 7 приложения
      template:
        src: dotnet-dockerfile.j2
        dest: "{{ dotnet_app_dir }}/test-api/Dockerfile"

    # Запуск Docker Registry
    - name: Запуск Docker Registry контейнера
      docker_container:
        name: dotnet-registry
        image: registry:2
        state: started
        restart_policy: unless-stopped
        ports:
          - "{{ registry_port }}:5000"
        volumes:
          - "{{ registry_data_dir }}:/var/lib/registry"
        env:
          REGISTRY_STORAGE_DELETE_ENABLED: "true"

    - name: Ожидание запуска registry
      wait_for:
        host: localhost
        port: "{{ registry_port }}"
        timeout: 30
        state: started

    # Сборка и публикация .NET 7 образа
    - name: Сборка Docker образа .NET 7 приложения
      docker_image:
        path: "{{ dotnet_app_dir }}/test-api"
        name: "test-dotnet-api"
        tag: "v1"
        state: present
        build:
          pull: yes

    - name: Тегирование .NET образа для локального registry
      command: "docker tag test-dotnet-api:v1 {{ registry_name }}/test-dotnet-api:v1"

    - name: Отправка .NET образа в локальный registry
      command: "docker push {{ registry_name }}/test-dotnet-api:v1"

    # Запуск тестового .NET контейнера
    - name: Запуск .NET 7 контейнера из registry
      docker_container:
        name: test-dotnet-app
        image: "{{ registry_name }}/test-dotnet-api:v1"
        state: started
        restart_policy: unless-stopped
        ports:
          - "8080:8080"
        env:
          ASPNETCORE_ENVIRONMENT: "Development"
          ASPNETCORE_URLS: "http://+:8080"

    - name: Ожидание запуска .NET приложения
      wait_for:
        host: localhost
        port: 8080
        timeout: 60
        state: started

    # Проверки и валидация
    - name: Проверка содержимого registry
      uri:
        url: "http://{{ registry_name }}/v2/_catalog"
        method: GET
      register: registry_catalog

    - name: Отображение содержимого registry
      debug:
        msg: "Registry содержит: {{ registry_catalog.json }}"

    - name: Проверка работы .NET приложения
      uri:
        url: "http://localhost:8080/weatherforecast"
        method: GET
        status_code: 200
      register: app_test

    - name: Отображение статуса .NET приложения
      debug:
        msg: ".NET приложение успешно запущено: {{ app_test.status }}"

    # Финальные проверки
    - name: Проверка версии Docker
      command: docker --version
      register: docker_version

    - name: Отображение версии Docker
      debug:
        msg: "Установлен: {{ docker_version.stdout }}"

    - name: Проверка работающих контейнеров
      command: docker ps --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}"
      register: running_containers

    - name: Отображение статуса контейнеров
      debug:
        msg: "Запущенные контейнеры:\n{{ running_containers.stdout }}"
```

## Шаблон Dockerfile для .NET 7: `dotnet-dockerfile.j2`

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY ["TestApi.csproj", "."]
RUN dotnet restore "./TestApi.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "TestApi.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "TestApi.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "TestApi.dll"]
```

## Запуск развертывания

### Создание inventory файла

```bash
# Создание inventory файла
echo "localhost ansible_connection=local" > inventory

# Для удаленных хостов
echo "[dotnet_hosts]" > inventory
echo "192.168.1.100 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa" >> inventory
```

### Выполнение плейбука

```bash
# Локальное выполнение
ansible-playbook -i inventory dotnet-docker.yml

# С повышенной детализацией
ansible-playbook -i inventory dotnet-docker.yml -v

# Только установка Docker
ansible-playbook -i inventory dotnet-docker.yml --tags "docker"

# Только развертывание .NET приложения
ansible-playbook -i inventory dotnet-docker.yml --tags "dotnet"
```

## Управление .NET 7 Registry

### Проверка состояния registry

```bash
# Просмотр всех .NET образов в registry
curl http://localhost:5000/v2/_catalog

# Просмотр тегов .NET образа
curl http://localhost:5000/v2/test-dotnet-api/tags/list

# Проверка здоровья .NET приложения
curl http://localhost:8080/health
```

### Мониторинг .NET контейнеров

```bash
# Просмотр логов .NET приложения
docker logs test-dotnet-app

# Проверка метрик .NET приложения
curl http://localhost:8080/metrics

# Мониторинг ресурсов
docker stats test-dotnet-app
```

## SSH туннель для удаленного доступа к .NET registry

### Создание безопасного туннеля

```bash
ssh -N -T \
  -o TCPKeepAlive=yes \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -o Compression=no \
  -c aes128-ctr \
  -o IPQoS=throughput \
  -L 5000:localhost:5000 \
  -L 8080:localhost:8080 \
  user@remote-host
```

### Использование туннелированного registry для .NET

```bash
# Тегирование .NET образов для удаленного registry
docker tag my-dotnet-app:latest localhost:5000/my-dotnet-app:v1.0

# Отправка через туннель
docker push localhost:5000/my-dotnet-app:v1.0

# Скачивание с удаленного хоста
docker pull localhost:5000/my-dotnet-app:v1.0

# Тестирование .NET приложения через туннель
curl http://localhost:8080/weatherforecast
```

## Расширенная конфигурация для .NET 7

### Настройка мониторинга и логирования

```yaml
- name: Настройка OpenTelemetry для .NET приложения
  lineinfile:
    path: "{{ dotnet_app_dir }}/test-api/appsettings.Development.json"
    line: '  "OpenTelemetry": { "Endpoint": "http://localhost:4317" }'
    insertafter: '"Logging": {'

- name: Настройка health checks
  copy:
    content: |
      app.MapHealthChecks("/health");
      app.MapMetrics();
    dest: "{{ dotnet_app_dir }}/test-api/Program.cs"
    backup: yes
```

**Архитектурный принцип**: Ansible обеспечивает идемпотентное, воспроизводимое развертывание Docker инфраструктуры, оптимизированной для .NET 7 приложений, с полной интеграцией сборки, регистрации и развертывания контейнеризированных .NET сервисов.
