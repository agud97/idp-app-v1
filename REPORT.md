# Отчёт о создании IDP (Internal Developer Platform)

## Обзор

Создана платформа IDP на базе Kubernetes с использованием ArgoCD и Crossplane, позволяющая разработчикам и тестировщикам деплоить приложения через простые YAML-файлы.

**Архитектура:**
- **ArgoCD** — GitOps-контроллер, следит за репозиторием и автоматически применяет изменения
- **Crossplane** — оператор, который по YAML-описанию создаёт реальные Kubernetes-ресурсы (Deployments, Services, и т.д.)
- **Go-Templating** — функция Crossplane для рендеринга шаблонов

**Поддерживаемые типы:**

### Базовые типы (generic, для разработчиков)
| Тип | Kind | Описание |
|-----|------|----------|
| Простое | `Application` | Один контейнер, без БД |
| С базой данных | `ApplicationWithDB` | Один контейнер + PostgreSQL + миграции |
| Мульти-сервис | `MultiServiceApp` | Несколько контейнеров из одного YAML |

### App-specific Kinds (15 штук)

**Simple Apps (5):**
| Kind | App Name | Image | Port | Replicas |
|------|----------|-------|------|----------|
| `FrontendApp` | frontend | nginx:alpine | 80 | 2 |
| `StaticFilesApp` | static-files | nginx:alpine | 80 | 1 |
| `DocsApp` | docs | nginx:alpine | 80 | 1 |
| `AdminDashboardApp` | admin-dashboard | nginx:alpine | 80 | 1 |
| `LandingPageApp` | landing-page | nginx:alpine | 80 | 1 |

**Apps With DB (5):**
| Kind | App Name | Image | Port | DB Name |
|------|----------|-------|------|---------|
| `BackendApp` | backend | nginx:alpine | 8080 | backend_db |
| `InventoryApp` | inventory | nginx:alpine | 8080 | inventory_db |
| `OrdersApp` | orders | nginx:alpine | 8080 | orders_db |
| `BillingApp` | billing | nginx:alpine | 8080 | billing_db |
| `AnalyticsApp` | analytics | nginx:alpine | 8080 | analytics_db |

**Multi-Service Apps (5):**
| Kind | App Name | Сервисы |
|------|----------|---------|
| `PlatformApp` | platform | gateway, auth, users, notifications, worker, redis |
| `SearchApp` | search | indexer, query, crawler |
| `MessagingApp` | messaging | broker, producer, consumer |
| `MonitoringApp` | monitoring | collector, aggregator, dashboard |
| `CicdApp` | cicd | controller, runner, registry |

### Тестовое окружение
| Тип | Kind | Описание |
|-----|------|----------|
| Тестовое окружение | `TestEnvironment` | Включение/выключение любых приложений через `enabled: true/false` |

---

## 1. Подключение к репозиторию и кластеру

### Клонирование репозитория с GitHub токеном
```bash
git clone https://agud97:<TOKEN>@github.com/agud97/idp-app-v1.git /root/proj/cross/idp-app-v1
```

### Проверка подключения к Kubernetes кластеру
```bash
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl cluster-info
```

**Результат:**
```
Kubernetes control plane is running at https://194.58.110.52:6443
CoreDNS is running at https://194.58.110.52:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

---

## 2. Установка ArgoCD

```bash
# Создание namespace
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl create namespace argocd

# Установка ArgoCD
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side=true --force-conflicts

# Настройка доступа к репозиторию
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl create secret generic repo-idp-app-v1 \
  --namespace=argocd \
  --from-literal=type=git \
  --from-literal=url=https://github.com/agud97/idp-app-v1.git \
  --from-literal=username=agud97 \
  --from-literal=password=<TOKEN>

KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl label secret repo-idp-app-v1 \
  -n argocd argocd.argoproj.io/secret-type=repository
```

---

## 3. Установка Crossplane

### Создан файл `applications/crossplane.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.crossplane.io/stable
    chart: crossplane
    targetRevision: "1.15.0"
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 4. Установка Crossplane Providers

### Kubernetes Provider (`crossplane/providers/provider-kubernetes.yaml`)
```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-kubernetes
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.13.0
```

### Go-Templating Function (`crossplane/providers/function-go-templating.yaml`)
```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-go-templating
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-go-templating:v0.5.0
```

### Проверка
```bash
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl get providers,functions
```

**Результат:**
```
NAME                  INSTALLED   HEALTHY   PACKAGE
provider-kubernetes   True        True      xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.13.0

NAME                       INSTALLED   HEALTHY   PACKAGE
function-go-templating     True        True      xpkg.upbound.io/crossplane-contrib/function-go-templating:v0.5.0
```

