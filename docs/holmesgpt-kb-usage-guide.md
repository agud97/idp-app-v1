# HolmesGPT + kb-stack: Инструкция по использованию

Этот документ описывает как задавать вопросы HolmesGPT с использованием данных из базы знаний кластера: Kubescape (security scan), Popeye (lint), Kubevious (topology).

---

## Архитектура взаимодействия

```
Пользователь / Robusta
      │
      │  POST /api/chat  {"ask": "..."}
      ▼
┌──────────────────────┐
│   HolmesGPT (LLM)    │   ← отвечает на вопросы, сам решает какие инструменты вызвать
│                      │
│  Toolset kb/stack:   │
│  ┌────────────────┐  │
│  │ kb_search      │──┼──▶ embedding-svc → Qdrant (semantic search)
│  │ kb_list_       │  │
│  │  findings      │──┼──▶ Qdrant (scroll/filter by doc_type)
│  │ kb_fetch_      │  │
│  │  artifact      │──┼──▶ MinIO (raw JSON файл по пути)
│  └────────────────┘  │
└──────────────────────┘
```

**Источники данных в базе знаний:**

| `doc_type` | Источник | Что содержит |
|---|---|---|
| `finding_kubescape` | Kubescape NSA scan | Security findings: failed controls, кол-во ресурсов |
| `finding_popeye` | Popeye lint | Ошибки и предупреждения по ресурсам кластера |
| `topology` | Kubevious | Дерево ресурсов кластера, алерты, свойства нод |

---

## Способ 1: HTTP API `/api/chat`

Основной способ работы с HolmesGPT — отправить вопрос через HTTP.
HolmesGPT сам решит, какие инструменты вызвать, и вернёт итоговый ответ.

### Запрос из kubectl (из пода HolmesGPT)

> **Порт**: HolmesGPT слушает на `localhost:5050` внутри пода (env `HOLMES_PORT=5050`).
> Service экспонирует его как 80→5050. Из пода всегда используй `localhost:5050`.
>
> **Таймаут**: LLM (Qwen3-30B через LMStudio) обрабатывает каждый шаг за 5–30 сек,
> а запрос с несколькими tool calls может занять 3–8 минут. Используй `timeout=600`.

```bash
export K="kubectl --kubeconfig ~/proj/cross/kubeconfig_6005021"

# Получить имя пода HolmesGPT
HOLMES_POD=$($K get pods -n holmesgpt --no-headers | grep "holmesgpt-holmes.*Running" | awk '{print $1}')

# Задать вопрос через API
$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import urllib.request, json

API = 'http://localhost:5050'
question = 'Какие security controls провалил Kubescape? Покажи топ-5 по количеству failed ресурсов.'

req = urllib.request.Request(
    f'{API}/api/chat',
    data=json.dumps({'ask': question}).encode(),
    headers={'Content-Type': 'application/json'},
    method='POST'
)
with urllib.request.urlopen(req, timeout=600) as r:
    resp = json.loads(r.read())
    print(resp['analysis'])
"
```

### Минимальная структура запроса

```json
{
  "ask": "Ваш вопрос"
}
```

### Расширенный запрос (с историей диалога)

```json
{
  "ask": "Покажи детали по контролю Ingress and Egress blocked",
  "conversation_history": [
    {
      "role": "assistant",
      "content": "Kubescape нашёл следующие failures: ..."
    }
  ]
}
```

---

## Способ 2: Прямой вызов инструментов (отладка и скрипты)

Инструменты `kb_tools.py` можно вызывать напрямую из пода HolmesGPT
для отладки, написания скриптов или быстрой проверки данных.

```bash
HOLMES_POD=$($K get pod -n holmesgpt -l app.kubernetes.io/name=holmes \
  -o jsonpath='{.items[0].metadata.name}')

# Синтаксис:
# python3 /kb-scripts/kb_tools.py search "<запрос>" [top_k] [cluster_id]
# python3 /kb-scripts/kb_tools.py list   [doc_type] [limit] [cluster_id]
```

---

## Источник 1: Kubescape (Security Scan)

**doc_type:** `finding_kubescape`
**Что хранится:** результаты NSA benchmark scan — какие security controls не прошли проверку и сколько ресурсов затронуто.

### Примеры вопросов к HolmesGPT

#### Обзор security posture

