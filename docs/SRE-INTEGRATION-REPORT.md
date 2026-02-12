# Отчет: SRE-интеграция для Kubernetes

**Дата:** 2026-02-12
**Репозиторий:** https://github.com/agud97/idp-app-v1

---

## Цель

Добавить observability-стек (метрики, логи, дашборды) и интегрировать его с HolmesGPT для автоматического расследования инцидентов. Всё работает через локальную LLM (LMStudio через tailscale VPN).

## Что было сделано

| Задача | Решение | Статус |
|--------|---------|--------|
| Метрики кластера | VictoriaMetrics k8s stack (VMSingle, VMAgent, VMAlert, Grafana) | Работает |
| Логи кластера | VictoriaLogs + vlagent collector (DaemonSet) | Работает |
| HolmesGPT + метрики | Обновление до v0.19.0, toolset `prometheus/metrics` | Работает |
| Custom runbooks | Через `additionalVolumes` в Helm values (без kubectl patch) | Работает |
| Автоматическое расследование | Alertmanager → webhook bridge → HolmesGPT `/api/investigate` | Работает |
| Визуализация | Grafana + плагин `victoriametrics-metrics-datasource` | Работает |

---

## Компоненты

### 1. VictoriaMetrics k8s stack (метрики)

**Файл:** `applications/vm-k8s-stack.yaml`
**Chart:** `victoria-metrics-k8s-stack` v0.45.0
**Namespace:** `monitoring`

| Компонент | Назначение | Service |
|-----------|-----------|---------|
| VMSingle | Хранилище метрик (15d retention, 50Gi) | `vmsingle-vm-k8s-stack-victoria-metrics-k8s-stack:8429` |
| VMAgent | Сбор метрик (scrape Kubernetes targets) | `vmagent-vm-k8s-stack-victoria-metrics-k8s-stack:8429` |
| VMAlert | Alerting rules | `vmalert-vm-k8s-stack-victoria-metrics-k8s-stack:8080` |
| VMAlertmanager | Маршрутизация алертов | `vmalertmanager-vm-k8s-stack-victoria-metrics-k8s-stack:9093` |
| Grafana | Дашборды | `vm-k8s-stack-grafana:80` |
| kube-state-metrics | Метрики K8s ресурсов | `vm-k8s-stack-kube-state-metrics:8080` |
| node-exporter | Метрики нод (DaemonSet x3) | `vm-k8s-stack-prometheus-node-exporter:9100` |
| VM Operator | CRD контроллер | `vm-k8s-stack-victoria-metrics-operator:8080` |

**Ключевые настройки:**

```yaml
vmsingle:
  spec:
    retentionPeriod: "15d"
    storage:
      storageClassName: openebs-hostpath
      resources:
        requests:
          storage: 50Gi

# ВАЖНО: VMAlertmanager CRD использует volumeClaimTemplate (не прямой PVC spec)
alertmanager:
  spec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: openebs-hostpath
          resources:
            requests:
              storage: 5Gi

grafana:
  plugins:
    - victoriametrics-metrics-datasource   # ID плагина (НЕ victoriametrics-datasource)
```

**ArgoCD sync options:**

```yaml
ignoreDifferences:
  - group: operator.victoriametrics.com
    kind: VMSingle
    jsonPointers: [/spec/storage]
  - group: operator.victoriametrics.com
    kind: VMAlertmanager
    jsonPointers: [/spec/storage]
syncOptions:
  - ServerSideApply=true
  - RespectIgnoreDifferences=true
```

### 2. VictoriaLogs (логи)

**Файл:** `applications/victoria-logs.yaml`
**Chart:** `victoria-logs-single` v0.11.26
**Namespace:** `victorialogs`

| Параметр | Значение |
|----------|---------|
| Retention | 30d |
| Storage | 50Gi (`openebs-hostpath`) |
| Service | `victoria-logs-victoria-logs-single-server.victorialogs.svc:9428` |
| StatefulSet | `victoria-logs-victoria-logs-single-server-0` |

### 3. VictoriaLogs Collector (сборщик логов)

**Файл:** `applications/victoria-logs-collector.yaml`
**Chart:** `victoria-logs-collector` v0.2.9
**Namespace:** `victorialogs`

Это **нативный vlagent** от VictoriaMetrics (не Vector). Работает как DaemonSet — по одному поду на каждую ноду кластера.

