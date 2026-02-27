# Отчёт: Установка и настройка Backstage IDP

**Дата:** 2026-02-27 — 2026-02-28
**Кластер:** Kubernetes на 194.58.110.52:6443
**GitOps-репозиторий:** github.com/agud97/idp-app-v1
**Итоговая версия:** backstage/backstage Helm chart 2.6.3

---

## 1. Цель и архитектура

Backstage развёртывается как **Internal Developer Portal (IDP)**: платформенная команда предоставляет шаблон `Create Test Environment`, разработчики через UI создают Pull Request, Crossplane по этому PR разворачивает реальные Kubernetes-ресурсы (Deployments, Pods). На момент завершения Backstage умеет:

- Создавать PR с TestEnvironment CR через Software Template
- Отображать Kubernetes-ресурсы (Pods, Deployments, ReplicaSets) каждого окружения во вкладке «Kubernetes»
- Автоматически регистрировать новые Component-сущности при scaffold

**Схема взаимодействия:**
```
Backstage UI
  → Scaffolder (fetch:template + publish:github:pull-request + catalog:register)
    → PR в idp-app-v1
      → merge → ArgoCD видит TestEnvironment CR
        → Crossplane создаёт Pods/Deployments с label environment=<name>
          → Backstage K8s plugin показывает ресурсы
```

---

## 2. Фаза 1 — Первичная установка (backstage/backstage 2.3.0)

### 2.1 ArgoCD Application

Backstage устанавливался через app-of-apps: создан файл `applications/backstage.yaml` в gitops-репозитории.

**Начальная конфигурация (упрощённо):**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backstage
  namespace: argocd
spec:
  source:
    repoURL: https://backstage.github.io/charts
    chart: backstage
    targetRevision: "2.3.0"
    helm:
      values: |
        backstage:
          image:
            registry: ghcr.io
            repository: backstage/backstage
            tag: latest
          appConfig:
            auth:
              environment: development
              providers:
                guest:
                  dangerouslyAllowOutsideDevelopment: true
            integrations:
              github:
                - host: github.com
                  token: ${GITHUB_TOKEN}
            catalog:
              locations:
                - type: url
                  target: https://github.com/agud97/idp-app-v1/blob/main/platform/backstage/catalog-info.yaml
            scaffolder:
              defaultAuthor:
                name: Backstage IDP
                email: backstage@idp.local
          extraEnvVars:
            - name: GITHUB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: backstage-secrets
                  key: GITHUB_TOKEN
        postgresql:
          enabled: true
          auth:
            password: "backstage-pg-pass"
            username: backstage
            database: backstage_plugin_catalog
          primary:
            persistence:
              storageClass: openebs-hostpath
              size: 5Gi
  destination:
    server: https://kubernetes.default.svc
    namespace: backstage
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Secret с GitHub-токеном** (создан вручную, не в git):
```bash
kubectl create secret generic backstage-secrets \
  --from-literal=GITHUB_TOKEN=$(cat ~/proj/cross/token-gh) \
  -n backstage
```

---

## 3. Проблема 1 — Удалённый образ PostgreSQL (bitnami)

### Симптом
После применения `applications/backstage.yaml` pod не стартовал. Ошибка в логах ArgoCD:

```
ImagePullBackOff: Failed to pull image
"docker.io/bitnami/postgresql:15.4.0-debian-11-r10":
rpc error: ... manifest unknown: manifest unknown
```

Чарт `backstage/backstage 2.3.0` зависел от `bitnami/postgresql`, который ссылался на образ `bitnami/postgresql:15.4.0-debian-11-r10`. Bitnami удалила старые теги из Docker Hub.

### Диагностика
```bash
kubectl describe pod backstage-postgresql-0 -n backstage
# → ImagePullBackOff

kubectl get events -n backstage --sort-by='.lastTimestamp'
# → Failed to pull image "bitnami/postgresql:15.4.0-debian-11-r10"
```

### Решение
Обновить чарт до `2.6.3` (последняя стабильная) и явно зафиксировать tag PostgreSQL на версию 16, которая тянется из `bitnamilegacy` репозитория:

```yaml
# applications/backstage.yaml — изменения
spec:
  source:
    targetRevision: "2.6.3"   # было 2.3.0
    helm:
      values: |
        postgresql:
          enabled: true
          image:
            tag: "16"          # добавлено явно
          auth:
            password: "backstage-pg-pass"
```

**Коммит:** `9db8669 fix(backstage): upgrade chart to 2.6.3, pin postgresql image to tag 16`