```bash
$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import urllib.request, json
req = urllib.request.Request(
    'http://localhost:5050/api/chat',
    data=json.dumps({
        'ask': 'Проанализируй результаты Kubescape security scan. Какие controls провалились? '
               'Сколько ресурсов затронуто? Расставь по приоритету.'
    }).encode(),
    headers={'Content-Type': 'application/json'}, method='POST'
)
with urllib.request.urlopen(req, timeout=600) as r:
    print(json.loads(r.read())['analysis'])
"
```

#### Конкретная проблема

```bash
$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import urllib.request, json
req = urllib.request.Request(
    'http://localhost:5050/api/chat',
    data=json.dumps({
        'ask': 'В кластере не выставлены resource limits на контейнеры. '
               'Найди в kubescape какие именно namespaces и поды затронуты.'
    }).encode(),
    headers={'Content-Type': 'application/json'}, method='POST'
)
with urllib.request.urlopen(req, timeout=600) as r:
    print(json.loads(r.read())['analysis'])
"
```

#### Прямой поиск (без LLM)

```bash
# Semantic search по kubescape данным
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py search \
  "privilege escalation containers root" 5 mvp-cloud

# Список всех kubescape документов
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py list finding_kubescape 10 mvp-cloud
```

#### Получить сырой JSON отчёт Kubescape

```bash
# 1. Найти path через kb_list
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py list finding_kubescape 1 mvp-cloud

# 2. Скачать полный JSON из MinIO (boto3, AWS4 — curl -u НЕ работает с MinIO S3)
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py fetch \
  "raw/kubescape/mvp-cloud/<ts>/findings.json" | head -100

# Или попросить HolmesGPT получить файл через kb_fetch_artifact:
$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import urllib.request, json
req = urllib.request.Request(
    'http://localhost:5050/api/chat',
    data=json.dumps({
        'ask': 'Используй kb_list_findings для поиска kubescape отчёта, '
               'затем kb_fetch_artifact чтобы получить полный JSON '
               'и расскажи что в нём написано про non-root containers.'
    }).encode(),
    headers={'Content-Type': 'application/json'}, method='POST'
)
with urllib.request.urlopen(req, timeout=180) as r:
    print(json.loads(r.read())['analysis'])
"
```

### Структура данных Kubescape в Qdrant

```
embed_text (индексируется для поиска):
  "kubescape security findings: Non-root containers(54 failed),
   Allow privilege escalation(45 failed), Immutable container filesystem(48 failed), ..."

Полный JSON (в MinIO raw/kubescape/...):
  {
    "summaryDetails": {
      "controls": {
        "C-0013": {
          "name": "Non-root containers",
          "status": "failed",
          "ResourceCounters": {"failedResources": 54, "passedResources": 0}
        },
        ...
      }
    },
    "results": [...]  ← per-resource details
  }
```

### Эффективные поисковые запросы для Kubescape

```bash
# Привилегии
python3 /kb-scripts/kb_tools.py search "privilege escalation root containers"     5 mvp-cloud
python3 /kb-scripts/kb_tools.py search "service account permissions RBAC admin"  5 mvp-cloud
python3 /kb-scripts/kb_tools.py search "host PID IPC network namespace"          5 mvp-cloud

# Сеть
python3 /kb-scripts/kb_tools.py search "ingress egress network policy blocked"   5 mvp-cloud
python3 /kb-scripts/kb_tools.py search "exposed services ports external"         5 mvp-cloud

# Конфигурация
python3 /kb-scripts/kb_tools.py search "secrets credentials configuration files" 5 mvp-cloud
python3 /kb-scripts/kb_tools.py search "resource limits CPU memory not set"      5 mvp-cloud
python3 /kb-scripts/kb_tools.py search "immutable filesystem read only"          5 mvp-cloud
```

---

## Источник 2: Popeye (Kubernetes Linter)

**doc_type:** `finding_popeye`
**Что хранится:** результаты lint проверки ресурсов кластера — ошибки и предупреждения по namespace, deployment, service, pod и т.д.

### Примеры вопросов к HolmesGPT

#### Обзор проблем кластера

```bash
$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import urllib.request, json
req = urllib.request.Request(
    'http://localhost:5050/api/chat',
    data=json.dumps({
        'ask': 'Получи последний Popeye lint отчёт из базы знаний. '
               'Какие ресурсы имеют ошибки? Что нужно починить в первую очередь?'
    }).encode(),
    headers={'Content-Type': 'application/json'}, method='POST'
)
with urllib.request.urlopen(req, timeout=180) as r:
    print(json.loads(r.read())['analysis'])
"
```

