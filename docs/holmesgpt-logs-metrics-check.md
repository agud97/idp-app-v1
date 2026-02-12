# Проверка анализа логов и метрик в HolmesGPT

## Предусловия

```bash
export KUBECONFIG=~/proj/cross/kubeconfig_6005021
```

Включённые toolsets (holmesgpt.yaml):
- `kubernetes/core` — kubectl-команды (get, describe, events)
- `kubernetes/logs` — чтение логов подов
- `prometheus/metrics` — PromQL-запросы к VictoriaMetrics
- `runbook` — автоматический подбор runbooks при investigate
- `bash` — выполнение bash-команд

Prometheus URL: `http://vmsingle-vm-k8s-stack-victoria-metrics-k8s-stack.monitoring.svc:8429`

---

## Быстрая проверка (3 команды)

Скопируй и запусти по одной. Каждая занимает 1-3 минуты.

### 1. Логи

```bash
kubectl run test-logs --rm -i --restart=Never --image=badouralix/curl-jq -- \
  sh -c 'curl -s --max-time 300 -X POST http://holmesgpt-holmes.holmesgpt.svc:80/api/chat \
  -H "Content-Type: application/json" \
  -d "{\"ask\": \"Check logs of holmesgpt-holmes deployment in holmesgpt namespace. Are there any errors or warnings?\"}" \
  | jq -r ".analysis"'
```

**Ожидаемый результат:** Текст с цитатами из реальных логов пода. Например:
> Found one warning: `No content found in file: /etc/holmes/config/model_list.yaml`

### 2. Метрики

```bash
kubectl run test-metrics --rm -i --restart=Never --image=badouralix/curl-jq -- \
  sh -c 'curl -s --max-time 300 -X POST http://holmesgpt-holmes.holmesgpt.svc:80/api/chat \
  -H "Content-Type: application/json" \
  -d "{\"ask\": \"Query Prometheus for current CPU usage across all nodes and top 5 pods by memory usage\"}" \
  | jq -r ".analysis"'
```

**Ожидаемый результат:** Реальные числа по CPU и памяти. Например:
> Node 192.168.1.109: ~6.8% CPU
> Top pod: vmsingle — 2083 MB memory

### 3. Логи + метрики вместе

```bash
kubectl run test-combined --rm -i --restart=Never --image=badouralix/curl-jq -- \
  sh -c 'curl -s --max-time 300 -X POST http://holmesgpt-holmes.holmesgpt.svc:80/api/chat \
  -H "Content-Type: application/json" \
  -d "{\"ask\": \"Investigate open-webui namespace: 1) check pod logs for errors 2) query prometheus for memory usage of pods in open-webui namespace\"}" \
  | jq -r ".analysis"'
```

**Ожидаемый результат:** Отчёт с данными из обоих источников — строки логов + числа по памяти.

---

## Результаты проверки 2026-02-12

| Проверка | Статус | Tool calls | Результат |
|---|---|---|---|
| Логи | OK | 4 | Прочитал логи holmesgpt, нашёл warning `model_list.yaml` |
| Метрики | OK | 8 | CPU по нодам (4-7%), top по памяти (VictoriaMetrics ~2GB) |
| Комбинированный | OK | ~10 | Статус подов open-webui, логи без ошибок, память: open-webui ~628MB, pipelines ~62MB |

---

## Расширенные проверки

### Проверка расследования с runbooks (/api/investigate)

В отличие от `/api/chat`, эндпоинт `/api/investigate` автоматически подтягивает runbooks по regex-паттерну из title.

#### Расследование с runbook (title матчит паттерн)

```bash
kubectl run test-investigate --rm -i --restart=Never --image=badouralix/curl-jq -- \
  sh -c 'curl -s --max-time 300 -X POST http://holmesgpt-holmes.holmesgpt.svc:80/api/investigate \
  -H "Content-Type: application/json" \
  -d "{
    \"source\": \"manual\",
    \"source_instance_id\": \"test\",
    \"title\": \"HighMemory usage on cluster nodes\",
    \"description\": \"Memory usage exceeded 80% on cluster nodes\",
    \"subject\": {},
    \"context\": {}
  }" | jq "{analysis: .analysis, instructions_count: (.instructions | length), tool_calls_count: (.tool_calls | length)}"'
```

**Ожидаемый результат:**
- `instructions_count` > 0 (runbook `.*HighMemory.*` подтянулся)
- `tool_calls_count` > 0 (kubectl + PromQL запросы)
- `analysis` — реальные метрики по памяти

#### Расследование без runbook (проверка что матчинг корректен)

```bash
kubectl run test-no-runbook --rm -i --restart=Never --image=badouralix/curl-jq -- \
  sh -c 'curl -s --max-time 300 -X POST http://holmesgpt-holmes.holmesgpt.svc:80/api/investigate \
  -H "Content-Type: application/json" \
  -d "{
    \"source\": \"manual\",
    \"source_instance_id\": \"test\",
    \"title\": \"random unrelated issue\",
    \"description\": \"Test\",
    \"subject\": {},
    \"context\": {}
  }" | jq "{instructions_count: (.instructions | length)}"'
```

**Ожидаемый результат:** `instructions_count: 0` — ни один runbook не сматчился.

### Проверка анализа логов конкретного namespace

