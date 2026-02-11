# Отчет: SRE-интеграция для Kubernetes

**Дата:** 2025-02-11
**Репозиторий:** https://github.com/agud97/idp-app-v1

---

## Цель

Создать решение для автоматического расследования инцидентов в Kubernetes-кластере с использованием локальной LLM (без публичных моделей).

## Проблематика

| Проблема | Решение |
|----------|---------|
| Нет интеграции с метриками | VictoriaMetrics (PromQL-совместимый) |
| Нет интеграции с логами | VictoriaLogs (LogsQL) |
| HolmesGPT v0.6.0 устарел | Обновление до v0.19.0 с toolsets |
| Нет базы знаний (RAG) | Knowledge Base toolset для Open WebUI |
| Нет подтверждения действий | Agent Mode с confirmation |

---

## Что было сделано

### 1. Добавлена VictoriaMetrics (метрики)

**Файл:** `applications/vm-k8s-stack.yaml`

Компоненты:
- **VMSingle** — хранилище метрик (15d retention, 50Gi storage)
- **VMAgent** — сбор метрик с Kubernetes
- **VMAlert** — выполнение alerting rules
- **VMAlertmanager** — маршрутизация алертов
- **Grafana** — визуализация с плагинами VictoriaMetrics

Endpoint: `http://vm-k8s-stack-victoria-metrics-k8s-stack-vmsingle.monitoring.svc:8429`

### 2. Добавлена VictoriaLogs (логи)

**Файлы:**
- `applications/victoria-logs.yaml` — сервер хранения
- `applications/victoria-logs-collector.yaml` — сборщик логов

Параметры:
- Retention: 30d
- Storage: 50Gi
- Endpoint: `http://vls-victoria-logs-single-server.victorialogs.svc:9428`

### 3. Обновлен HolmesGPT

**Файл:** `applications/holmesgpt.yaml`

Изменения:
- Версия: 0.6.0 → 0.19.0
- Добавлены toolsets: `kubernetes/core`, `kubernetes/logs`, `prometheus/metrics`, `victorialogs/logs`, `knowledge/base`
- Включен Agent Mode (`INTERACTIVE_MODE=true`, `REQUIRE_CONFIRMATION=true`)
- Расширенные RBAC права

### 4. Созданы custom toolsets

**VictoriaLogs Toolset:** `platform/holmesgpt/victorialogs-toolset.yaml`

Инструменты:
- `query_logs` — запрос логов по namespace/pod
- `query_error_logs` — поиск ошибок
- `list_log_streams` — список потоков
- `search_logs` — поиск по тексту
- `logs_stats` — статистика логов

**Knowledge Base Toolset:** `platform/holmesgpt/knowledge-base-toolset.yaml`

Инструменты:
- `search_runbooks` — поиск runbooks в RAG
- `search_incident_history` — поиск post-mortems
- `search_knowledge` — универсальный поиск
- `add_note` — добавление заметок

### 5. Добавлены SRE Runbooks

**Файл:** `platform/holmesgpt/sre-runbooks.yaml`

Сценарии:
- Общий workflow для всех инцидентов
- OOMKilled / OutOfMemory
- CrashLoopBackOff
- High CPU / CPU Throttling
- HTTP 5xx ошибки
- Disk / Storage issues
- Node NotReady
- Tailscale/LLM-Proxy connectivity

### 6. Настроен Alertmanager Bridge

**Файл:** `platform/holmesgpt/alertmanager-webhook.yaml`

Python-сервис, который:
- Принимает webhook от Alertmanager
- Вызывает `/api/investigate` HolmesGPT
- Логирует результаты расследования

**Файл:** `platform/holmesgpt/vmalertmanagerconfig.yaml`

Конфигурация маршрутизации алертов с severity `critical|warning` в HolmesGPT.

---

## Архитектура

```
┌─────────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────────┐   │
│  │VictoriaMetrics│ │VictoriaLogs │   │ Open WebUI + Qdrant │   │
│  │  (metrics)   │   │  (logs)     │   │  (knowledge base)   │   │
│  │   :8429      │   │   :9428     │   │        :8080        │   │
│  └──────┬──────┘   └──────┬──────┘   └───────────┬─────────┘   │
│         │                 │                      │              │
│         │    Toolsets     │        Toolset       │              │
│         └─────────────────┼──────────────────────┘              │
│                           │                                     │
│                           ▼                                     │
│                  ┌─────────────────┐                            │
│                  │   HolmesGPT     │◄── Alertmanager Webhook    │
│                  │   Agent Mode    │                            │
│                  └────────┬────────┘                            │
│                           │                                     │
│                           ▼                                     │
│                  ┌─────────────────┐                            │
│                  │  Kubernetes API │                            │
│                  └─────────────────┘                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                      ┌───────┴───────┐
                      │  GPU Server   │
                      │  (NVIDIA A5000)│
                      │  vLLM / Qwen  │
                      └───────────────┘
```