#### Конкретный тип ресурса

```bash
$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import urllib.request, json
req = urllib.request.Request(
    'http://localhost:5050/api/chat',
    data=json.dumps({
        'ask': 'Используй kb_list_findings с doc_type=finding_popeye чтобы найти popeye отчёт, '
               'затем kb_fetch_artifact чтобы получить полный JSON. '
               'Выведи все sanitizers с error > 0 и объясни причины.'
    }).encode(),
    headers={'Content-Type': 'application/json'}, method='POST'
)
with urllib.request.urlopen(req, timeout=180) as r:
    print(json.loads(r.read())['analysis'])
"
```

#### Прямой поиск (без LLM)

```bash
# Список popeye документов
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py list finding_popeye 5 mvp-cloud

# Получить сырой отчёт и разобрать (boto3, AWS4 — curl -u НЕ работает с MinIO S3)
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py fetch \
  "raw/popeye/mvp-cloud/<ts>/report.json" | python3 -c "
import sys, json
d = json.load(sys.stdin)
sanitizers = d.get('popeye', {}).get('sanitizers', [])
for s in sanitizers:
    t = s.get('tally', {})
    if t.get('error', 0) or t.get('warning', 0):
        print(f\"{s['sanitizer']:30} errors={t.get('error',0):3}  warnings={t.get('warning',0):3}\")
"
```

### Структура данных Popeye в Qdrant

```
embed_text:
  "popeye lint report: namespaces errors=0 warnings=5
   deployments errors=2 warnings=18 services errors=0 warnings=3 ..."

Полный JSON (в MinIO raw/popeye/...):
  {
    "popeye": {
      "sanitizers": [
        {
          "sanitizer": "namespaces",
          "tally": {"error": 0, "warning": 5, "ok": 3, "info": 0, "score": 100}
        },
        {
          "sanitizer": "deployments",
          "tally": {"error": 2, "warning": 18, ...}
          "issues": {
            "kb-system/minio": [
              {"group": "__root__", "gvr": "apps/v1/deployments",
               "level": 2, "message": "..."}
            ]
          }
        }
      ]
    }
  }

Уровни severity в popeye:
  0 = OK, 1 = Info, 2 = Warning, 3 = Error
```

### Эффективные поисковые запросы для Popeye

> **Примечание:** Semantic search по Popeye работает лучше через `kb_list_findings` + `kb_fetch_artifact` (анализ полного JSON), так как в embed_text хранится краткая сводка. Для детального анализа всегда запрашивай полный файл.

```bash
# Общий обзор
python3 /kb-scripts/kb_tools.py search "popeye errors warnings resources"  5 mvp-cloud
python3 /kb-scripts/kb_tools.py search "lint issues kubernetes resources"   5 mvp-cloud

# После получения path — полный анализ (boto3, AWS4 — curl -u НЕ работает)
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py fetch \
  "raw/popeye/mvp-cloud/<ts>/report.json" | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(json.dumps(d['popeye']['sanitizers'], indent=2))
"
```

---

## Источник 3: Kubevious (Topology)

**doc_type:** `topology`
**Что хранится:** снимок дерева ресурсов кластера из Kubevious — все ноды с их свойствами и алертами.

### Примеры вопросов к HolmesGPT

#### Обзор топологии кластера

```bash
$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import urllib.request, json
req = urllib.request.Request(
    'http://localhost:5050/api/chat',
    data=json.dumps({
        'ask': 'Используй kb-stack инструменты чтобы получить topology данные. '
               'Какие namespaces существуют в кластере? '
               'Есть ли алерты на каких-нибудь ресурсах?'
    }).encode(),
    headers={'Content-Type': 'application/json'}, method='POST'
)
with urllib.request.urlopen(req, timeout=180) as r:
    print(json.loads(r.read())['analysis'])
"
```

#### Поиск конкретного workload в топологии

```bash
$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import urllib.request, json
req = urllib.request.Request(
    'http://localhost:5050/api/chat',
    data=json.dumps({
        'ask': 'Найди в topology данных всё что связано с namespace holmesgpt. '
               'Какие deployments там есть? Какие у них свойства (props)?'
    }).encode(),
    headers={'Content-Type': 'application/json'}, method='POST'
)
with urllib.request.urlopen(req, timeout=180) as r:
    print(json.loads(r.read())['analysis'])
"
```

