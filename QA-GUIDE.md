# Руководство тестировщика: Создание тестового окружения

## Как это работает

Платформа IDP позволяет развернуть тестовое окружение **одним YAML-файлом**. Вы указываете, какие приложения нужны — платформа создаёт всё автоматически: namespace, deployments, services, базы данных.

**Принцип:** каждое приложение включается/выключается флагом `enabled: true/false`. Выключили — все ресурсы удаляются. Включили — создаются заново.

---

## Быстрый старт

### 1. Склонируйте репозиторий

```bash
git clone https://github.com/agud97/idp-app-v1.git
cd idp-app-v1
```

### 2. Создайте файл окружения

Создайте файл `developer-apps/my-env.yaml`:

```yaml
apiVersion: idp.example.com/v1alpha1
kind: TestEnvironment
metadata:
  name: my-env
  namespace: default
spec:
  name: test1                    # Префикс для namespace (test1-frontend, test1-backend, ...)

  frontend:
    enabled: true                # Включить frontend
  backend:
    enabled: true                # Включить backend с БД
  platform:
    enabled: false               # Не нужен — выключен
```

### 3. Запушьте в Git

```bash
git add developer-apps/my-env.yaml
git commit -m "Add test1 environment"
git push
```

### 4. Дождитесь деплоя (1-2 минуты)

ArgoCD автоматически увидит изменения и создаст окружение.

### 5. Проверьте

```bash
kubectl get pods -A | grep "^test1-"
```

---

## Доступные приложения

### Simple Apps (без БД)

| Поле в YAML | Kind | Описание | Порт | Реплики по умолчанию |
|-------------|------|----------|------|---------------------|
| `frontend` | FrontendApp | Фронтенд | 80 | 2 |
| `staticFiles` | StaticFilesApp | Статические файлы | 80 | 1 |
| `docs` | DocsApp | Документация | 80 | 1 |
| `adminDashboard` | AdminDashboardApp | Админ-панель | 80 | 1 |
| `landingPage` | LandingPageApp | Лендинг | 80 | 1 |

### Apps With DB (PostgreSQL)

| Поле в YAML | Kind | Описание | Порт | БД по умолчанию |
|-------------|------|----------|------|----------------|
| `backend` | BackendApp | Бэкенд API | 8080 | backend_db |
| `inventory` | InventoryApp | Склад | 8080 | inventory_db |
| `orders` | OrdersApp | Заказы | 8080 | orders_db |
| `billing` | BillingApp | Биллинг | 8080 | billing_db |
| `analytics` | AnalyticsApp | Аналитика | 8080 | analytics_db |

### Multi-Service Apps (микросервисы)

| Поле в YAML | Kind | Сервисы по умолчанию |
|-------------|------|---------------------|
| `platform` | PlatformApp | gateway, auth, users, notifications, worker, redis |
| `search` | SearchApp | indexer, query, crawler |
| `messaging` | MessagingApp | broker, producer, consumer |
| `monitoring` | MonitoringApp | collector, aggregator, dashboard |
| `cicd` | CicdApp | controller, runner, registry |

---

## Полный пример: QA окружение

```yaml
apiVersion: idp.example.com/v1alpha1
kind: TestEnvironment
metadata:
  name: qa-env
  namespace: default
spec:
  name: qa

  # ===== Simple Apps =====
  frontend:
    enabled: true
    replicas: 3                          # Переопределить количество реплик
    env:
      - name: API_URL
        value: "http://backend.qa-backend:80"
      - name: ENV
        value: qa
    ingress:
      enabled: true
      host: qa-frontend.example.com

  staticFiles:
    enabled: true

  docs:
    enabled: false

  adminDashboard:
    enabled: false

  landingPage:
    enabled: false

  # ===== Apps with DB =====
  backend:
    enabled: true
    replicas: 2
    port: 8080
    env:
      - name: ENV
        value: qa
    database:
      version: "15"
      storage: "1Gi"
      dbName: backend_qa
      username: backend
      password: qapass123
    migration:
      enabled: true
      image: postgres:15
      command:
        - /bin/sh
        - -c
        - "until pg_isready -h backend-db -U backend; do sleep 2; done && PGPASSWORD=$POSTGRES_PASSWORD psql -h backend-db -U backend -d backend_qa -c 'CREATE TABLE IF NOT EXISTS users (id SERIAL PRIMARY KEY, name VARCHAR(255), email VARCHAR(255));' && echo 'Migration done'"
    ingress:
      enabled: true
      host: qa-api.example.com

  inventory:
    enabled: false

  orders:
    enabled: false

  billing:
    enabled: false

  analytics:
    enabled: false

  # ===== Multi-service Apps =====
  platform:
    enabled: true
    services:
      - name: gateway
        image: nginx:alpine
        replicas: 2
        port: 80
        env:
          - name: AUTH_SERVICE
            value: "http://auth:80"
          - name: USER_SERVICE
            value: "http://users:80"
        ingress:
          enabled: true
          host: qa-gateway.example.com

      - name: auth
        image: nginx:alpine
        replicas: 2
        port: 8080
        env:
          - name: JWT_ISSUER
            value: qa-auth

      - name: users
        image: nginx:alpine
        replicas: 2
        port: 8080

      - name: notifications
        image: nginx:alpine
        replicas: 1
        port: 8080

      - name: worker
        image: nginx:alpine
        replicas: 2
        port: 8080
        env:
          - name: QUEUE_URL
            value: "redis://redis:6379"

      - name: redis
        image: redis:7-alpine
        replicas: 1
        port: 6379

  search:
    enabled: false

  messaging:
    enabled: false

  monitoring:
    enabled: false

  cicd:
    enabled: false
```