---

## 5. Установка OpenEBS для локального хранилища

### Создан файл `applications/openebs.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: openebs
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://openebs.github.io/dynamic-localpv-provisioner
    chart: localpv-provisioner
    targetRevision: "4.1.0"
    helm:
      values: |
        hostpathClass:
          enabled: true
          isDefaultClass: true
          name: openebs-hostpath
  destination:
    server: https://kubernetes.default.svc
    namespace: openebs
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Результат:**
```
NAME                         PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE
openebs-hostpath (default)   openebs.io/local   Delete          WaitForFirstConsumer
```

---

## 6. Создание IDP типов приложений

### 6.1. Базовые generic типы (в `crossplane/idp/base/`)

| Тип | XRD | Composition | Что создаёт |
|-----|-----|-------------|-------------|
| Application | `definition.yaml` | `composition.yaml` | Namespace, Deployment, Service, ConfigMap, Secret, Ingress |
| ApplicationWithDB | `definition-with-db.yaml` | `composition-with-db.yaml` | + PostgreSQL Deployment, PVC, DB Secret, миграции |
| MultiServiceApp | `definition-multi.yaml` | `composition-multi.yaml` | N × (Deployment, Service, ConfigMap, Ingress) в одном namespace |

### 6.2. App-specific Kinds (в `crossplane/idp/apps/`)

15 приложений, каждое со своим Kind, дефолтными значениями и `environmentName` полем для namespace-префикса.

Каждое приложение деплоится самостоятельно (standalone) или через TestEnvironment.

**Дизайн:**
- Если `environmentName` задан: namespace = `{envName}-{appName}`
- Если не задан: namespace = `{appName}`
- Все параметры (image, replicas, port) имеют дефолты, можно переопределить

**Минимизация дублирования шаблонов:**
Все apps одного базового типа используют идентичный go-template, только переменные в начале отличаются:
```
{{- $appName := "frontend" }}
{{- $defaultImage := "nginx:alpine" }}
{{- $defaultPort := 80 }}
{{- $defaultReplicas := 2 }}
... (тело шаблона одинаковое для всех simple apps)
```

### 6.3. TestEnvironment (в `crossplane/idp/environment/`)

**Новый дизайн** — вместо массивов `simpleApps[]`, `appsWithDB[]`, `multiServiceApps[]` используются именованные поля с `enabled: true/false`:

```yaml
spec:
  name: qa
  frontend:
    enabled: true
    replicas: 3
  backend:
    enabled: true
    database:
      password: my-pass
  platform:
    enabled: true
  search:
    enabled: false
```

**Цепочка создания:**
```
TestEnvironment → Composition рендерит Object → Object создаёт Claim (FrontendApp) → Composition FrontendApp → K8s ресурсы
```

**Цепочка удаления (enabled: false):**
```
Object не рендерится → Claim удаляется → все ресурсы удаляются → чистый teardown
```

---

## 7. Тестирование

### Тест 1: Простое приложение (Application)

```yaml
# developer-apps/test-simple-app.yaml
apiVersion: idp.example.com/v1alpha1
kind: Application
metadata:
  name: test-simple-app
spec:
  name: test-simple-app
  image: nginx:alpine
  replicas: 2
```

**Результат:** 2 пода Running, Service, ConfigMap, Ingress

---

### Тест 2: Приложение с БД (ApplicationWithDB)

```yaml
# developer-apps/test-app-with-db.yaml
apiVersion: idp.example.com/v1alpha1
kind: ApplicationWithDB
metadata:
  name: test-app-with-db
spec:
  name: test-app-with-db
  image: nginx:alpine
  replicas: 2
  database:
    version: "15"
    dbName: testdb
    username: testuser
    password: testpass123
  migration:
    enabled: true
    image: postgres:15
    command: ["/bin/sh", "-c", "psql ... CREATE TABLE users ..."]
```

**Результат:** 2 app пода + 1 PostgreSQL, миграция выполнена, таблица создана

---

### Тест 3: Мульти-сервисное приложение (MultiServiceApp)

```yaml
# developer-apps/test-multi-service.yaml
apiVersion: idp.example.com/v1alpha1
kind: MultiServiceApp
metadata:
  name: test-multi-service
spec:
  name: test-multi-service
  services:
    - name: frontend
      replicas: 2
    - name: backend
      replicas: 3
    - name: redis
      replicas: 1
    - name: worker
      replicas: 2
```