#### Прямой разбор topology файла

```bash
# Найти path topology файла
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py list topology 3 mvp-cloud

# Получить полный файл и вывести все namespaces с алертами
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py fetch \
  "raw/kubevious/mvp-cloud/<ts>/topology.json" | python3 -c "
import sys, json
d = json.load(sys.stdin)
nodes = d.get('nodes', [])
print(f'Total nodes: {len(nodes)}')
print()
for n in nodes:
    alerts = n.get('alerts')
    if alerts and alerts != '[]' and alerts != 'null':
        try:
            a = json.loads(alerts) if isinstance(alerts, str) else alerts
            if a:
                print(f\"DN: {n['dn']}\")
                print(f\"Alerts: {json.dumps(a, indent=2)[:300]}\")
                print('---')
        except:
            pass
"
```

#### Показать дерево namespace'ов

```bash
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py fetch \
  "raw/kubevious/mvp-cloud/<ts>/topology.json" | python3 -c "
import sys, json
d = json.load(sys.stdin)
nodes = d.get('nodes', [])
# Показать первые 3 уровня дерева
for n in nodes:
    dn = n.get('dn', '')
    depth = dn.count('/')
    if depth <= 3:
        print('  ' * depth + dn.split('/')[-1] + f'  ({dn})')
" | head -50
```

### Структура данных Kubevious в Qdrant

```
embed_text:
  "kubevious topology: root/ns[argocd] | root/ns[holmesgpt] |
   root/ns[kb-system] | root/ns[monitoring] | ..."

Полный JSON (в MinIO raw/kubevious/...):
  {
    "nodes": [
      {
        "dn": "root/ns[kb-system]/app[minio]/deploy[minio]",
        "node": {"kind": "Deployment", "name": "minio", ...},
        "props": {"replicas": 1, "ready": 1, ...},
        "alerts": []
      },
      ...
    ]
  }

DN (Distinguished Name) — путь в дереве:
  root/ns[<namespace>]/app[<app>]/deploy[<name>]
  root/ns[<namespace>]/app[<app>]/pod[<name>]
  root/ns[<namespace>]/svc[<name>]
```

---

## Комбинированные запросы (все источники)

### Security + Lint: общий health check кластера

```bash
$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import urllib.request, json
req = urllib.request.Request(
    'http://localhost:5050/api/chat',
    data=json.dumps({
        'ask': 'Сделай полный health check кластера mvp-cloud. '
               'Используй kb_search и kb_list_findings для получения данных из всех источников: '
               'kubescape (security), popeye (lint), kubevious (topology). '
               'Составь итоговый отчёт с приоритизированным списком проблем.'
    }).encode(),
    headers={'Content-Type': 'application/json'}, method='POST'
)
with urllib.request.urlopen(req, timeout=240) as r:
    print(json.loads(r.read())['analysis'])
"
```

### Сравнение topology и security findings

```bash
$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import urllib.request, json
req = urllib.request.Request(
    'http://localhost:5050/api/chat',
    data=json.dumps({
        'ask': 'Используя данные из kb-stack: '
               '1) Найди все namespaces через topology '
               '2) Проверь security findings от kubescape для каждого namespace '
               '3) Покажи какие namespaces имеют наибольшие security риски'
    }).encode(),
    headers={'Content-Type': 'application/json'}, method='POST'
)
with urllib.request.urlopen(req, timeout=240) as r:
    print(json.loads(r.read())['analysis'])
"
```

---

## Работа с инструментами напрямую

### kb_search — семантический поиск

```bash
# Синтаксис
python3 /kb-scripts/kb_tools.py search "<запрос>" [top_k] [cluster_id]

# Примеры
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py search "pods without CPU memory limits" 10 mvp-cloud

$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py search "security vulnerability privilege" 5 mvp-cloud

# Вывод:
# [score=0.648] doc_type=finding_kubescape
#   source: raw/kubescape/mvp-cloud/20260225T.../findings.json
#   text: kubescape security findings: Non-root containers(54 failed), ...
```