```yaml
remoteWrite:
  - url: "http://victoria-logs-victoria-logs-single-server.victorialogs.svc:9428"
```

Поды: 3 шт. (по количеству нод).

### 4. HolmesGPT v0.19.0

**Файл:** `applications/holmesgpt.yaml`
**Chart:** `holmes` v0.19.0 (robusta-charts)
**Namespace:** `holmesgpt`

| Параметр | Значение |
|----------|---------|
| Модель | `openai/qwen3-coder-30b-a3b-instruct-mlx` |
| LLM endpoint | `http://llm-proxy.llm-proxy.svc.cluster.local:8080/v1` |
| Service | `holmesgpt-holmes.holmesgpt.svc:80` |

**Активные toolsets:**

| Toolset | Назначение | Конфигурация |
|---------|-----------|--------------|
| `kubernetes/core` | Состояние ресурсов (kubectl get/describe) | — |
| `kubernetes/logs` | Логи подов (kubectl logs) | — |
| `prometheus/metrics` | PromQL-запросы к VictoriaMetrics | `prometheus_url: http://vmsingle-...svc:8429` |
| `runbook` | Автоподбор runbooks по regex-паттернам | — |
| `bash` | Выполнение shell-команд (с allow/deny list) | — |

**Custom runbooks (через `additionalVolumes`):**

Chart `holmes` 0.19.0 поддерживает `additionalVolumes`/`additionalVolumeMounts` — kubectl patch больше не нужен.

```yaml
additionalVolumes:
  - name: custom-runbooks
    configMap:
      name: custom-runbooks
  - name: sre-runbooks
    configMap:
      name: sre-runbooks
additionalVolumeMounts:
  - name: custom-runbooks
    mountPath: /app/holmes/plugins/runbooks/custom-runbooks.yaml
    subPath: custom-runbooks.yaml
  - name: sre-runbooks
    mountPath: /app/holmes/plugins/runbooks/sre-runbooks.yaml
    subPath: sre-runbooks.yaml
```

Загруженные runbooks:

| Файл | Источник | Runbooks |
|------|----------|----------|
| `jira.yaml` | Builtin | Jira issues |
| `kube-prometheus-stack.yaml` | Builtin | KubeSchedulerDown, KubeControllerManagerDown и др. |
| `custom-runbooks.yaml` | ConfigMap | Tailscale/LLM-proxy, Pod failures, LLM connectivity |
| `sre-runbooks.yaml` | ConfigMap | OOMKilled, CrashLoopBackOff, High CPU, HTTP 5xx, Disk, Node NotReady |

### 5. Alertmanager → HolmesGPT Bridge

**Файл:** `platform/holmesgpt/alertmanager-webhook.yaml`

Python HTTP-сервис, который:
1. Принимает webhook от VMAlertmanager на порту `:9095`
2. Парсит firing alerts
3. Вызывает `POST /api/investigate` на HolmesGPT с title, description, labels
4. Логирует результаты расследования

**Файл:** `platform/holmesgpt/vmalertmanagerconfig.yaml`

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMAlertmanagerConfig
spec:
  route:
    receiver: holmesgpt
    matchers:
      - severity =~ "critical|warning"
    group_wait: 30s
    group_interval: 5m
    repeat_interval: 4h
  receivers:
    - name: holmesgpt
      webhook_configs:
        - url: "http://holmes-alertmanager-bridge.holmesgpt.svc:9095/webhook"
          send_resolved: false
