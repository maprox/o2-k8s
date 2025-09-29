# O2 Kubernetes Deployment

Kubernetes конфигурация для развертывания системы мониторинга транспорта O2 от компании Maprox.

## Описание проекта

O2 - это система мониторинга транспорта, состоящая из веб-интерфейса, сервера Node.js для обработки данных в реальном времени, статических файлов и базовой инфраструктуры (база данных, кэш, очереди сообщений).

## Архитектура

### Микросервисы

1. **Web** (`o2-web.yaml`) - Основное веб-приложение
   - Образ: `maprox/o2-web`
   - Порт: 80
   - Домен: `observer.maprox.net`

2. **Node** (`o2-node.yaml`) - Node.js сервер для реального времени
   - Образ: `maprox/node`
   - Порт: 3000 
   - Домен: `observer.maprox.net`
   - Поддержка WebSocket

3. **Static** (`o2-static.yaml`) - Статические файлы
   - Образ: `maprox/static`
   - Порт: 80
   - Домен: `static.maprox.net`

### Обработчики данных (handlers/)

1. **Pipe Balancer** (`o2-pipe-balancer.yaml`) - Балансировщик пайплайнов обработки данных
2. **Pipe Galileo** (`o2-pipe-galileo.yaml`) - Обработчик протокола Galileo
3. **Pipe ConfigMap** (`o2-pipe-configmap.yaml`) - Конфигурация пайплайнов

### Фоновые задачи (jobs/)

1. **Device Command Create** (`o2-job-mon-device-command-create.yaml`) - Создание команд для устройств
   - JOB_START: `mon_device`
   - JOB_KEY: `command.create`

2. **Device Packet Check Azimuth** (`o2-job-mon-device-packet-check-azimuth.yaml`) - Проверка азимута в пакетах
3. **Device Packet Receive** (`o2-job-mon-device-packet-receive.yaml`) - Получение пакетов от устройств
4. **Geofence Presence** (`o2-job-geofence-presence.yaml`) - Обработка присутствия в геозонах
   - Образ: `maprox/observer-job-geofence-presence`
   - AMQP_QUEUE_NAME: `prod.mon.device.geofence.update.onpacket`
   - Использует PostgreSQL и Redis

5. **Device Geofence Update OnPacket** (`o2-job-mon-device-geofence-update-onpacket.yaml`) - Обновление геозон устройств при получении пакета
   - Образ: `maprox/observer-job-geofence-presence`
   - AMQP_QUEUE_NAME: `prod.mon.device.geofence.update.onpacket`
   - Поддержка уведомлений через `x_notification`

6. **Device Packet Check Coord** (`o2-job-mon-device-packet-check-coord.yaml`) - Проверка координат в пакетах устройств
   - JOB_START: `mon_device`
   - JOB_KEY: `packet.check.coord`

7. **Fuel Check OnPacket** (`o2-job-mon-fuel-check-onpacket.yaml`) - Проверка топлива при получении пакета
   - JOB_START: `mon_fuel`
   - JOB_KEY: `check.onpacket`

8. **Ignition Check OnPacket** (`o2-job-mon-ignition-check-onpacket.yaml`) - Проверка зажигания при получении пакета
   - JOB_START: `mon_ignition`
   - JOB_KEY: `check.onpacket`

9. **Packet Update Address** (`o2-job-mon-packet-update-address.yaml`) - Обновление адресов в пакетах
   - JOB_START: `mon_packet`
   - JOB_KEY: `update.address`

10. **Track Check OnPacket** (`o2-job-mon-track-check-onpacket.yaml`) - Проверка трека при получении пакета
    - JOB_START: `mon_track`
    - JOB_KEY: `check.onpacket`

11. **Track Create OnPacket** (`o2-job-mon-track-create-onpacket.yaml`) - Создание трека при получении пакета
    - JOB_START: `mon_track`
    - JOB_KEY: `create.onpacket`

Большинство задач используют образ `maprox/o2-web` с переменной окружения `SRV_RUN=JOB`, за исключением геозон-сервиса, который использует специализированный образ `maprox/observer-job-geofence-presence`.

### Инфраструктура

1. **PostgreSQL** (`o2-postgresql.yaml`) - Основная база данных
   - Образ: `postgis/postgis:12-3.5` (PostGIS для геоданных)
   - Хранилище: 100Gi
   - Внешний доступ: NodePort 32432

2. **Redis** (`o2-redis.yaml`) - Кэш и сессии
   - Образ: `redis:latest`
   - Хранилище: 2Gi
   - Внешний доступ: NodePort 32379

3. **RabbitMQ** (`o2-rabbitmq.yaml`) - Очереди сообщений
   - Веб-интерфейс: `o2-rabbitmq.primne.com`
   - Внешний доступ: NodePort 32672

## Структура проекта

