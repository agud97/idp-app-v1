# Отчёт о создании IDP (Internal Developer Platform)

## Обзор

Создана платформа IDP на базе Kubernetes с использованием ArgoCD и Crossplane, позволяющая разработчикам и тестировщикам деплоить приложения через простые YAML-файлы.

**Поддерживаемые типы приложений:**
| Тип | Kind | Описание |
|-----|------|----------|
| Простое | `Application` | Один контейнер, без БД |
| С базой данных | `ApplicationWithDB` | Один контейнер + PostgreSQL + миграции |
| Мульти-сервис | `MultiServiceApp` | Несколько контейнеров из одного YAML |
| Тестовое окружение | `TestEnvironment` | **Все типы вместе** — для тестировщиков |

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

### 6.1. Простое приложение (`Application`)

**XRD:** `crossplane/idp/definition.yaml`
**Composition:** `crossplane/idp/composition.yaml`

Создаёт: Namespace, Deployment, Service, ConfigMap, Secret, Ingress

### 6.2. Приложение с БД (`ApplicationWithDB`)

**XRD:** `crossplane/idp/definition-with-db.yaml`
**Composition:** `crossplane/idp/composition-with-db.yaml`

Создаёт: + PostgreSQL Deployment, PVC, DB Secret, Init-контейнер для миграций

### 6.3. Мульти-сервисное приложение (`MultiServiceApp`)

**XRD:** `crossplane/idp/definition-multi.yaml`
**Composition:** `crossplane/idp/composition-multi.yaml`

Создаёт: N × (Deployment, Service, ConfigMap, Ingress) в одном namespace

### 6.4. Тестовое окружение (`TestEnvironment`)

**XRD:** `crossplane/idp/definition-environment.yaml`
**Composition:** `crossplane/idp/composition-environment.yaml`

Объединяет все три типа в одном YAML для тестировщиков.

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

**Результат:** ✅ 2 пода Running, Service, ConfigMap, Ingress

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

**Результат:** ✅ 2 app пода + 1 PostgreSQL, миграция выполнена, таблица создана

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

**Результат:** ✅ 8 подов (4 сервиса), все Running

---

### Тест 4: Тестовое окружение (TestEnvironment)

```yaml
# developer-apps/qa-environment.yaml
apiVersion: idp.example.com/v1alpha1
kind: TestEnvironment
metadata:
  name: qa-env
spec:
  name: qa

  simpleApps:
    - name: frontend
      replicas: 2
    - name: static-files
      replicas: 1

  appsWithDB:
    - name: backend
      replicas: 2
      database:
        dbName: backend_qa
      migration:
        enabled: true

  multiServiceApps:
    - name: platform
      services:
        - name: gateway
          replicas: 2
        - name: auth
          replicas: 2
        - name: users
          replicas: 2
        - name: notifications
          replicas: 1
        - name: worker
          replicas: 2
        - name: redis
          replicas: 1
```

**Результат:**
```
NAMESPACE         PODS   ОПИСАНИЕ
qa-frontend       2      Simple app
qa-static-files   1      Simple app
qa-backend        3      App + PostgreSQL
qa-platform       10     6 микросервисов
─────────────────────────────────────
ИТОГО             16     4 namespace
```

**Проверка:**
```bash
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl get pods -A | grep "^qa-"
```

```
qa-backend       backend-xxx        1/1     Running
qa-backend       backend-xxx        1/1     Running
qa-backend       backend-db-xxx     1/1     Running
qa-frontend      frontend-xxx       1/1     Running
qa-frontend      frontend-xxx       1/1     Running
qa-platform      auth-xxx           1/1     Running
qa-platform      auth-xxx           1/1     Running
qa-platform      gateway-xxx        1/1     Running
qa-platform      gateway-xxx        1/1     Running
qa-platform      notifications-xxx  1/1     Running
qa-platform      redis-xxx          1/1     Running
qa-platform      users-xxx          1/1     Running
qa-platform      users-xxx          1/1     Running
qa-platform      worker-xxx         1/1     Running
qa-platform      worker-xxx         1/1     Running
qa-static-files  static-files-xxx   1/1     Running
```

**Все 16 подов Running!**

---

## 8. Финальная структура репозитория

```
idp-app-v1/
├── README.md
├── REPORT.md
├── applications/
│   ├── crossplane.yaml
│   ├── crossplane-provider-kubernetes.yaml
│   ├── crossplane-idp.yaml
│   ├── developer-apps.yaml
│   └── openebs.yaml
├── crossplane/
│   ├── idp/
│   │   ├── definition.yaml              # XRD Application
│   │   ├── definition-with-db.yaml      # XRD ApplicationWithDB
│   │   ├── definition-multi.yaml        # XRD MultiServiceApp
│   │   ├── definition-environment.yaml  # XRD TestEnvironment
│   │   ├── composition.yaml             # Composition Application
│   │   ├── composition-with-db.yaml     # Composition ApplicationWithDB
│   │   ├── composition-multi.yaml       # Composition MultiServiceApp
│   │   └── composition-environment.yaml # Composition TestEnvironment
│   └── providers/
│       ├── provider-kubernetes.yaml
│       ├── provider-kubernetes-config.yaml
│       └── function-go-templating.yaml
└── developer-apps/
    ├── test-simple-app.yaml
    ├── test-app-with-db.yaml
    ├── test-multi-service.yaml
    └── qa-environment.yaml              # Полное тестовое окружение
```

---

## 9. Все Crossplane XRDs

```bash
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl get xrd
```

```
NAME                                  ESTABLISHED   OFFERED
xapplications.idp.example.com         True          True
xapplicationwithdbs.idp.example.com   True          True
xmultiserviceapps.idp.example.com     True          True
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
```

---

## Итог

| Компонент | Статус |
|-----------|--------|
| ArgoCD | Установлен и настроен ✅ |
| Crossplane | Установлен (v1.15.0) ✅ |
| Kubernetes Provider | Установлен (v0.13.0) ✅ |
| Function Go-Templating | Установлен (v0.5.0) ✅ |
| OpenEBS | Установлен (StorageClass default) ✅ |
| IDP Application | Работает ✅ |
| IDP ApplicationWithDB | Работает ✅ |
| IDP MultiServiceApp | Работает ✅ |
| IDP TestEnvironment | Работает ✅ |

### Типы приложений

| Тип | Kind | Для кого | Что создаётся |
|-----|------|----------|---------------|
| Простое | `Application` | Разработчик | Namespace, Deployment, Service, ConfigMap, Ingress |
| С БД | `ApplicationWithDB` | Разработчик | + PostgreSQL, PVC, миграции |
| Мульти-сервис | `MultiServiceApp` | Разработчик | N × (Deployment, Service, ConfigMap, Ingress) |
| Тестовое окружение | `TestEnvironment` | **Тестировщик** | **Все типы вместе в одном YAML** |

### Команды для проверки

```bash
# Все IDP ресурсы
kubectl get applications.idp.example.com -A
kubectl get applicationwithdbs.idp.example.com -A
kubectl get multiserviceapps.idp.example.com -A
kubectl get testenvironments.idp.example.com -A

# Все поды тестового окружения
kubectl get pods -A | grep "^qa-"
```

**Репозиторий:** https://github.com/agud97/idp-app-v1
