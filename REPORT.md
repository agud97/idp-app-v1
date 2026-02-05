# Отчёт о создании IDP (Internal Developer Platform)

## Обзор

Создана платформа IDP на базе Kubernetes с использованием ArgoCD и Crossplane, позволяющая разработчикам деплоить приложения через простые YAML-файлы.

---

## 1. Подключение к репозиторию и кластеру

### Клонирование репозитория с GitHub токеном
```bash
git clone https://agud97:<TOKEN>@github.com/agud97/idp-app-v1.git /root/proj/cross/idp-app-v1
```

### Проверка доступа к push
```bash
cd /root/proj/cross/idp-app-v1
git remote -v
git push --dry-run
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

### Создание namespace
```bash
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl create namespace argocd
```

### Установка ArgoCD из официальных манифестов
```bash
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side=true --force-conflicts
```

### Ожидание готовности
```bash
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl wait \
  --for=condition=available --timeout=120s \
  deployment -l app.kubernetes.io/part-of=argocd -n argocd
```

### Настройка доступа к репозиторию
```bash
# Создание секрета с credentials
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl create secret generic repo-idp-app-v1 \
  --namespace=argocd \
  --from-literal=type=git \
  --from-literal=url=https://github.com/agud97/idp-app-v1.git \
  --from-literal=username=agud97 \
  --from-literal=password=<TOKEN>

# Добавление label для ArgoCD
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
    helm:
      values: |
        args:
          - --enable-external-secret-stores
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

### Применение и проверка
```bash
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl apply -f applications/crossplane.yaml

# Проверка статуса
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl get applications -n argocd
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl get pods -n crossplane-system
```

---

## 4. Установка Crossplane Kubernetes Provider

### Создан файл `crossplane/providers/provider-kubernetes.yaml`
```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-kubernetes
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.13.0
  controllerConfigRef:
    name: provider-kubernetes-config
---
apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: provider-kubernetes-config
spec:
  serviceAccountName: provider-kubernetes
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: provider-kubernetes
  namespace: crossplane-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: provider-kubernetes-cluster-admin
subjects:
  - kind: ServiceAccount
    name: provider-kubernetes
    namespace: crossplane-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### Создан файл `crossplane/providers/provider-kubernetes-config.yaml`
```yaml
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: InjectedIdentity
```

### Создан файл `crossplane/providers/function-go-templating.yaml`
```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-go-templating
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-go-templating:v0.5.0
```

### Проверка провайдеров
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

## 5. Создание IDP - Простое приложение (Application)

### Создан XRD `crossplane/idp/definition.yaml`
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xapplications.idp.example.com
spec:
  group: idp.example.com
  names:
    kind: XApplication
    plural: xapplications
  claimNames:
    kind: Application
    plural: applications
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - name
                - image
              properties:
                name:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
                  default: 1
                port:
                  type: integer
                  default: 80
                env:
                  type: array
                  items:
                    type: object
                    properties:
                      name:
                        type: string
                      value:
                        type: string
                secrets:
                  type: array
                  items:
                    type: object
                    properties:
                      name:
                        type: string
                      value:
                        type: string
                ingress:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                    host:
                      type: string
```

### Создан Composition `crossplane/idp/composition.yaml`

Composition использует `function-go-templating` для генерации:
- Namespace
- Deployment
- Service
- ConfigMap (если есть env)
- Secret (если есть secrets)
- Ingress (если enabled)

---

## 6. Создание IDP - Приложение с БД (ApplicationWithDB)

### Создан XRD `crossplane/idp/definition-with-db.yaml`

Дополнительные поля:
- `database.enabled`, `database.engine`, `database.version`, `database.storage`, `database.dbName`, `database.username`, `database.password`
- `migration.enabled`, `migration.image`, `migration.command`

### Создан Composition `crossplane/idp/composition-with-db.yaml`

Генерирует дополнительно:
- PostgreSQL Deployment
- PostgreSQL Service
- PersistentVolumeClaim
- DB Credentials Secret (с DATABASE_URL)
- Init-контейнер для миграций

---

## 7. Установка OpenEBS для локального хранилища

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
          basePath: /var/openebs/local
        localpv:
          image:
            registry: docker.io/
            repository: openebs/provisioner-localpv
            tag: 4.1.0
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

### Проверка StorageClass
```bash
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl get storageclass
```

**Результат:**
```
NAME                         PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE
openebs-hostpath (default)   openebs.io/local   Delete          WaitForFirstConsumer
```

---

## 8. Создание ArgoCD Applications для IDP

### `applications/crossplane-provider-kubernetes.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-provider-kubernetes
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/agud97/idp-app-v1.git
    path: crossplane/providers
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### `applications/crossplane-idp.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-idp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/agud97/idp-app-v1.git
    path: crossplane/idp
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### `applications/developer-apps.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: developer-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/agud97/idp-app-v1.git
    path: developer-apps
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 9. Тестирование