```
o2-k8s/
├── o2-web.yaml          # Веб-приложение
├── o2-node.yaml         # Node.js сервер
├── o2-static.yaml       # Статические файлы
├── o2-postgres.yaml     # PostgreSQL база данных
├── o2-redis.yaml        # Redis кэш
├── o2-rabbitmq.yaml     # RabbitMQ очереди
├── handlers/            # Обработчики данных
│   ├── o2-pipe-balancer.yaml    # Балансировщик пайплайнов
│   ├── o2-pipe-configmap.yaml   # Конфигурация пайплайнов
│   └── o2-pipe-galileo.yaml     # Обработчик протокола Galileo
├── jobs/                # Фоновые задачи
│   ├── o2-job-configmap.yaml                    # Конфигурация задач
│   ├── o2-job-geofence-presence.yaml            # Обработка присутствия в геозонах
│   ├── o2-job-mon-device-command-create.yaml    # Создание команд устройств
│   ├── o2-job-mon-device-geofence-update-onpacket.yaml  # Обновление геозон устройств при получении пакета
│   ├── o2-job-mon-device-packet-check-azimuth.yaml  # Проверка азимута пакетов
│   ├── o2-job-mon-device-packet-check-coord.yaml   # Проверка координат в пакетах устройств
│   ├── o2-job-mon-device-packet-receive.yaml    # Получение пакетов устройств
│   ├── o2-job-mon-fuel-check-onpacket.yaml      # Проверка топлива при получении пакета
│   ├── o2-job-mon-ignition-check-onpacket.yaml  # Проверка зажигания при получении пакета
│   ├── o2-job-mon-packet-update-address.yaml    # Обновление адресов в пакетах
│   ├── o2-job-mon-track-check-onpacket.yaml     # Проверка трека при получении пакета
│   └── o2-job-mon-track-create-onpacket.yaml    # Создание трека при получении пакета
├── secrets/             # Конфиденциальные данные
│   ├── postgres.yaml    # Пароли для БД
│   ├── rabbitmq.yaml    # Пароли для RabbitMQ
│   └── web.yaml         # Конфигурация веб-приложения
└── backup/              # Резервные копии конфигураций
    └── postgresql/      # Бэкап PostgreSQL
```

## Требования

- Kubernetes кластер
- NGINX Ingress Controller
- cert-manager для SSL сертификатов
- Local Path Provisioner для PV

## Установка

### 1. Создание namespace

```bash
kubectl create namespace o2
```

### 2. Применение секретов

```bash
kubectl apply -f secrets/
```

### 3. Развертывание инфраструктуры

```bash
# База данных
kubectl apply -f o2-postgresql.yaml

# Redis
kubectl apply -f o2-redis.yaml

# RabbitMQ (требует Helm chart)
kubectl apply -f o2-rabbitmq.yaml
```

### 4. Развертывание приложений

```bash
# Веб-приложение
kubectl apply -f o2-web.yaml

# Node.js сервер
kubectl apply -f o2-node.yaml

# Статические файлы
kubectl apply -f o2-static.yaml
```

### 5. Развертывание обработчиков и задач

```bash
# Обработчики данных
kubectl apply -f handlers/

# Фоновые задачи
kubectl apply -f jobs/
```

## Конфигурация

### SSL сертификаты

Используется Let's Encrypt с cert-manager:
- Email: `z@sunsay.ru`
- ClusterIssuer: `letsencrypt`

### Домены

- **observer.maprox.net** - основное приложение
- **static.maprox.net** - статические ресурсы
- **o2-rabbitmq.primne.com** - веб-интерфейс RabbitMQ

### Внешний доступ

Для разработки настроены NodePort сервисы:
- PostgreSQL: `:32432`
- Redis: `:32379`
- RabbitMQ: `:32672`

## Переменные окружения

### Node.js сервер
- `NODE_ENV=prod`
- `PORT=3000`
- `AMQP_HOST=rabbitmq`
- `REDIS_CONNECT_URL=redis://redis:6379`
- Интеграция с observer.maprox.net

### Web приложение
- Интеграция с внешними сервисами:
  - Google Analytics
  - SMS провайдеры (SMSPilot, SMSTeam)
  - Платежные системы (RBK Money)
  - Email (SMTP)
  - Yandex Metrika
  - Jasper Reports

## Мониторинг и логи

```bash
# Проверка статуса подов
kubectl get pods -n o2

# Логи сервисов
kubectl logs -n o2 deployment/web
kubectl logs -n o2 deployment/node
kubectl logs -n o2 deployment/postgres
kubectl logs -n o2 deployment/redis

# Проверка ingress
kubectl get ingress -n o2
```

## Безопасность

⚠️ **ВАЖНО**: Файлы в папке `secrets/` содержат конфиденциальную информацию:
- Пароли баз данных
- API ключи
- Сертификаты

Убедитесь, что они не попадают в публичный репозиторий.

## Обновление

```bash
# Обновление образов
kubectl set image deployment/web web=maprox/o2-web:latest -n o2
kubectl set image deployment/node node=maprox/node:latest -n o2
kubectl set image deployment/static static=maprox/static:latest -n o2
```

## Резервное копирование

В папке `backup/` находятся резервные копии конфигураций PostgreSQL.

## Автор

Copyright (c) 2025 Alexander Lyapko  
Лицензия: MIT

## Поддержка

Для вопросов и поддержки обращайтесь к команде разработки Maprox.