После пуша в git ArgoCD автоматически синхронизировался и PostgreSQL поднялся.

---

## 4. Проблема 2 — Race condition при инициализации БД (duplicate key)

### Симптом
Backstage pod падал при старте с ошибкой в логах:

```
error: duplicate key value violates unique constraint "knex_migrations_pkey"
detail: Key (id)=(1) already exists.
```

### Причина
При деплое нового чарта Kubernetes поднял два ReplicaSet одновременно: старый (от 2.3.0) и новый (2.6.3). Оба пода пытались запустить миграции базы данных параллельно. При конкурентной вставке в таблицу `knex_migrations` возникал конфликт.

### Диагностика
```bash
kubectl get rs -n backstage
# NAME                        DESIRED   CURRENT   READY
# backstage-7d9f8c4b9         1         1         0   ← новый (падает)
# backstage-5c8b4f6d2         1         1         1   ← старый (держится)

kubectl logs backstage-7d9f8c4b9-xxx -n backstage | grep "duplicate key"
# duplicate key value violates unique constraint "knex_migrations_pkey"
```

### Решение
Вручную остановить старый ReplicaSet (обнулить replicas), затем пересоздать новый pod:

```bash
# Найти старый RS
kubectl get rs -n backstage

# Масштабировать старый RS до 0
kubectl scale rs backstage-5c8b4f6d2 --replicas=0 -n backstage

# Удалить падающий pod нового RS — K8s пересоздаст его без конкурента
kubectl delete pod backstage-7d9f8c4b9-xxx -n backstage

# Backstage запустился нормально
kubectl get pods -n backstage
# NAME                          READY   STATUS    RESTARTS
# backstage-7d9f8c4b9-yyy       1/1     Running   0
```

---

## 5. Фаза 2 — Software Template для создания окружений

### 5.1 Структура шаблона

Создана структура в `platform/backstage/templates/test-environment/`:

```
template.yaml          ← определение шаблона
skeleton/
  developer-apps/
    ${{ values.envName }}.yaml    ← TestEnvironment CR (Crossplane)
```

**template.yaml — ключевые шаги:**
```yaml
steps:
  - id: fetch-template
    action: fetch:template
    input:
      url: ./skeleton
      values:
        envName: ${{ parameters.envName }}
        enableFrontend: ${{ parameters.enableFrontend }}
        # ...

  - id: publish-pr
    action: publish:github:pull-request
    input:
      repoUrl: github.com?owner=agud97&repo=idp-app-v1
      title: "feat: create test environment ${{ parameters.envName }}"
      branchName: backstage/env-${{ parameters.envName }}
      commitMessage: "feat: create test environment ${{ parameters.envName }}"
```

**skeleton/developer-apps/${{ values.envName }}.yaml:**
```yaml
apiVersion: idp.example.com/v1alpha1
kind: TestEnvironment
metadata:
  name: ${{ values.envName }}-env
  namespace: default
spec:
  name: ${{ values.envName }}
  frontend:
    enabled: ${{ values.enableFrontend }}
    replicas: ${{ values.frontendReplicas | default(2) }}
  backend:
    enabled: ${{ values.enableBackend }}
    database:
      password: ${{ values.backendDbPassword | default('changeme') }}
  platform:
    enabled: ${{ values.enablePlatform }}
```

**catalog-info.yaml** (регистрирует шаблон):
```yaml
apiVersion: backstage.io/v1alpha1
kind: Location
metadata:
  name: idp-templates
spec:
  targets:
    - ./templates/test-environment/template.yaml
```

---

## 6. Проблема 3 — Scaffolder: дубликат PR ("A pull request already exists")

### Симптом
При повторном запуске шаблона для того же окружения (например, при изменении параметров) scaffolder завершался ошибкой:

```
Error: A pull request already exists for branch backstage/env-demo123
```

### Причина
`publish:github:pull-request` по умолчанию не обновляет существующие PR — при попытке создать PR для уже существующей ветки GitHub API возвращал ошибку 422.

### Решение
Добавить параметр `update: true` в шаг `publish-pr`:

```yaml
- id: publish-pr
  action: publish:github:pull-request
  input:
    repoUrl: github.com?owner=agud97&repo=idp-app-v1
    title: "feat: create test environment ${{ parameters.envName }}"
    branchName: backstage/env-${{ parameters.envName }}
    commitMessage: "feat: create test environment ${{ parameters.envName }}"
    update: true    # ← добавлено: обновляет существующий PR вместо ошибки
```