---

## Workflow Agent Mode

```
1. ALERT → HolmesGPT получает инцидент
       │
       ▼
2. RAG SEARCH → Поиск в базе знаний
   - search_runbooks()
   - search_incident_history()
       │
       ▼
3. GATHER DATA → Сбор live данных
   - Метрики (VictoriaMetrics)
   - Логи (VictoriaLogs)
   - K8s state (kubectl)
       │
       ▼
4. PROPOSE ACTION → Предложение с подтверждением
   "Выполнить kubectl patch ...?"
       │
       ▼
5. CONFIRM → Пользователь подтверждает
       │
       ▼
6. EXECUTE → Команда выполняется
```

---

## Файловая структура

```
idp-app-v1/
├── applications/
│   ├── holmesgpt.yaml              # Обновлено v0.19.0
│   ├── vm-k8s-stack.yaml           # NEW
│   ├── victoria-logs.yaml          # NEW
│   └── victoria-logs-collector.yaml # NEW
│
└── platform/holmesgpt/
    ├── victorialogs-toolset.yaml   # NEW
    ├── knowledge-base-toolset.yaml # NEW
    ├── sre-runbooks.yaml           # NEW
    ├── alertmanager-webhook.yaml   # NEW
    └── vmalertmanagerconfig.yaml   # NEW
```

---

## Установка

### Предварительные требования

- Kubernetes кластер с ArgoCD
- Storage class `openebs-hostpath`
- LLM endpoint (vLLM/LMStudio)

### Шаги

```bash
# 1. Установить CRD
helm repo add vm https://victoriametrics.github.io/helm-charts/
helm show crds vm/victoria-metrics-k8s-stack --version 0.45.0 | kubectl apply -f -

# 2. ArgoCD синхронизирует Applications автоматически

# 3. Применить ConfigMaps
kubectl apply -f platform/holmesgpt/victorialogs-toolset.yaml
kubectl apply -f platform/holmesgpt/knowledge-base-toolset.yaml
kubectl apply -f platform/holmesgpt/sre-runbooks.yaml
kubectl apply -f platform/holmesgpt/alertmanager-webhook.yaml
kubectl apply -f platform/holmesgpt/vmalertmanagerconfig.yaml

# 4. Patch deployment
kubectl patch deployment holmesgpt-holmes -n holmesgpt --type=json -p='[
  {"op":"add","path":"/spec/template/spec/volumes/-","value":{"name":"victorialogs-toolset","configMap":{"name":"victorialogs-toolset"}}},
  {"op":"add","path":"/spec/template/spec/volumes/-","value":{"name":"knowledge-base-toolset","configMap":{"name":"knowledge-base-toolset"}}},
  {"op":"add","path":"/spec/template/spec/volumes/-","value":{"name":"sre-runbooks","configMap":{"name":"sre-runbooks"}}}
]'

kubectl patch deployment holmesgpt-holmes -n holmesgpt --type=json -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts/-","value":{"name":"victorialogs-toolset","mountPath":"/app/holmes/plugins/toolsets/victorialogs-toolset.yaml","subPath":"victorialogs-toolset.yaml"}},
  {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts/-","value":{"name":"knowledge-base-toolset","mountPath":"/app/holmes/plugins/toolsets/knowledge-base-toolset.yaml","subPath":"knowledge-base-toolset.yaml"}},
  {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts/-","value":{"name":"sre-runbooks","mountPath":"/app/holmes/plugins/runbooks/sre-runbooks.yaml","subPath":"sre-runbooks.yaml"}}
]'
```

---

## Проверка

```bash
# VictoriaMetrics
kubectl run test-vm --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s 'http://vm-k8s-stack-victoria-metrics-k8s-stack-vmsingle.monitoring.svc:8429/api/v1/query?query=up'

# VictoriaLogs
kubectl run test-vl --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s 'http://vls-victoria-logs-single-server.victorialogs.svc:9428/health'

# HolmesGPT
kubectl port-forward -n holmesgpt svc/holmesgpt-holmes 8088:80
curl -X POST http://localhost:8088/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"ask": "Check CPU usage in monitoring namespace"}'
```

---

## Рекомендации

1. **Загрузить документы в Open WebUI** — runbooks, post-mortems, архитектуру
2. **Настроить LLM endpoint** — обновить `OPENAI_API_BASE` в holmesgpt.yaml
3. **Создать API ключ Open WebUI** — для Knowledge Base toolset
4. **Настроить alerting rules** — добавить VMAlert rules для критичных метрик

---

## Коммит

```
c3d59a8 feat: Add SRE integration with VictoriaMetrics, VictoriaLogs and HolmesGPT Agent Mode
```