```

---

## Архитектура

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                            │
│                                                                      │
│  ┌──────────────── monitoring ──────────────────┐                    │
│  │ VMSingle :8429   VMAgent     VMAlert         │                    │
│  │ VMAlertmanager   Grafana :80  node-exporter  │                    │
│  │ kube-state-metrics  VM Operator              │                    │
│  └─────────────┬────────────────────┬───────────┘                    │
│                │ PromQL             │ webhook                        │
│                │                    ▼                                 │
│  ┌──── victorialogs ────┐  ┌──── holmesgpt ─────────────────────┐   │
│  │ VictoriaLogs :9428   │  │ HolmesGPT :80 (v0.19.0)           │   │
│  │ vlagent collector x3 │  │ alertmanager-bridge :9095           │   │
│  └──────────────────────┘  │                                     │   │
│                            │ toolsets:                            │   │
│                            │  - kubernetes/core (kubectl)        │   │
│                            │  - kubernetes/logs (kubectl logs)   │   │
│                            │  - prometheus/metrics (PromQL→VM)   │   │
│                            │  - runbook (regex match)            │   │
│                            │  - bash (shell commands)            │   │
│                            └──────────┬──────────────────────────┘   │
│                                       │                              │
│                                       ▼                              │
│                            ┌──────────────────┐                      │
│                            │   llm-proxy :8080│                      │
│                            │   (nginx+tailscale)                     │
│                            └────────┬─────────┘                      │
└─────────────────────────────────────┼────────────────────────────────┘
                                      │ tailscale VPN
                                      ▼
                            ┌──────────────────┐
                            │  Ноутбук/Сервер  │
                            │  LMStudio :1234  │
                            │  qwen3-coder-30b-a3b-instruct-mlx     │
                            └──────────────────┘
```

### Потоки данных

**Метрики:**
```
K8s targets → VMAgent (scrape) → VMSingle (store) → VMAlert (evaluate rules)
                                      ↓                      ↓
                                  Grafana (UI)         VMAlertmanager
                                                            ↓
                                                    alertmanager-bridge
                                                            ↓
                                                    HolmesGPT /api/investigate
```

**Логи:**
```
Контейнеры → /var/log/pods/ → vlagent collector (DaemonSet) → VictoriaLogs (store)
```

**Расследование инцидента:**
```
1. Alert firing → VMAlertmanager → webhook bridge → HolmesGPT
2. HolmesGPT подбирает runbook по regex-паттерну title
3. HolmesGPT выполняет kubectl, PromQL-запросы к VictoriaMetrics
4. LLM анализирует данные → возвращает analysis + рекомендации
```

---

## Файловая структура

```
idp-app-v1/
├── applications/
│   ├── apps.yaml                   # App-of-apps (syncs all files in this dir)
│   ├── holmesgpt.yaml              # Holmes v0.19.0 + prometheus toolset
│   ├── vm-k8s-stack.yaml           # VictoriaMetrics + Grafana
│   ├── victoria-logs.yaml          # VictoriaLogs server
│   └── victoria-logs-collector.yaml # vlagent DaemonSet
│
├── platform/holmesgpt/
│   ├── custom-runbooks-configmap.yaml  # 3 custom runbooks (tailscale, pods, llm)
│   ├── sre-runbooks.yaml               # 8 SRE runbooks (OOM, crash, CPU, etc.)
│   ├── alertmanager-webhook.yaml        # Bridge deployment + service
│   ├── vmalertmanagerconfig.yaml        # Alert routing → bridge
│   ├── victorialogs-toolset.yaml        # Custom toolset (не используется chart'ом)
│   └── knowledge-base-toolset.yaml      # Custom toolset (не используется chart'ом)
│
└── docs/
    └── SRE-INTEGRATION-REPORT.md        # Этот файл
```

---

## Установка

### Предварительные требования

- Kubernetes кластер с ArgoCD
- Storage class `openebs-hostpath`
- LLM endpoint (LMStudio/vLLM через llm-proxy)
- app-of-apps (`applications/apps.yaml`) уже установлен

### Порядок установки

ArgoCD applications синхронизируются автоматически через app-of-apps. Ручные шаги:

```bash
export KUBECONFIG=~/proj/cross/kubeconfig_6005021

# 1. ConfigMaps (не управляются ArgoCD — holmesgpt App указывает на Helm chart)
kubectl apply -f platform/holmesgpt/custom-runbooks-configmap.yaml
kubectl apply -f platform/holmesgpt/sre-runbooks.yaml

# 2. Alertmanager bridge
kubectl apply -f platform/holmesgpt/alertmanager-webhook.yaml
kubectl apply -f platform/holmesgpt/vmalertmanagerconfig.yaml

# 3. Проверить статус
kubectl get applications -n argocd
kubectl get pods -n monitoring
kubectl get pods -n victorialogs
kubectl get pods -n holmesgpt
```

**kubectl patch больше не нужен** — chart `holmes` 0.19.0 поддерживает `additionalVolumes`/`additionalVolumeMounts` через Helm values.

---

## Проверка

### ArgoCD

