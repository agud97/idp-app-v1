# IDP - Internal Developer Platform

Платформа для быстрого деплоя приложений в Kubernetes. Разработчику достаточно создать простой YAML-файл — всё остальное создастся автоматически.

## Быстрый старт

1. Склонируйте репозиторий
2. Создайте YAML-файл в папке `developer-apps/`
3. Сделайте commit и push
4. Приложение автоматически задеплоится в кластер

## Типы приложений

### 1. Простое приложение (`Application`)

Для stateless приложений без базы данных.

**Что создаётся автоматически:**
- Namespace
- Deployment
- Service (ClusterIP)
- ConfigMap (переменные окружения)
- Secret (секретные переменные)
- Ingress (опционально)

**Пример:**

```yaml
apiVersion: idp.example.com/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: default
spec:
  # Обязательные поля
  name: my-app                    # Имя приложения
  image: nginx:alpine             # Docker образ

  # Опциональные поля
  replicas: 2                     # Количество реплик (по умолчанию: 1)
  port: 80                        # Порт контейнера (по умолчанию: 80)

  # Переменные окружения (ConfigMap)
  env:
    - name: APP_ENV
      value: production
    - name: LOG_LEVEL
      value: info

  # Секретные переменные (Secret)
  secrets:
    - name: API_KEY
      value: my-secret-key

  # Ingress (опционально)
  ingress:
    enabled: true
    host: my-app.example.com
```

---

### 2. Приложение с базой данных (`ApplicationWithDB`)

Для приложений, которым нужна PostgreSQL база данных и миграции.

**Что создаётся автоматически:**
- Namespace
- PostgreSQL Deployment + Service + PVC
- Secret с DATABASE_URL
- App Deployment с init-контейнером для миграций
- Service (ClusterIP)
- ConfigMap (переменные окружения)
- Secret (секретные переменные)
- Ingress (опционально)

**Пример:**

```yaml
apiVersion: idp.example.com/v1alpha1
kind: ApplicationWithDB
metadata:
  name: backend-api
  namespace: default
spec:
  # Обязательные поля
  name: backend-api               # Имя приложения
  image: myapp:latest             # Docker образ

  # Опциональные поля
  replicas: 2                     # Количество реплик (по умолчанию: 1)
  port: 8080                      # Порт контейнера (по умолчанию: 80)

  # Переменные окружения (ConfigMap)
  env:
    - name: APP_ENV
      value: production

  # База данных
  database:
    enabled: true
    engine: postgres              # Только postgres
    version: "15"                 # Версия PostgreSQL
    storage: "1Gi"                # Размер диска
    dbName: mydb                  # Имя базы данных
    username: myuser              # Пользователь
    password: mypassword          # Пароль

  # Миграции (init-контейнер)
  migration:
    enabled: true
    image: myapp:latest           # Образ для миграций
    command:
      - /bin/sh
      - -c
      - "npm run migrate"         # Команда миграции

  # Ingress (опционально)
  ingress:
    enabled: true
    host: backend-api.example.com
```

---

### 3. Мульти-сервисное приложение (`MultiServiceApp`)

Для приложений из нескольких микросервисов. Указываете список сервисов — для каждого создаётся отдельный Deployment.

**Что создаётся автоматически (для каждого сервиса):**
- Общий Namespace
- Deployment (для каждого сервиса)
- Service (для каждого сервиса)
- ConfigMap (если есть env)
- Secret (если есть secrets)
- Ingress (опционально, для каждого сервиса)

**Пример:**

```yaml
apiVersion: idp.example.com/v1alpha1
kind: MultiServiceApp
metadata:
  name: my-platform
  namespace: default
spec:
  name: my-platform               # Имя приложения (namespace)

  services:
    # Frontend
    - name: frontend
      image: nginx:alpine
      replicas: 2
      port: 80
      env:
        - name: API_URL
          value: "http://backend:80"
      ingress:
        enabled: true
        host: app.example.com

    # Backend API
    - name: backend
      image: myapi:latest
      replicas: 3
      port: 8080
      env:
        - name: DATABASE_HOST
          value: "postgres"
      secrets:
        - name: JWT_SECRET
          value: my-secret
      ingress:
        enabled: true
        host: api.example.com

    # Redis cache
    - name: redis
      image: redis:7-alpine
      replicas: 1
      port: 6379

    # Background worker
    - name: worker
      image: myworker:latest
      replicas: 2
      port: 8080
      env:
        - name: QUEUE_URL
          value: "redis://redis:6379"
```

**Сервисы могут обращаться друг к другу по имени:**
- `http://frontend:80`
- `http://backend:80`
- `redis:6379`

---

### 4. Тестовое окружение (`TestEnvironment`)

**Для тестировщиков.** Позволяет развернуть все типы приложений одним YAML-файлом.

**Что создаётся:**
- Все простые приложения (`simpleApps`)
- Все приложения с БД (`appsWithDB`)
- Все мульти-сервисные приложения (`multiServiceApps`)

Каждое приложение получает namespace с префиксом окружения: `{env-name}-{app-name}`

**Пример:**