**Параметры:**
- `top_k` — количество результатов (default: 10)
- `cluster_id` — ID кластера (default: mvp-cloud)
- Score от 0 до 1: чем выше, тем релевантнее

### kb_list_findings — листинг по типу документа

```bash
# Синтаксис
python3 /kb-scripts/kb_tools.py list [doc_type] [limit] [cluster_id]

# Все документы
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py list "" 20 mvp-cloud

# Только kubescape
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py list finding_kubescape 5 mvp-cloud

# Только popeye
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py list finding_popeye 5 mvp-cloud

# Только topology
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py list topology 3 mvp-cloud
```

**doc_type значения:**
| doc_type | Описание |
|---|---|
| `finding_kubescape` | NSA security scan |
| `finding_popeye` | Lint report |
| `topology` | Kubevious topology snapshot |
| `object_card` | Другие объекты |
| `""` (пусто) | Все типы |

### kb_fetch_artifact — получить сырой файл из MinIO

> **Важно**: MinIO S3 API (порт 9000) требует AWS Signature V4. `curl -u user:pass`
> (HTTP Basic Auth) не работает — MinIO вернёт XML ошибку `InvalidRequest`.
> `kb_fetch_artifact` использует `kb_tools.py fetch` (boto3), который выполняет
> корректную подпись автоматически.

```bash
# 1. Найти path через kb_list
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py list finding_kubescape 1 mvp-cloud
# → source: raw/kubescape/mvp-cloud/20260225T190008Z/findings.json

# 2. Скачать файл через kb_tools.py fetch (boto3, работает корректно)
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py fetch \
  "raw/kubescape/mvp-cloud/20260225T190008Z/findings.json" | head -60

# 3. Попросить HolmesGPT вызвать инструмент:
# "используй kb_fetch_artifact с path=raw/kubescape/mvp-cloud/<ts>/findings.json"
```

---

## Как HolmesGPT выбирает инструменты

HolmesGPT использует LLM (Qwen3 через LMStudio) и сам решает когда вызывать `kb_search`, `kb_list_findings`, `kb_fetch_artifact`. Чтобы направить его в нужную сторону:

| Хочу | Как формулировать запрос |
|---|---|
| Semantic search | "найди в базе знаний...", "поищи в kb-stack..." |
| Конкретный тип | "получи kubescape данные", "используй popeye отчёт" |
| Полный файл | "получи полный JSON файл из MinIO", "используй kb_fetch_artifact" |
| Все источники | "проверь все источники kb-stack", "сделай полный анализ" |
| Только поиск | "используй kb_search с запросом..." |
| Только список | "используй kb_list_findings с doc_type=..." |

### Пример реального вызова (верифицировано 2026-02-25)

Запрос: `"Используй kb_search чтобы найти security проблемы в кластере mvp-cloud"`

Лог HolmesGPT (из `kubectl logs`):
```
✅ Toolset kb/stack
Received /api/chat request: model=None
Running tool #1 kb_search: python3 /kb-scripts/kb_tools.py search "'security problems'" 10 mvp-cloud
  Finished #1 in 0.12s, output length: 4,866 characters
Running tool #2 kb_search: python3 /kb-scripts/kb_tools.py search "'kubescape security findings'" 5 mvp-cloud
  Finished #2 in 0.13s, output length: 2,678 characters
Running tool #3 kb_list_findings: python3 /kb-scripts/kb_tools.py list finding_kubescape 10 mvp-cloud
  Finished #3 in 0.08s
Running tool #4 kb_fetch_artifact: ...
  Finished #4 in 0.21s
```

Фрагмент ответа LLM:
```
## Security Issues Found in mvp-cloud Cluster

### Critical Security Issues:
1. **Non-root containers** - 54 failed checks
2. **Allow privilege escalation** - 45 failed checks
3. **Immutable container filesystem** - 48 failed checks

### High Severity Issues:
1. **Ingress and Egress blocked** - 55 failed checks
2. **Automatic mapping of service account** - 9 failed checks
3. **Administrative Roles** - 3 failed checks
```

### Подсказка к описанию toolset

```yaml
# platform/holmesgpt/kb-stack-toolset.yaml
# Описание toolset (что видит LLM):
description: |
  Query Kubernetes cluster knowledge base built from Kubescape security scans,
  Popeye linting reports, and Kubevious topology snapshots.
  Use these tools to find security findings, topology info, and resource issues.
```

---