```bash
kubectl run test-logs-ns --rm -i --restart=Never --image=badouralix/curl-jq -- \
  sh -c 'curl -s --max-time 300 -X POST http://holmesgpt-holmes.holmesgpt.svc:80/api/chat \
  -H "Content-Type: application/json" \
  -d "{\"ask\": \"Get logs from all pods in monitoring namespace and check for errors\"}" \
  | jq -r ".analysis"'
```

### Проверка метрик: healthy/down targets

```bash
kubectl run test-targets --rm -i --restart=Never --image=badouralix/curl-jq -- \
  sh -c 'curl -s --max-time 300 -X POST http://holmesgpt-holmes.holmesgpt.svc:80/api/chat \
  -H "Content-Type: application/json" \
  -d "{\"ask\": \"Query prometheus for up metric to check how many targets are healthy vs down\"}" \
  | jq -r ".analysis"'
```

### Просмотр tool_calls (какие инструменты использовались)

Добавь к любой команде вместо `jq -r ".analysis"`:

```
jq "{analysis: .analysis, tool_calls: [.tool_calls[]? | {function: .function_name, args: .arguments}]}"
```

Полный пример:

```bash
kubectl run test-toolcalls --rm -i --restart=Never --image=badouralix/curl-jq -- \
  sh -c 'curl -s --max-time 300 -X POST http://holmesgpt-holmes.holmesgpt.svc:80/api/chat \
  -H "Content-Type: application/json" \
  -d "{\"ask\": \"Show me the last 10 log lines from llm-proxy deployment in llm-proxy namespace\"}" \
  | jq "{analysis: .analysis, tool_calls: [.tool_calls[]? | {function: .function_name, args: .arguments}]}"'
```

---

## Диагностика: если не работает

### Логи не анализируются

```bash
# 1. RBAC — есть ли доступ к логам?
kubectl auth can-i get pods/log \
  --as=system:serviceaccount:holmesgpt:holmesgpt-holmes -n monitoring

kubectl auth can-i get pods/log \
  --as=system:serviceaccount:holmesgpt:holmesgpt-holmes -n open-webui

# Ожидаемый результат: yes

# 2. Проверить логи самого HolmesGPT
kubectl logs -n holmesgpt deploy/holmesgpt-holmes --tail=50

# 3. Проверить что toolsets в Application корректны
kubectl get application -n argocd holmesgpt \
  -o jsonpath='{.spec.source.helm.values}' | grep -A2 "kubernetes/logs"

# Ожидаемый результат:
#   kubernetes/logs:
#     enabled: true
```

### Метрики не анализируются

```bash
# 1. Проверить что VictoriaMetrics доступна
kubectl run test-vm --rm -i --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 5 \
  "http://vmsingle-vm-k8s-stack-victoria-metrics-k8s-stack.monitoring.svc:8429/api/v1/query?query=up"

# Ожидаемый результат: JSON с данными метрик (status: "success")

# 2. Проверить конфигурацию prometheus_url
kubectl get application -n argocd holmesgpt \
  -o jsonpath='{.spec.source.helm.values}' | grep -A2 prometheus

# Ожидаемый результат:
#   prometheus_url: "http://vmsingle-vm-k8s-stack-victoria-metrics-k8s-stack.monitoring.svc:8429"

# 3. Проверить что HolmesGPT pod работает
kubectl get pods -n holmesgpt
kubectl logs -n holmesgpt deploy/holmesgpt-holmes --tail=30
```

### LLM не отвечает

```bash
# Проверить что llm-proxy и tailscale работают
kubectl exec -n llm-proxy deploy/llm-proxy -c tailscale-sidecar -- tailscale status
kubectl exec -n llm-proxy deploy/llm-proxy -c nginx -- wget -q -O- --timeout=10 http://127.0.0.1:8080/v1/models

# Проверить текущую модель
kubectl run test-model --rm -i --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 10 http://holmesgpt-holmes.holmesgpt.svc:80/api/model
```

---

## Справка по API

| Эндпоинт | Назначение | Runbooks | Toolsets |
|---|---|---|---|
| `POST /api/chat` | Свободный чат | Нет | Все включённые |
| `POST /api/investigate` | Расследование инцидента | Да (regex по title) | Все включённые |
| `GET /api/model` | Текущая модель | — | — |

### Формат запроса /api/chat

```json
{"ask": "Ваш вопрос"}
```

### Формат запроса /api/investigate

```json
{
  "source": "manual",
  "source_instance_id": "test",
  "title": "Заголовок инцидента (матчится с runbooks)",
  "description": "Описание проблемы",
  "subject": {"namespace": "имя-namespace"},
  "context": {}
}
```

### Формат ответа

```json
{
  "analysis": "Текстовый анализ от LLM",
  "instructions": ["Инструкции из сматченных runbooks"],
  "tool_calls": [{"function_name": "...", "arguments": "..."}]
}
```

---

## Примечания

- Каждый запрос занимает 1-3 минуты (LLM думает + выполняет kubectl/PromQL)
- `--max-time 300` — таймаут 5 минут, можно увеличить для сложных запросов
- Образ `badouralix/curl-jq` содержит curl + jq для форматирования ответа
- Образ `curlimages/curl` — только curl, без jq
- Если `kubectl run` зависает — EOF-ошибка кластера, повторить
- Имена тест-подов (`test-logs`, `test-metrics`) должны быть уникальными; если pod уже существует — удалить: `kubectl delete pod test-logs --force`