**Коммит:** `e926670 fix(backstage): add update:true to prevent duplicate PR error`

---

## 7. Фаза 3 — Kubernetes Plugin + каталог окружений

### 7.1 Задача
После merge PR Crossplane создавал Pods и Deployments, но их нельзя было видеть в Backstage. Нужно:
1. Показывать реальные K8s-ресурсы во вкладке «Kubernetes» для каждого окружения
2. Автоматически регистрировать Component-сущности в каталоге

### 7.2 RBAC — ServiceAccount для Backstage

Создан набор RBAC-ресурсов **напрямую через kubectl** (не через GitOps — секреты не хранятся в git):

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backstage-k8s-reader
  namespace: backstage
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: backstage-k8s-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "namespaces", "events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["idp.example.com"]
    resources: ["testenvironments"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: backstage-k8s-reader
subjects:
  - kind: ServiceAccount
    name: backstage-k8s-reader
    namespace: backstage
roleRef:
  kind: ClusterRole
  name: backstage-k8s-reader
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Secret
metadata:
  name: backstage-k8s-sa-token
  namespace: backstage
  annotations:
    kubernetes.io/service-account.name: backstage-k8s-reader
type: kubernetes.io/service-account-token
EOF
```

Извлечение токена и добавление в существующий secret:
```bash
# Подождать, пока Kubernetes заполнит токен в secret
sleep 3

# Получить токен в base64
K8S_TOKEN=$(kubectl get secret backstage-k8s-sa-token -n backstage \
  -o jsonpath='{.data.token}')

# Добавить ключ K8S_SA_TOKEN в backstage-secrets
kubectl patch secret backstage-secrets -n backstage --type='json' \
  -p="[{\"op\":\"add\",\"path\":\"/data/K8S_SA_TOKEN\",\"value\":\"${K8S_TOKEN}\"}]"

# Проверка
kubectl get secret backstage-secrets -n backstage \
  -o jsonpath='{.data}' | python3 -c \
  "import sys,json; d=json.load(sys.stdin); print(list(d.keys()))"
# ['GITHUB_TOKEN', 'K8S_SA_TOKEN']
```

### 7.3 Обновление Backstage через GitOps

#### Проблема 4 — ArgoCD selfHeal откатывает прямые изменения

**Первая попытка** — прямой `kubectl patch` ArgoCD Application:
```bash
kubectl patch application backstage -n argocd \
  --type=json -p="[{\"op\":\"replace\",\"path\":\"/spec/source/helm/values\",\"value\":\"...новые values...\"}]"
```

**Результат:** ArgoCD через ~30 секунд откатил изменения обратно, так как Application управляется app-of-apps (`applications/apps.yaml`) с `selfHeal: true`. Pod перезапустился с новым конфигом, но потом ArgoCD снова применил старую версию из git.

**Диагностика отката:**
```bash
# Проверяем env vars в pod
kubectl exec -n backstage deployment/backstage -- env | grep K8S
# BACKSTAGE_K8S_TOKEN отсутствует → ArgoCD откатил Application

# Проверяем что в Application spec
kubectl get application backstage -n argocd \
  -o jsonpath='{.spec.source.helm.values}' | grep -c "kubernetes"
# 0 → конфиг откатился
```

**Правило:** При `selfHeal: true` в app-of-apps все изменения ArgoCD Application **обязательно делать только через git**. `kubectl patch` на Application работает временно — до следующей reconciliation.

**Решение:** Обновить `applications/backstage.yaml` в git-репозитории:

```yaml
# Добавленные блоки в helm.values:
kubernetes:
  serviceLocatorMethod:
    type: multiTenant
  clusterLocatorMethods:
    - type: config
      clusters:
        - url: https://194.58.110.52:6443
          name: main
          authProvider: serviceAccount
          serviceAccountToken: ${BACKSTAGE_K8S_TOKEN}
          skipTLSVerify: true
  customResources:
    - group: 'idp.example.com'
      apiVersion: 'v1alpha1'
      plural: 'testenvironments'

# В catalog.locations добавлены:
- type: url
  target: https://github.com/agud97/idp-app-v1/blob/main/developer-apps/catalog/qa-environment.yaml
- type: url
  target: https://github.com/agud97/idp-app-v1/blob/main/developer-apps/catalog/demo123.yaml

# В extraEnvVars добавлено:
- name: BACKSTAGE_K8S_TOKEN
  valueFrom:
    secretKeyRef:
      name: backstage-secrets
      key: K8S_SA_TOKEN
```

**Коммит и пуш:**
```bash
git add applications/backstage.yaml platform/backstage/templates/test-environment/template.yaml
git commit -m "feat: configure Backstage kubernetes plugin via GitOps"
GIT_TOKEN=$(cat ~/proj/cross/token-gh)
git push "https://agud97:${GIT_TOKEN}@github.com/agud97/idp-app-v1.git" main
```

#### Проблема 5 — app-of-apps не подхватил новый коммит

После пуша `backstage` Application всё ещё показывал старые значения. `apps` (app-of-apps) был OutOfSync, но selfHeal не срабатывал автоматически достаточно быстро.

**Диагностика:**
```bash
kubectl get application -n argocd -o wide | grep -E "NAME|apps|backstage"
# apps    OutOfSync  Healthy  3baad05...  ← старый коммит, а нужен 65993f1
# backstage  Synced  Healthy  2.6.3
```

**Решение — принудительный refresh и sync app-of-apps:**
```bash
# Hard refresh — сбросить кэш git
kubectl annotate application apps -n argocd \
  argocd.argoproj.io/refresh="hard" --overwrite

# Запустить sync operation
kubectl patch application apps -n argocd --type=merge -p '{
  "operation": {
    "sync": {"revision": "HEAD", "prune": true},
    "initiatedBy": {"username": "admin"}
  }
}'