## Скрипт полного health check

Запустить полный анализ и сохранить результат:

```bash
export K="kubectl --kubeconfig ~/proj/cross/kubeconfig_6005021"
HOLMES_POD=$($K get pod -n holmesgpt -l app.kubernetes.io/name=holmes \
  -o jsonpath='{.items[0].metadata.name}')

$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import urllib.request, json, datetime

API = 'http://localhost:5050'
questions = [
    ('Kubescape Security',
     'Используй kb_list_findings с doc_type=finding_kubescape и kb_fetch_artifact '
     'чтобы получить последний security scan. '
     'Перечисли ВСЕ failed controls с количеством затронутых ресурсов. '
     'Расставь по приоритету от критичного к низкому.'),

    ('Popeye Lint',
     'Используй kb_list_findings с doc_type=finding_popeye и kb_fetch_artifact '
     'чтобы получить последний popeye отчёт. '
     'Перечисли все sanitizers где error > 0 или warning > 5.'),

    ('Topology Overview',
     'Используй kb_list_findings с doc_type=topology и kb_fetch_artifact '
     'чтобы получить topology snapshot. '
     'Покажи список всех namespaces и есть ли у каких-то ресурсов алерты.'),
]

for title, question in questions:
    print(f'\\n{'='*60}')
    print(f'  {title}')
    print(f'{'='*60}\\n')

    req = urllib.request.Request(
        f'{API}/api/chat',
        data=json.dumps({'ask': question}).encode(),
        headers={'Content-Type': 'application/json'},
        method='POST'
    )
    try:
        with urllib.request.urlopen(req, timeout=300) as r:
            resp = json.loads(r.read())
            print(resp['analysis'])
    except Exception as e:
        print(f'ERROR: {e}')
" 2>&1 | tee /tmp/kb-health-$(date +%Y%m%d).txt

echo "Результат сохранён в /tmp/kb-health-$(date +%Y%m%d).txt"
```

---

## Отладка и диагностика

### Проверить что toolset загружен и активен

```bash
HOLMES_POD=$($K get pods -n holmesgpt --no-headers | grep "holmesgpt-holmes.*Running" | awk '{print $1}')

# 1. Файлы смонтированы
$K exec -n holmesgpt $HOLMES_POD -- ls -la \
  /app/holmes/plugins/toolsets/kb-stack-toolset.yaml \
  /kb-scripts/kb_tools.py

# 2. Toolset имеет статус ENABLED (не DISABLED!)
$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import sys; sys.path.insert(0,'/venv/lib/python3.11/site-packages'); sys.path.insert(0,'/app')
from holmes.config import Config
te = Config.load_from_env().create_toolcalling_llm(dal=None).tool_executor
for ts in te.toolsets:
    if 'kb' in ts.name:
        print(f'{ts.name}: enabled={ts.enabled}, tools={[t.name for t in ts.tools]}')
" 2>&1 | grep -v WARNING
# Ожидается: kb/stack: enabled=True, tools=['kb_search', 'kb_list_findings', 'kb_fetch_artifact']

# 3. Проверить по логам старта пода
$K logs -n holmesgpt $HOLMES_POD | grep "Toolset kb"
# Ожидается: ✅ Toolset kb/stack

# 4. Если toolset DISABLED — убедиться что в Helm values есть:
$K get application holmesgpt -n argocd \
  -o jsonpath='{.spec.source.helm.values}' | grep -A1 "kb/stack"
# Ожидается:
#   kb/stack:
#     enabled: true
```

### Проверить embedding-svc (нужен для kb_search)

```bash
$K run emb-check --rm -i --restart=Never -n kb-system --image=curlimages/curl \
  --command -- sh -c '
    echo "=== health ==="
    curl -s http://embedding-svc.kb-system.svc:7997/health
    echo ""
    echo "=== test embedding ==="
    curl -s -X POST http://embedding-svc.kb-system.svc:7997/v1/embeddings \
      -H "Content-Type: application/json" \
      -d "{\"input\":\"test query\",\"model\":\"all-MiniLM-L6-v2\"}" \
      | python3 -c "import sys,json; d=json.load(sys.stdin); print(f\"vector_size={len(d['"'"'data'"'"'][0]['"'"'embedding'"'"'])}\")"
  '
# Ожидается: {"status": "ok", "vector_size": 384} и vector_size=384
```