```yaml
apiVersion: idp.example.com/v1alpha1
kind: TestEnvironment
metadata:
  name: qa-env
  namespace: default
spec:
  name: qa                          # Префикс для всех namespace

  # Простые приложения
  simpleApps:
    - name: frontend
      image: nginx:alpine
      replicas: 2
      port: 80
      env:
        - name: API_URL
          value: "http://backend.qa-backend:80"
      ingress:
        enabled: true
        host: qa-frontend.example.com

  # Приложения с PostgreSQL
  appsWithDB:
    - name: backend
      image: myapi:latest
      replicas: 2
      port: 8080
      database:
        version: "15"
        storage: "1Gi"
        dbName: backend_qa
        username: backend
        password: qapass123
      migration:
        enabled: true
        image: postgres:15
        command: ["/bin/sh", "-c", "psql ... -c 'CREATE TABLE ...'"]
      ingress:
        enabled: true
        host: qa-api.example.com

  # Мульти-сервисные приложения
  multiServiceApps:
    - name: platform
      services:
        - name: gateway
          image: nginx:alpine
          replicas: 2
          ingress:
            enabled: true
            host: qa-gateway.example.com
        - name: auth
          image: nginx:alpine
          replicas: 2
        - name: redis
          image: redis:7-alpine
          replicas: 1
```

**Проверить статус:**
```bash
kubectl get testenvironments.idp.example.com -A
```

**Созданные namespace:**
- `qa-frontend` — простое приложение
- `qa-backend` — приложение с БД
- `qa-platform` — мульти-сервисное приложение

---

## Переменные окружения для БД

При использовании `ApplicationWithDB` в контейнер автоматически передаются:

| Переменная | Описание |
|------------|----------|
| `DATABASE_URL` | Полный URL подключения |
| `POSTGRES_DB` | Имя базы данных |
| `POSTGRES_USER` | Пользователь |
| `POSTGRES_PASSWORD` | Пароль |

**Формат DATABASE_URL:**
```
postgres://username:password@app-name-db:5432/dbname?sslmode=disable
```

## Структура репозитория

```
idp-app-v1/
├── README.md                      # Этот файл
├── applications/                  # ArgoCD Applications (не трогать)
├── crossplane/                    # Crossplane конфигурация (не трогать)
│   ├── idp/                       # XRD и Compositions
│   └── providers/                 # Провайдеры Crossplane
└── developer-apps/                # СЮДА ДОБАВЛЯЙТЕ ВАШИ ПРИЛОЖЕНИЯ
    ├── test-simple-app.yaml       # Пример простого приложения
    ├── test-app-with-db.yaml      # Пример приложения с БД
    ├── test-multi-service.yaml    # Пример мульти-сервисного приложения
    └── qa-environment.yaml        # Пример тестового окружения (всё вместе)
```

## Проверка статуса

После push изменений, приложение появится в кластере через 1-2 минуты.

**Проверить статус:**
```bash
# Простое приложение
kubectl get applications.idp.example.com -A

# Приложение с БД
kubectl get applicationwithdbs.idp.example.com -A

# Мульти-сервисное приложение
kubectl get multiserviceapps.idp.example.com -A

# Тестовое окружение
kubectl get testenvironments.idp.example.com -A

# Поды приложения
kubectl get pods -n <имя-приложения>
```

## Примеры команд миграций

### Node.js (Prisma)
```yaml
migration:
  enabled: true
  image: myapp:latest
  command:
    - /bin/sh
    - -c
    - "npx prisma migrate deploy"
```

### Node.js (TypeORM)
```yaml
migration:
  enabled: true
  image: myapp:latest
  command:
    - /bin/sh
    - -c
    - "npm run typeorm migration:run"
```

### Python (Alembic)
```yaml
migration:
  enabled: true
  image: myapp:latest
  command:
    - /bin/sh
    - -c
    - "alembic upgrade head"
```

### Go (golang-migrate)
```yaml
migration:
  enabled: true
  image: myapp:latest
  command:
    - /bin/sh
    - -c
    - "/app/migrate -path /app/migrations -database $DATABASE_URL up"
```

### Raw SQL
```yaml
migration:
  enabled: true
  image: postgres:15
  command:
    - /bin/sh
    - -c
    - "until pg_isready -h myapp-db -U myuser; do sleep 2; done && PGPASSWORD=$POSTGRES_PASSWORD psql -h myapp-db -U myuser -d mydb -f /migrations/init.sql"
```

## FAQ

**Q: Как обновить приложение?**
A: Измените YAML-файл и сделайте push. Изменения применятся автоматически.

**Q: Как удалить приложение?**
A: Удалите YAML-файл из `developer-apps/` и сделайте push.

**Q: Как посмотреть логи?**
A: `kubectl logs -n <имя-приложения> -l app=<имя-приложения>`

**Q: Как подключиться к базе данных?**
A: `kubectl exec -n <имя-приложения> -it deploy/<имя-приложения>-db -- psql -U <username> -d <dbname>`

**Q: Почему приложение не запускается?**
A: Проверьте статус: `kubectl describe application.idp.example.com <имя>` или `kubectl get events -n <имя-приложения>`