```bash
kubectl get applications -n argocd -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'
# Все приложения должны быть Synced/Healthy
```

### VictoriaMetrics

```bash
# Запрос метрик (количество up targets)
kubectl run test-vm --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s 'http://vmsingle-vm-k8s-stack-victoria-metrics-k8s-stack.monitoring.svc:8429/api/v1/query?query=up'
```

### VictoriaLogs

```bash
# Health check
kubectl run test-vl --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s 'http://victoria-logs-victoria-logs-single-server.victorialogs.svc:9428/health'

# Запрос логов (последние 5 минут)
kubectl run test-vlq --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s 'http://victoria-logs-victoria-logs-single-server.victorialogs.svc:9428/select/logsql/query?query=*&limit=5'
```

### HolmesGPT

```bash
# Проверить модель
kubectl run test-model --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s http://holmesgpt-holmes.holmesgpt.svc:80/api/model

# Тестовый чат (PromQL-запрос через toolset)
kubectl run test-chat --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 300 -X POST http://holmesgpt-holmes.holmesgpt.svc:80/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"ask": "What is the CPU usage in monitoring namespace? Use prometheus metrics."}'

# Проверить runbooks
kubectl exec -n holmesgpt deploy/holmesgpt-holmes -- ls /app/holmes/plugins/runbooks/
# Ожидаемо: custom-runbooks.yaml  jira.yaml  kube-prometheus-stack.yaml  sre-runbooks.yaml
```

### Alertmanager Bridge

```bash
# Health check
kubectl run test-bridge --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s http://holmes-alertmanager-bridge.holmesgpt.svc:9095/health

# Тестовый webhook (имитация alert)
kubectl run test-webhook --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s -X POST http://holmes-alertmanager-bridge.holmesgpt.svc:9095/webhook \
  -H 'Content-Type: application/json' \
  -d '{"alerts":[{"status":"firing","labels":{"alertname":"TestAlert","severity":"warning","namespace":"default"},"annotations":{"description":"Test alert for verification"}}]}'
```

---

## Исправленные ошибки

При первоначальном деплое было 4 критические ошибки. Подробный отчёт: `~/proj/cross/report-sre-stack-fixes.md`.

| Проблема | Причина | Исправление | Коммит |
|----------|---------|-------------|--------|
| Grafana CrashLoopBackOff | Plugin ID `victoriametrics-datasource` не существует | Заменить на `victoriametrics-metrics-datasource` | `af1417e` |
| HolmesGPT InvalidImageName | `image:` в chart — строка, передан объект | Убрать блок `image:` из values | `9fbf816` |
| victoria-logs-collector не деплоится | Chart version `0.16.0` не существует | Заменить на `0.2.9` | `9fbf816` |
| ArgoCD Unknown diff error | VMAlertmanager CRD: `storage.resources` не в схеме | Использовать `volumeClaimTemplate` | `713a996` |

### Важные нюансы

1. **Chart `holmes` 0.19.0: `image:` — строка, не объект.** Формат: `image: holmes:0.19.0`, `registry: robustadev`. Нельзя передавать `image: {repository:..., tag:...}`.

2. **VMAlertmanager vs VMSingle CRD:** VMSingle поддерживает прямой PVC spec (`storageClassName`, `resources`). VMAlertmanager требует `volumeClaimTemplate.spec.storageClassName`.

3. **app-of-apps перезаписывает kubectl apply.** `applications/apps.yaml` с `selfHeal: true` откатывает ручные изменения. Все правки в Application ресурсах — только через git push.

4. **victoria-logs-collector — не Vector.** Это нативный `vlagent` от VictoriaMetrics. Версии начинаются с `0.0.x`/`0.2.x`.

5. **Grafana plugin ID:** `victoriametrics-metrics-datasource` (не `victoriametrics-datasource`).

---

## Коммиты

```
3d9b539 feat: Add SRE integration with VictoriaMetrics, VictoriaLogs and HolmesGPT Agent Mode
9fbf816 fix: resolve deployment issues in SRE stack
f2e4ac8 fix(vm-k8s-stack): add ignoreDifferences for CRD storage fields
713a996 fix(vm-k8s-stack): use volumeClaimTemplate for alertmanager storage
af1417e fix(grafana): correct plugin ID to victoriametrics-metrics-datasource
```