### Проверить Qdrant коллекцию

```bash
$K run qdrant-check --rm -i --restart=Never -n kb-system --image=curlimages/curl \
  --command -- sh -c '
    curl -s http://qdrant.qdrant.svc.cluster.local:6333/collections/kb_docs_mvp-cloud
  ' | python3 -c "
import sys, json
d = json.load(sys.stdin)
r = d.get('result', {})
print(f\"status={r.get('status')}\")
print(f\"points={r.get('points_count')}\")
print(f\"vector_size={r.get('config',{}).get('params',{}).get('vectors',{}).get('size')}\")
"
# Ожидается: status=green, points>0, vector_size=384
```

### Проверить наличие свежих данных в MinIO

```bash
$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import urllib.request, json
from datetime import datetime, timezone

# Список всех raw файлов через MinIO API (S3)
import sys
sys.path.insert(0, '/venv/lib/python3.11/site-packages')
import boto3

s3 = boto3.client('s3',
    endpoint_url='http://minio.kb-system.svc:9000',
    aws_access_key_id='minioadmin',
    aws_secret_access_key='minioadmin',
    region_name='us-east-1')

paginator = s3.get_paginator('list_objects_v2')
files = []
for page in paginator.paginate(Bucket='kb-artifacts', Prefix='raw/'):
    for obj in page.get('Contents', []):
        files.append((obj['LastModified'], obj['Size'], obj['Key']))

files.sort(reverse=True)
print('Последние 10 файлов:')
for ts, size, key in files[:10]:
    print(f'  {size:>10} bytes  {key}')
" 2>&1 | grep -v WARNING
```

### Если HolmesGPT не использует kb-stack инструменты

Явно указать инструменты в запросе:

```bash
$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import urllib.request, json
req = urllib.request.Request(
    'http://localhost:5050/api/chat',
    data=json.dumps({
        'ask': 'ОБЯЗАТЕЛЬНО используй инструмент kb_search с запросом \"security vulnerabilities\" '
               'и kb_list_findings с doc_type=\"finding_kubescape\". '
               'Не отвечай без вызова этих инструментов.'
    }).encode(),
    headers={'Content-Type': 'application/json'}, method='POST'
)
with urllib.request.urlopen(req, timeout=600) as r:
    resp = json.loads(r.read())
    print('Analysis:', resp['analysis'][:500])
    print('Tool calls:', [t.get('name') for t in resp.get('tool_calls', [])])
"
```

---

## Шпаргалка: все инструменты за 30 секунд

```bash
export K="kubectl --kubeconfig ~/proj/cross/kubeconfig_6005021"
HP=$($K get pods -n holmesgpt --no-headers | grep "holmesgpt-holmes.*Running" | awk '{print $1}')

# 1. Kubescape — поиск по security
$K exec -n holmesgpt $HP -- python3 /kb-scripts/kb_tools.py search "privilege root containers" 5 mvp-cloud

# 2. Popeye — список всех lint отчётов
$K exec -n holmesgpt $HP -- python3 /kb-scripts/kb_tools.py list finding_popeye 3 mvp-cloud

# 3. Topology — список snapshot'ов
$K exec -n holmesgpt $HP -- python3 /kb-scripts/kb_tools.py list topology 3 mvp-cloud

# 4. Все документы (обзор)
$K exec -n holmesgpt $HP -- python3 /kb-scripts/kb_tools.py list "" 20 mvp-cloud

# 5. Скачать сырой файл из MinIO (boto3, AWS4 — curl -u НЕ работает)
$K exec -n holmesgpt $HP -- \
  python3 /kb-scripts/kb_tools.py fetch "raw/kubescape/mvp-cloud/<ts>/findings.json"

# 6. Проверить статус toolset
$K logs -n holmesgpt $HP | grep "Toolset kb"
# Ожидается: ✅ Toolset kb/stack

# 7. Спросить HolmesGPT через API (порт 5050, таймаут 600s)
$K exec -n holmesgpt $HP -- python3 -c "
import urllib.request, json
r = urllib.request.urlopen(urllib.request.Request(
    'http://localhost:5050/api/chat',
    json.dumps({'ask': 'Найди security проблемы в кластере используя kb-stack'}).encode(),
    {'Content-Type': 'application/json'}, 'POST'), timeout=600)
print(json.loads(r.read())['analysis'])
"
```