sleep 20
kubectl get application -n argocd | grep backstage
# backstage  Synced  Healthy  2.6.3  ← теперь с новыми values
```

**Проверка что BACKSTAGE_K8S_TOKEN появился в pod:**
```bash
kubectl exec -n backstage deployment/backstage -- env | grep BACKSTAGE_K8S_TOKEN
# BACKSTAGE_K8S_TOKEN=eyJhbGciOiJSUzI1NiIs...  ✓
```

---

## 8. Проблема 6 — GitHub Catalog Provider не работает

### Симптом
В `applications/backstage.yaml` была попытка использовать `catalog.providers.github` для автоматического обнаружения файлов `developer-apps/catalog/*.yaml`:

```yaml
catalog:
  providers:
    github:
      default:
        organization: 'agud97'
        catalogPath: '/developer-apps/catalog/*.yaml'
        filters:
          repository: 'idp-app-v1'
        schedule:
          frequency: { minutes: 1 }
          timeout: { minutes: 3 }
```

Scheduled task для GitHub provider **не регистрировался** в логах. Компоненты не появлялись в каталоге через минуту и даже через 5 минут.

### Диагностика
```bash
# Проверка зарегистрированных scheduled tasks при старте
kubectl logs deployment/backstage -n backstage 2>&1 | grep "Registered scheduled task"
# catalog_orphan_cleanup
# notification-cleaner
# search_index_software_catalog
# search_index_techdocs
# ← НЕТ github_catalog_entity_provider
```

### Причина
Образ `ghcr.io/backstage/backstage:latest` (официальный demo-образ Backstage) **не включает** пакет `@backstage/plugin-catalog-backend-module-github`. Секция `catalog.providers.github` в app-config молча игнорируется — модуль должен быть установлен в коде приложения и явно зарегистрирован.

### Решение
Отказаться от GitHub provider, использовать два подхода:

1. **Статические locations** в `catalog.locations` для уже существующих окружений:
```yaml
catalog:
  locations:
    - type: url
      target: https://github.com/agud97/idp-app-v1/blob/main/developer-apps/catalog/qa-environment.yaml
    - type: url
      target: https://github.com/agud97/idp-app-v1/blob/main/developer-apps/catalog/demo123.yaml
```

2. **`catalog:register` action** в scaffolder-шаблоне для новых окружений — регистрирует Location-сущность в БД Backstage динамически при scaffold:
```yaml
- id: register-catalog
  name: Register Component in Catalog
  action: catalog:register
  input:
    catalogInfoUrl: https://github.com/agud97/idp-app-v1/blob/main/developer-apps/catalog/${{ parameters.envName }}.yaml
    optional: true
```

> **Примечание:** `optional: true` позволяет шагу не падать, если файл ещё не появился на ветке `main` (PR ещё не merged). После merge файл появляется по URL, Backstage при следующей refresh-итерации читает сущность и регистрирует Component.

---

## 9. Каталог Component-сущностей для окружений

### Skeleton шаблона

**Проблема:** Исходный `skeleton/catalog-info.yaml` помещался в корень репозитория при каждом scaffold. Если бы создавалось второе окружение — файл перезаписывался бы.

**Решение:** Переместить в `skeleton/developer-apps/catalog/${{ values.envName }}.yaml`:

```
skeleton/
  developer-apps/
    ${{ values.envName }}.yaml           ← TestEnvironment CR
    catalog/
      ${{ values.envName }}.yaml         ← Component (новое место)
```

Содержимое `skeleton/developer-apps/catalog/${{ values.envName }}.yaml`:
```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${{ values.envName }}-env
  description: Test environment ${{ values.envName }}
  annotations:
    backstage.io/managed-by-location: "url:https://github.com/agud97/idp-app-v1/blob/main/developer-apps/${{ values.envName }}.yaml"
    backstage.io/kubernetes-label-selector: 'environment=${{ values.envName }}'
spec:
  type: environment
  lifecycle: experimental
  owner: platform-team
```

Аннотация `backstage.io/kubernetes-label-selector: 'environment=<name>'` указывает Kubernetes plugin, по какому label искать ресурсы в кластере.

### Catalog-записи для существующих окружений

Созданы вручную в `developer-apps/catalog/`:

**developer-apps/catalog/qa-environment.yaml:**
```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: qa-environment-env
  description: Test environment qa-environment
  annotations:
    backstage.io/managed-by-location: "url:https://github.com/agud97/idp-app-v1/blob/main/developer-apps/qa-environment.yaml"
    backstage.io/kubernetes-label-selector: 'environment=qa'
spec:
  type: environment
  lifecycle: experimental
  owner: platform-team
```

---

## 10. Label `environment` в Crossplane compositions

### Проблема
Kubernetes plugin использует label selector `environment=<name>` для поиска ресурсов. Существующие Pods и Deployments имели только `app=frontend` / `app=backend` — без `environment`.

### Изменения в compositions

**crossplane/idp/apps/frontend/composition.yaml** — Deployment:
```yaml
# Было:
metadata:
  labels:
    app: {{ $appName }}
template:
  metadata:
    labels:
      app: {{ $appName }}

# Стало:
metadata:
  labels:
    app: {{ $appName }}
    {{- if ne $envName "" }}
    environment: {{ $envName }}
    {{- end }}
template:
  metadata:
    labels:
      app: {{ $appName }}
      {{- if ne $envName "" }}
      environment: {{ $envName }}
      {{- end }}
```

Аналогично для `crossplane/idp/apps/backend/composition.yaml` — оба Deployment: `backend` и `backend-db`.

**Важно:** `selector.matchLabels` **не менялся** (оставлен только `app: <name>`), так как селектор Deployment в Kubernetes immutable. Добавление label только в `metadata.labels` и `template.metadata.labels` безопасно — Crossplane обновит существующие Deployments rolling update'ом, pods получат новый label.

---

## 11. Проверка работы

### Итоговый статус Backstage:
```bash
kubectl get pods -n backstage
# NAME                         READY   STATUS    RESTARTS   AGE
# backstage-584f4649c8-f9wjk   1/1     Running   0          ~12m
# backstage-postgresql-0        1/1     Running   0          ~3h

kubectl get application backstage -n argocd \
  -o jsonpath='{.status.sync.status} {.status.health.status}'
# Synced Healthy
```

### Проверка каталога через API:
```bash
# port-forward
kubectl port-forward svc/backstage 17007:7007 -n backstage &

# Получение токена
TOKEN=$(curl -s -X POST http://localhost:17007/api/auth/guest/refresh \
  -H "Content-Type: application/json" -d '{}' | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(d['backstageIdentity']['token'])")

# Список сущностей в каталоге
curl -s "http://localhost:17007/api/catalog/entities" \
  -H "Authorization: Bearer $TOKEN" | python3 -c "
import sys,json
for e in json.load(sys.stdin):
    print(e['kind'] + '/' + e['metadata']['name'])
"
# Location/generated-b076f861...
# Component/demo123-env          ← ✓
# Component/qa-environment-env   ← ✓
# Template/test-environment      ← ✓
# Location/idp-templates
```

### Проверка Kubernetes plugin:
```bash
# POST запрос к workloads API
curl -s "http://localhost:17007/api/kubernetes/resources/workloads/query" \
  -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"entityRef":"component:default/demo123-env","auth":{}}' | python3 -c "
import sys,json
d=json.load(sys.stdin)
for item in d['items']:
    print('Cluster:', item['cluster']['name'])
    for res in item['resources']:
        for r in res['resources']:
            print(f'  {res[\"type\"]}/{r[\"metadata\"][\"name\"]} in {r[\"metadata\"][\"namespace\"]}')
"
# Cluster: main
#   pods/backend-5cff9d4cf9-mlbvp in demo123-backend
#   pods/backend-db-68cdd9d66f-fpzp8 in demo123-backend
#   pods/frontend-6d64bbcd76-z45d7 in demo123-frontend
#   deployments/backend in demo123-backend
#   deployments/backend-db in demo123-backend
#   deployments/frontend in demo123-frontend
#   replicasets/backend-5cff9d4cf9 in demo123-backend
#   replicasets/backend-db-68cdd9d66f in demo123-backend
#   replicasets/frontend-6d64bbcd76 in demo123-frontend
#   customresources/demo123-env in default    ← TestEnvironment CR тоже виден!
```

---

## 12. Итоговая архитектура файлов

### В gitops-репозитории (idp-app-v1):

```
applications/
  backstage.yaml                     ← ArgoCD Application (Helm chart 2.6.3)

platform/backstage/
  catalog-info.yaml                  ← Location: регистрирует шаблоны
  templates/test-environment/
    template.yaml                    ← Scaffolder template
    skeleton/
      developer-apps/
        ${{ values.envName }}.yaml   ← TestEnvironment CR
        catalog/
          ${{ values.envName }}.yaml ← Component с k8s-label-selector

developer-apps/
  qa-environment.yaml                ← TestEnvironment CR для QA
  demo123.yaml                       ← TestEnvironment CR для demo123
  catalog/
    qa-environment.yaml              ← Component для QA
    demo123.yaml                     ← Component для demo123

crossplane/idp/apps/
  frontend/composition.yaml          ← +environment label в pod template
  backend/composition.yaml           ← +environment label в pod template
```

### Kubernetes ресурсы (созданы напрямую, не в git):

```
backstage namespace:
  ServiceAccount: backstage-k8s-reader
  Secret: backstage-k8s-sa-token (type: kubernetes.io/service-account-token)
  Secret: backstage-secrets (GITHUB_TOKEN + K8S_SA_TOKEN)

cluster-wide:
  ClusterRole: backstage-k8s-reader
  ClusterRoleBinding: backstage-k8s-reader
```

---

## 13. Ключевые уроки

| # | Проблема | Решение |
|---|----------|---------|
| 1 | `bitnami/postgresql:15.4.0-debian-11-r10` удалён | Обновить chart 2.3.0→2.6.3, `postgresql.image.tag: "16"` |
| 2 | Race condition двух pods при миграции БД | Scale старый RS до 0, пересоздать pod |
| 3 | Дубликат PR при повторном scaffold | `update: true` в `publish:github:pull-request` |
| 4 | `kubectl patch` на Application откатывается ArgoCD | Всегда редактировать `applications/*.yaml` в git |
| 5 | app-of-apps не подхватил коммит | `kubectl annotate ... refresh=hard` + `kubectl patch ... sync` |
| 6 | `catalog.providers.github` молча игнорируется | Модуль не в образе; использовать статические locations + `catalog:register` |
| 7 | Kubernetes plugin не находит pods | Добавить label `environment` в Crossplane compositions |

---

## 14. Команды для эксплуатации

```bash
# Port-forward Backstage
kubectl port-forward svc/backstage 17007:7007 -n backstage

# Получить токен (guest auth)
TOKEN=$(curl -s -X POST http://localhost:17007/api/auth/guest/refresh \
  -H "Content-Type: application/json" -d '{}' | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['backstageIdentity']['token'])")

# Список Component-сущностей
curl -s "http://localhost:17007/api/catalog/entities?filter=kind=Component" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool | grep '"name"'

# K8s ресурсы для окружения
curl -s "http://localhost:17007/api/kubernetes/resources/workloads/query" \
  -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"entityRef":"component:default/demo123-env","auth":{}}'

# Принудительный sync ArgoCD
kubectl annotate application apps -n argocd \
  argocd.argoproj.io/refresh="hard" --overwrite
kubectl patch application apps -n argocd --type=merge -p \
  '{"operation":{"sync":{"revision":"HEAD","prune":true},"initiatedBy":{"username":"admin"}}}'

# Логи Backstage
kubectl logs deployment/backstage -n backstage --tail=100 -f

# Добавить новый environment в каталог (после создания developer-apps/catalog/NAME.yaml в git):
# Просто добавить URL в applications/backstage.yaml → catalog.locations
```