### Тест 1: Простое приложение

**Создан файл `developer-apps/test-simple-app.yaml`:**
```yaml
apiVersion: idp.example.com/v1alpha1
kind: Application
metadata:
  name: test-simple-app
  namespace: default
spec:
  name: test-simple-app
  image: nginx:alpine
  replicas: 2
  port: 80
  env:
    - name: APP_NAME
      value: test-simple-app
    - name: ENVIRONMENT
      value: testing
  ingress:
    enabled: true
    host: test-simple-app.example.com
```

**Коммит и push:**
```bash
git add .
git commit -m "Add test-simple-app to verify IDP"
git push origin main
```

**Проверка:**
```bash
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl get applications.idp.example.com -A
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl get all -n test-simple-app
```

**Результат:**
```
NAME              SYNCED   READY
test-simple-app   True     True

pod/test-simple-app-xxx   1/1     Running
pod/test-simple-app-xxx   1/1     Running
deployment.apps/test-simple-app   2/2
service/test-simple-app   ClusterIP
```

---

### Тест 2: Приложение с БД и миграцией

**Создан файл `developer-apps/test-app-with-db.yaml`:**
```yaml
apiVersion: idp.example.com/v1alpha1
kind: ApplicationWithDB
metadata:
  name: test-app-with-db
  namespace: default
spec:
  name: test-app-with-db
  image: nginx:alpine
  replicas: 2
  port: 8080
  env:
    - name: APP_NAME
      value: test-app-with-db
  database:
    enabled: true
    engine: postgres
    version: "15"
    storage: "1Gi"
    dbName: testdb
    username: testuser
    password: testpass123
  migration:
    enabled: true
    image: postgres:15
    command:
      - /bin/sh
      - -c
      - "echo 'Starting migration' && until pg_isready -h test-app-with-db-db -p 5432 -U testuser; do sleep 2; done && PGPASSWORD=$POSTGRES_PASSWORD psql -h test-app-with-db-db -U testuser -d testdb -c 'CREATE TABLE IF NOT EXISTS users (id SERIAL PRIMARY KEY, name VARCHAR(255));' && echo 'Migration done'"
  ingress:
    enabled: true
    host: test-app-with-db.example.com
```

**Коммит и push:**
```bash
git add .
git commit -m "Add test-app-with-db to verify IDP with database"
git push origin main
```

**Проверка:**
```bash
# Статус приложения
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl get applicationwithdbs -A

# Все ресурсы
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl get all,pvc -n test-app-with-db

# Проверка init-контейнера (миграции)
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl describe pod -n test-app-with-db -l app=test-app-with-db | grep -A 20 "Init Containers:"

# Проверка созданной таблицы в БД
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl exec -n test-app-with-db deploy/test-app-with-db-db -- psql -U testuser -d testdb -c "\dt"
```

**Результат:**
```
# Приложение
NAME               SYNCED   READY
test-app-with-db   True     True

# Поды
pod/test-app-with-db-xxx      1/1     Running   # App replica 1
pod/test-app-with-db-xxx      1/1     Running   # App replica 2
pod/test-app-with-db-db-xxx   1/1     Running   # PostgreSQL

# Init-контейнер
State:      Terminated
Reason:     Completed
Exit Code:  0

# Таблица в БД
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | users | table | testuser
```

---

## 10. Финальная структура репозитория

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
│   │   ├── definition.yaml
│   │   ├── definition-with-db.yaml
│   │   ├── composition.yaml
│   │   └── composition-with-db.yaml
│   └── providers/
│       ├── provider-kubernetes.yaml
│       ├── provider-kubernetes-config.yaml
│       └── function-go-templating.yaml
└── developer-apps/
    ├── test-simple-app.yaml
    └── test-app-with-db.yaml
```

---

## 11. Все ArgoCD Applications

```bash
KUBECONFIG=/root/proj/cross/kubeconfig_6005021 kubectl get applications -n argocd
```

**Результат:**
```
NAME                             SYNC STATUS   HEALTH STATUS
crossplane                       Synced        Healthy
crossplane-idp                   Synced        Healthy
crossplane-provider-kubernetes   Synced        Healthy
developer-apps                   Synced        Healthy
openebs                          Synced        Healthy
```

---

## 12. Список коммитов

```bash
git log --oneline
```

```
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
| ArgoCD | Установлен и настроен |
| Crossplane | Установлен (v1.15.0) |
| Kubernetes Provider | Установлен (v0.13.0) |
| Function Go-Templating | Установлен (v0.5.0) |
| OpenEBS | Установлен (StorageClass default) |
| IDP Application | Работает |
| IDP ApplicationWithDB | Работает |
| README | Добавлен |

**Репозиторий:** https://github.com/agud97/idp-app-v1