**Результат:** 8 подов (4 сервиса), все Running

---

### Тест 4: Standalone FrontendApp

```yaml
# developer-apps/test-frontend-standalone.yaml
apiVersion: idp.example.com/v1alpha1
kind: FrontendApp
metadata:
  name: test-frontend-standalone
spec:
  replicas: 2
  ingress:
    enabled: true
    host: frontend.example.com
```

**Результат:** namespace `frontend`, 2 пода Running (без environmentName → namespace = appName)

---

### Тест 5: TestEnvironment с enable/disable

```yaml
# developer-apps/qa-environment.yaml
apiVersion: idp.example.com/v1alpha1
kind: TestEnvironment
metadata:
  name: qa-env
spec:
  name: qa
  frontend:
    enabled: true
    replicas: 3
  staticFiles:
    enabled: true
  backend:
    enabled: true
    database:
      dbName: backend_qa
      password: qapass123
  platform:
    enabled: true
    services: [gateway, auth, users, notifications, worker, redis]
  # Остальные 11 приложений: enabled: false
```

**Результат:**
```
NAMESPACE         PODS   ОПИСАНИЕ
qa-frontend       3      FrontendApp (replicas: 3 override)
qa-static-files   1      StaticFilesApp
qa-backend        3      BackendApp + PostgreSQL (миграция + PVC)
qa-platform       9      PlatformApp (6 микросервисов)
────────────────────────────────────────
ИТОГО             16     4 namespace, 11 приложений отключены
```

**Disabled apps проверка:** Namespace для docs, admin-dashboard, landing-page, inventory, orders, billing, analytics, search, messaging, monitoring, cicd — не существуют (корректно).

---

### Тест 6: Toggle frontend off/on

| Шаг | qa-frontend namespace | Поды | Standalone frontend |
|-----|----------------------|------|---------------------|
| До | Active, 3 пода | Running | Не затронут |
| `enabled: false` (push) | **Удалён** | **Удалены** | Не затронут |
| `enabled: true` (push) | **Создан заново** | **3 Running** | Не затронут |

Чистый teardown и пересоздание работают. Standalone `FrontendApp` в namespace `frontend` не затронут toggle-ом.

---

## 8. Финальная структура репозитория

```
idp-app-v1/
├── README.md                               # Для разработчиков
├── REPORT.md                               # Этот отчёт
├── QA-GUIDE.md                             # Для тестировщиков
├── applications/
│   ├── crossplane.yaml
│   ├── crossplane-provider-kubernetes.yaml
│   ├── crossplane-idp.yaml
│   ├── developer-apps.yaml
│   └── openebs.yaml
├── crossplane/
│   ├── idp/
│   │   ├── base/                           # Generic типы
│   │   │   ├── definition.yaml             # XApplication
│   │   │   ├── composition.yaml
│   │   │   ├── definition-with-db.yaml     # XApplicationWithDB
│   │   │   ├── composition-with-db.yaml
│   │   │   ├── definition-multi.yaml       # XMultiServiceApp
│   │   │   └── composition-multi.yaml
│   │   ├── apps/                           # 15 app-specific Kinds
│   │   │   ├── frontend/                   # FrontendApp
│   │   │   │   ├── definition.yaml
│   │   │   │   └── composition.yaml
│   │   │   ├── static-files/               # StaticFilesApp
│   │   │   ├── docs/                       # DocsApp
│   │   │   ├── admin-dashboard/            # AdminDashboardApp
│   │   │   ├── landing-page/               # LandingPageApp
│   │   │   ├── backend/                    # BackendApp (с БД)
│   │   │   ├── inventory/                  # InventoryApp (с БД)
│   │   │   ├── orders/                     # OrdersApp (с БД)
│   │   │   ├── billing/                    # BillingApp (с БД)
│   │   │   ├── analytics/                  # AnalyticsApp (с БД)
│   │   │   ├── platform/                   # PlatformApp (мульти-сервис)
│   │   │   ├── search/                     # SearchApp (мульти-сервис)
│   │   │   ├── messaging/                  # MessagingApp (мульти-сервис)
│   │   │   ├── monitoring/                 # MonitoringApp (мульти-сервис)
│   │   │   └── cicd/                       # CicdApp (мульти-сервис)
│   │   └── environment/                    # TestEnvironment
│   │       ├── definition.yaml
│   │       └── composition.yaml
│   └── providers/
│       ├── provider-kubernetes.yaml
│       ├── provider-kubernetes-config.yaml
│       └── function-go-templating.yaml
└── developer-apps/
    ├── test-simple-app.yaml
    ├── test-app-with-db.yaml
    ├── test-multi-service.yaml
    ├── test-frontend-standalone.yaml       # Standalone FrontendApp
    └── qa-environment.yaml                 # TestEnvironment с enable/disable
```

