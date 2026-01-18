# ДЗ: бизнес‑система + HLD (C4 container) + Kubernetes YAML (+ NetworkPolicies)

## 1) Выбранная бизнес‑система
**VFX Asset Store** — онлайн‑магазин/маркетплейс для покупки и скачивания VFX/3D‑ассетов (модели, текстуры, FX‑пресеты).
Пользователь: смотрит каталог → кладёт в корзину → оплачивает → получает ссылку на скачивание.

---

## 2) Основные компоненты (микросервисы) и что они делают (с образами)
Ниже перечислены сервисы, которые мы деплоим в Kubernetes, и внешние зависимости.

### Kubernetes‑сервисы (контейнеры)
1) **web-frontend** (образ: `ghcr.io/example/vfx-frontend:1.0.0`)
- Статический фронтенд (SPA) + отдача ассетов через Nginx.
- Ходит в `api-gateway` по `/api`.

2) **api-gateway** (образ: `ghcr.io/example/vfx-gateway:1.0.0`)
- BFF / API gateway: единая точка входа для фронта.
- Маршрутизация в `catalog-service` и `order-service`.
- Простая валидация JWT (ключ в Secret), rate limiting (можно через Nginx/Ingress или внутри).
- Логи/метрики.

3) **catalog-service** (образ: `ghcr.io/example/vfx-catalog:1.0.0`)
- Каталог ассетов, карточки товара, цены, теги.
- CRUD для контента (в реальном мире — админка).
- Читает/пишет в PostgreSQL.
- Доступ к объектному хранилищу (S3) для превью/файлов (только ссылки и метаданные, сами файлы — в S3).

4) **order-service** (образ: `ghcr.io/example/vfx-order:1.0.0`)
- Корзина, создание заказа, статусы оплаты.
- Вызывает внешний **Payment Provider** по REST (например, Stripe/CloudPayments).
- После успешной оплаты пишет заказ в PostgreSQL и публикует событие `OrderPaid` в RabbitMQ.

5) **notification-worker** (образ: `ghcr.io/example/vfx-notify:1.0.0`)
- Consumer очереди RabbitMQ.
- Получает `OrderPaid` → вызывает внешний Email/SMS провайдер (SendGrid/Twilio) → отправляет письмо со ссылкой на скачивание.

### Хранилища и инфраструктура
- **PostgreSQL** (вне k8s) — транзакционная БД (каталог, заказы).
- **RabbitMQ** (вне k8s) — очередь сообщений (события по заказам).
- **S3/MinIO** (вне k8s) — объектное хранилище файлов ассетов и превью.

### Внешние зависимости (REST/источники данных)
- **Payment API** (REST): создание платежа/проверка статуса.
- **Email/SMS API** (REST): отправка уведомлений.
- **Object Storage API** (S3): загрузка/выдача файлов (может быть через presigned URL).
- (Опционально) **Fraud scoring API** (REST) для проверки риск‑профиля заказа.

---

## 3) Основные сценарии (потоки данных)
1) **Просмотр каталога**
- Frontend → API Gateway → Catalog Service → PostgreSQL → обратно.

2) **Покупка**
- Frontend → API Gateway → Order Service.
- Order Service → Payment Provider (REST) → получает `payment_intent_id`/redirect URL.
- После webhook/подтверждения оплаты: Order Service пишет статус в PostgreSQL и публикует `OrderPaid` в RabbitMQ.
- Notification Worker читает `OrderPaid` → Email Provider (REST) → письмо пользователю.

3) **Скачивание**
- Ссылка в письме ведёт на API Gateway → Order Service проверяет права → выдаёт presigned URL на объект в S3.

---

## 4) HLD: C4 container диаграмма (Mermaid)

```mermaid
flowchart LR
  user([Пользователь / Browser])
  admin([Администратор контента])

  subgraph internet[Интернет]
    pay[Payment Provider\n(REST)]
    mail[Email/SMS Provider\n(REST)]
    s3[Object Storage S3/MinIO\n(S3 API)]
  end

  subgraph k8s[Kubernetes Cluster: vfx-store]
    ing[Ingress Controller]
    fe[web-frontend\nNginx + SPA]
    gw[api-gateway\nBFF]
    cat[catalog-service]
    ord[order-service]
    wkr[notification-worker]
  end

  subgraph ext[Сервисы вне Kubernetes]
    pg[(PostgreSQL)]
    mq[(RabbitMQ)]
  end

  user -->|HTTPS| ing
  ing -->|/| fe
  ing -->|/api| gw

  admin -->|HTTPS /api/admin| gw

  gw --> cat
  gw --> ord

  cat --> pg
  cat --> s3

  ord --> pg
  ord --> pay
  ord --> mq

  wkr --> mq
  wkr --> mail
  wkr --> s3
```

---

## 5) Что будет вне Kubernetes (по заданию): порты, DNS/IP, сервер, системные требования
выносим stateful‑компоненты в managed‑сервисы

### 5.1 PostgreSQL (вне k8s)
- **DNS**: `postgres.vfx.internal`
- **IP (пример)**: `10.10.0.10`
- **Порт**: `5432/tcp`
- **Сервер** (пример): 4 vCPU, 16 GB RAM, SSD 200 GB, Linux.
- **Системные требования**:
  - SSD (важно для latency),
  - ежедневные бэкапы + PITR,
  - включён TLS,
  - ограничение доступа по сети (только из k8s‑нод / подсети).

### 5.2 RabbitMQ (вне k8s)
- **DNS**: `rabbitmq.vfx.internal`
- **IP (пример)**: `10.10.0.11`
- **Порт**: `5672/tcp` (AMQP), `15672/tcp` (management, только админам)
- **Сервер** (пример): 2 vCPU, 8 GB RAM, SSD 50–100 GB.
- **Требования**:
  - TLS,
  - отдельные пользователи/виртуальные хосты,
  - retention/TTL для очередей, DLQ.

### 5.3 Object Storage (вне k8s)
- **DNS**: `s3.vfx.internal` (или публичный S3).
- **Порт**: `443/tcp`
- **Требования**:
  - bucket policy,
  - presigned URL,
  - версионирование объектов (опционально).

---

## 6) Kubernetes‑архитектура (какие ресурсы используем)
Минимально по условию используем: **Deployment, Service, Ingress, ConfigMap, Secret, CronJob** и прописываем **requests/limits**.

- `Namespace`: изоляция проекта.
- `ConfigMap`: не‑секретные конфиги (URL сервисов, флаги).
- `Secret`: пароли/ключи (DB, RabbitMQ, JWT).
- `Deployment`: stateless сервисы.
- `Service`: стабильные DNS‑имена внутри кластера.
- `Ingress`: вход по домену и маршрутизация `/` и `/api`.
- `CronJob`: ночная чистка временных файлов/истёкших ссылок (пример).

---

## 7) NetworkPolicies (разграничение доступа)
Идея:
1) Включаем **default deny** по ingress/egress.
2) Разрешаем **DNS** (kube-dns).
3) Разрешаем вход только через Ingress Controller к frontend и gateway.
4) Разрешаем gateway → catalog/order.
5) Разрешаем catalog/order/worker выход к внешним сервисам (DB/RabbitMQ/Email/Payment/S3) по конкретным портам и (по возможности) по IP‑диапазонам.