---

## Минимальный пример: только frontend + backend

```yaml
apiVersion: idp.example.com/v1alpha1
kind: TestEnvironment
metadata:
  name: smoke-test
  namespace: default
spec:
  name: smoke

  frontend:
    enabled: true

  backend:
    enabled: true
    database:
      password: testpass
```

Создаст:
- `smoke-frontend` — namespace с 2 подами nginx
- `smoke-backend` — namespace с 1 подом nginx + 1 PostgreSQL

---

## Включение/выключение приложений (toggle)

### Отключить приложение

Измените `enabled: true` на `enabled: false`:

```yaml
  frontend:
    enabled: false       # Было true
```

```bash
git add developer-apps/my-env.yaml
git commit -m "Disable frontend"
git push
```

Через 1-2 минуты:
- Namespace `{env}-frontend` будет удалён
- Все поды, сервисы, configmaps — удалены
- Остальные приложения **не затронуты**

### Включить приложение обратно

Измените `enabled: false` на `enabled: true`:

```yaml
  frontend:
    enabled: true        # Было false
```

```bash
git add developer-apps/my-env.yaml
git commit -m "Enable frontend"
git push
```

Через 1-2 минуты:
- Namespace `{env}-frontend` создан заново
- Все поды запущены

---

## Переопределение параметров

Каждое приложение имеет разумные значения по умолчанию. Вы можете переопределить любой параметр:

### Простое приложение
```yaml
frontend:
  enabled: true
  replicas: 5                    # По умолчанию 2
  image: my-frontend:v2.0        # По умолчанию nginx:alpine
  port: 3000                     # По умолчанию 80
  env:
    - name: MY_VAR
      value: my-value
  ingress:
    enabled: true
    host: my-frontend.example.com
```

### Приложение с БД
```yaml
backend:
  enabled: true
  replicas: 3
  port: 8080
  database:
    version: "15"               # Версия PostgreSQL
    storage: "5Gi"              # Размер диска
    dbName: my_database         # Имя БД
    username: myuser            # Пользователь
    password: mypassword        # Пароль
  migration:
    enabled: true
    image: postgres:15
    command:
      - /bin/sh
      - -c
      - "psql -h backend-db -U myuser -d my_database -c 'CREATE TABLE ...'"
```

### Мульти-сервисное приложение
```yaml
platform:
  enabled: true
  services:
    - name: gateway
      image: nginx:alpine
      replicas: 2
      port: 80
    - name: auth
      image: nginx:alpine
      replicas: 1
      port: 8080
    # Добавьте свои сервисы...
```

---

## Несколько окружений одновременно

Можно создать несколько TestEnvironment с разными именами. Каждое окружение создаёт namespace с уникальным префиксом:

**developer-apps/qa-env.yaml:**
```yaml
spec:
  name: qa           # → qa-frontend, qa-backend, ...
  frontend:
    enabled: true
  backend:
    enabled: true
```

**developer-apps/staging-env.yaml:**
```yaml
spec:
  name: staging      # → staging-frontend, staging-backend, ...
  frontend:
    enabled: true
  backend:
    enabled: true
```

Оба окружения будут работать параллельно и не мешать друг другу.

---

## Удаление окружения

Удалите YAML-файл и запушьте:

```bash
rm developer-apps/my-env.yaml
git add -A
git commit -m "Remove test environment"
git push
```

Все namespace и ресурсы будут удалены автоматически.

---

## Полезные команды

```bash
# Статус окружения
kubectl get testenvironment -A

# Все поды окружения (замените qa на ваш префикс)
kubectl get pods -A | grep "^qa-"

# Namespace окружения
kubectl get ns | grep "^qa-"

# Логи приложения
kubectl logs -n qa-frontend -l app=frontend

# Логи базы данных
kubectl logs -n qa-backend -l app=backend-db

# Подключиться к БД
kubectl exec -n qa-backend -it deploy/backend-db -- psql -U backend -d backend_qa

# Описание проблемы (если не запускается)
kubectl describe pods -n qa-frontend
kubectl get events -n qa-frontend

# Статусы всех claims
kubectl get frontendapp,backendapp,platformapp,staticfilesapp -A
```

---

## Обращение сервисов друг к другу

Внутри одного namespace сервисы обращаются по имени:
```
http://auth:80
redis://redis:6379
```

Между namespace — по полному DNS:
```
http://backend.qa-backend:80
http://gateway.qa-platform:80
```

Формат: `http://{service-name}.{namespace}:{port}`

---

## FAQ

**Q: Сколько времени занимает деплой?**
A: 1-2 минуты после `git push`.

**Q: Можно ли создать несколько окружений?**
A: Да, создайте несколько YAML-файлов с разными `spec.name` (qa, staging, dev, ...).

**Q: Что происходит при `enabled: false`?**
A: Все ресурсы приложения (namespace, поды, сервисы, БД, PVC) удаляются. При `enabled: true` — создаются заново с нуля.

**Q: Данные в БД сохраняются при toggle?**
A: Нет. При `enabled: false` PVC удаляется. Для сохранения данных не выключайте приложение с БД.

**Q: Как обновить параметры приложения?**
A: Измените YAML и запушьте. Изменения применятся автоматически (replicas, image, env, ...).

**Q: Standalone приложения и TestEnvironment мешают друг другу?**
A: Нет. Standalone FrontendApp создаёт namespace `frontend`, а TestEnvironment — namespace `{env}-frontend`. Это разные namespace.

**Q: Какие приложения обязательны?**
A: Никакие. Включайте только те, что нужны для теста.