---

## 9. Все Crossplane XRDs (19 штук)

```bash
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl get xrd
```

```
NAME                                  ESTABLISHED   OFFERED
xadmindashboardapps.idp.example.com   True          True
xanalyticsapps.idp.example.com        True          True
xapplications.idp.example.com         True          True
xapplicationwithdbs.idp.example.com   True          True
xbackendapps.idp.example.com          True          True
xbillingapps.idp.example.com          True          True
xcicdapps.idp.example.com             True          True
xdocsapps.idp.example.com             True          True
xfrontendapps.idp.example.com         True          True
xinventoryapps.idp.example.com        True          True
xlandingpageapps.idp.example.com      True          True
xmessagingapps.idp.example.com        True          True
xmonitoringapps.idp.example.com       True          True
xmultiserviceapps.idp.example.com     True          True
xordersapps.idp.example.com           True          True
xplatformapps.idp.example.com         True          True
xsearchapps.idp.example.com           True          True
xstaticfilesapps.idp.example.com      True          True
xtestenvironments.idp.example.com     True          True
```

---

## 10. Все ArgoCD Applications

```bash
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl get applications -n argocd
```

```
NAME                             SYNC STATUS   HEALTH STATUS
crossplane                       Synced        Healthy
crossplane-idp                   Synced        Healthy
crossplane-provider-kubernetes   Synced        Healthy
developer-apps                   Synced        Healthy
openebs                          Synced        Healthy
```

---

## 11. Список коммитов

```
4cd52cf Test: re-enable frontend in qa-environment
1595070 Test: disable frontend in qa-environment
87572e0 Refactor: 15 app-specific Crossplane Kinds with enable/disable TestEnvironment
58257e2 Add TestEnvironment for deploying all app types together
06f6915 Update report with MultiServiceApp documentation
3945f05 Add MultiServiceApp for multi-container deployments
6db8e6b Add detailed implementation report
d452012 Add README with developer instructions
8f3dceb Fix composition template for multiline commands
c94d051 Add test-app-with-db to verify IDP with database
66478e9 Add test-simple-app to verify IDP
115604a Fix OpenEBS image name to provisioner-localpv
14d4b9f Fix OpenEBS to use localpv-provisioner chart
e956e8f Add OpenEBS for local storage
5501409 Add ApplicationWithDB for apps with database and migrations
e6d36b1 Add Kubernetes ProviderConfig
f32f94d Add IDP platform with Crossplane XRD and Composition
3c100e0 Add Crossplane Kubernetes provider
1e9fa6f Add ArgoCD Application for Crossplane
ab7c5f5 Initial commit
```

---

## Итог

| Компонент | Статус |
|-----------|--------|
| ArgoCD | Установлен и настроен |
| Crossplane | Установлен (v1.15.0) |
| Kubernetes Provider | Установлен (v0.13.0) |
| Function Go-Templating | Установлен (v0.5.0) |
| OpenEBS | Установлен (StorageClass default) |
| 3 базовых типа (Application, ApplicationWithDB, MultiServiceApp) | Работают |
| 15 app-specific Kinds | Работают |
| TestEnvironment с enable/disable | Работает |
| Toggle on/off (teardown/recreate) | Протестирован |

### Типы приложений

| Тип | Kind | Для кого | Что создаётся |
|-----|------|----------|---------------|
| Простое | `Application` | Разработчик | Namespace, Deployment, Service, ConfigMap, Ingress |
| С БД | `ApplicationWithDB` | Разработчик | + PostgreSQL, PVC, миграции |
| Мульти-сервис | `MultiServiceApp` | Разработчик | N × (Deployment, Service, ConfigMap, Ingress) |
| App-specific (15 штук) | `FrontendApp`, `BackendApp`, ... | Разработчик/Тестировщик | Специализированные приложения с дефолтами |
| Тестовое окружение | `TestEnvironment` | **Тестировщик** | **enable/disable любых приложений** |

### Команды для проверки

```bash
# Все XRDs
kubectl get xrd

# Все app-specific claims
kubectl get frontendapp,backendapp,platformapp,staticfilesapp -A

# TestEnvironment
kubectl get testenvironments.idp.example.com -A

# Все поды тестового окружения
kubectl get pods -A | grep "^qa-"
```

**Репозиторий:** https://github.com/agud97/idp-app-v1
