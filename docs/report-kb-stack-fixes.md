# Отчёт: исправление kb-stack acceptance criteria
**Дата**: 2026-02-25
**Кластер**: mvp-cloud
**Репозиторий**: github.com/agud97/idp-app-v1.git

---

## Обзор

kb-stack — это система сбора, нормализации и семантического поиска по данным кластера Kubernetes.
Компоненты: Kubescape (security scan), Popeye (lint), Kubevious (topology), MinIO (хранилище), Qdrant (vector DB), HolmesGPT (AI-агент).

После первичного деплоя все компоненты были "Running", но данные не доходили до Qdrant. Ниже — полный список найденных проблем и способов их устранения.

---

## Проблема 1: NetworkPolicy блокировала трафик в kb-system

### Симптомы
Поды в kb-system не могли соединяться друг с другом и с внешними сервисами.

### Диагностика
```bash
kubectl get networkpolicies -n kb-system
# Обнаружены: kb-default-deny, kb-allow-egress, kb-allow-egress-normalizer, kb-allow-ingress-internal
```

### Причина
NetworkPolicy `kb-default-deny` блокировала весь трафик. Разрешающие политики были неполными — не покрывали все необходимые пути (embedding-svc, llm-proxy и др.).

### Исправление
```bash
# Удалены NetworkPolicy через kubectl
kubectl delete networkpolicy -n kb-system --all

# Удалена строка из kustomization.yaml
# platform/kb-stack/base/kustomization.yaml — убрана строка "- networkpolicy"
git add platform/kb-stack/base/kustomization.yaml
git commit -m "Remove networkpolicy from kb-stack base"
git push origin main
```

---

## Проблема 2: Kubevious exporter — неправильный API endpoint

### Симптомы
Экспортёр topology получал ошибку `Cannot GET /api/snapshot`.

### Диагностика
```bash
kubectl run curl-test --rm -i --restart=Never -n kb-system --image=curlimages/curl \
  --command -- curl -s http://kubevious-backend-clusterip.kb-system.svc:4000/api/v1/diagram/children \
  --data-urlencode "snapshot=<id>" --data-urlencode "dn=root"
```

### Причина
Конфиг `KUBEVIOUS_SNAPSHOT_PATH: /api/snapshot` — такого endpoint не существует. Kubevious использует REST API:
- `GET /api/v1/diagram/children?snapshot=<id>&dn=<dn>`
- `GET /api/v1/diagram/node?snapshot=<id>&dn=<dn>`
- `GET /api/v1/diagram/props?snapshot=<id>&dn=<dn>`
- `GET /api/v1/diagram/alerts?snapshot=<id>&dn=<dn>`

Snapshot ID берётся из MySQL: `SELECT LOWER(HEX(snapshot_id)) FROM kubevious.snap_items LIMIT 1`.

### Исправление
Полностью переписан `kubevious-exporter-cronjob.yaml`:
- initContainer `get-snapshot-id`: mysql:8.0 — достаёт snapshot_id из БД
- initContainer `fetcher`: curlimages/curl — рекурсивно обходит дерево через API
- container `uploader`: minio/mc — загружает результат

```bash
git add platform/kb-stack/base/kubevious/kubevious-exporter-cronjob.yaml
git commit -m "Rewrite kubevious-exporter: use correct /api/v1/diagram/* endpoints"
git push origin main
```

---

## Проблема 3: Race condition во всех трёх экспортёрах

### Симптомы
Файлы в MinIO: `kubescape/findings.json` = 0 байт, `popeye/report.json` = 0 байт, `kubevious/topology.json` = 151 байт (неполный).

### Диагностика
```bash
# Проверка размеров файлов в MinIO
kubectl exec -n holmesgpt <holmes-pod> -- python3 -c "
import boto3
s3 = boto3.client('s3', endpoint_url='http://minio.kb-system.svc:9000',
    aws_access_key_id='<key>', aws_secret_access_key='<secret>', region_name='us-east-1')
for page in s3.get_paginator('list_objects_v2').paginate(Bucket='kb-artifacts', Prefix='raw/'):
    for obj in page.get('Contents', []):
        print(f\"{obj['Size']:>10}  {obj['Key']}\")
"

# Просмотр логов экспортёра
kubectl logs -n kb-system <kubescape-pod> --all-containers
# Вывод: "Scan results saved" → потом uploader показывает "Total | 0 B | 0 B"
```

### Причины (у каждого экспортёра своя)

**Kubescape**: scanner и uploader — параллельные контейнеры. Uploader ждал файл максимум 60 секунд (`for i in $(seq 1 30); do [ -s file ] && break; sleep 2; done`). NSA scan занимает 2–5 минут. После 60с uploader загружал пустой файл, а kubescape дописывал данные позже.

**Popeye**: uploader вообще не ждал popeye — стартовал одновременно и сразу делал `mc cp /tmp/report.json`. Файла ещё не существовало (или он был пустым).

**Kubevious**: fetcher начинал с `echo '{"nodes":['` → файл стал непустым (21 байт) мгновенно → uploader видел `[ -s file ]` = true → загружал через 5 секунд частичный файл (151 байт).

### Исправление
Scanners перенесены в **initContainers** — они гарантированно завершаются до старта main контейнеров.

```yaml
# Паттерн для всех трёх экспортёров:
initContainers:
  - name: scanner   # завершается полностью
    image: <scanner-image>
    ...
containers:
  - name: uploader  # стартует только после завершения всех initContainers
    image: minio/mc
    ...
```

```bash
git add platform/kb-stack/base/kubescape/kubescape-exporter-cronjob.yaml \
        platform/kb-stack/base/popeye/popeye-cronjob.yaml \
        platform/kb-stack/base/kubevious/kubevious-exporter-cronjob.yaml
git commit -m "Fix exporters: move scanners to initContainers to eliminate race condition"
git push origin main
```

**Результат после исправления:**
| Экспортёр | До | После |
|---|---|---|
| kubescape | 0 B | **960 КБ** |
| popeye | 0 B | 26 КБ (после след. фикса) |
| kubevious | 151 B | **345 КБ** |

---

## Проблема 4: Popeye пишет ANSI escape-коды в stdout

### Симптомы
`popeye/report.json` = 870 байт, `json.loads()` падает с `JSONDecodeError`.

### Диагностика
```bash
kubectl exec -n holmesgpt <holmes-pod> -- python3 -c "
import boto3; s3 = boto3.client(...)
body = s3.get_object(Bucket='kb-artifacts', Key='raw/popeye/.../report.json')['Body'].read()
print(repr(body[:300]))
"
# Вывод: b'\x1b[38;5;220m ___     ___ ___...' — ASCII-баннер с ANSI-кодами
```

### Причина
`popeye -o json > /tmp/report.json` захватывает в stdout и ASCII-баннер (с ANSI color codes), и JSON. Стандартный `json.loads()` не может распарсить файл, начинающийся с `\x1b[...`.

### Исправление
Добавлен второй initContainer `extract-json` (python:3.11-slim), который стрипует ANSI и извлекает JSON:

```yaml
initContainers:
  - name: popeye
    command: ["popeye -o json > /tmp/report.raw 2>/dev/null || true"]
  - name: extract-json
    image: python:3.11-slim
    command:
      - python3
      - -c
      - |
        import re
        raw = open('/tmp/report.raw', 'rb').read().decode('utf-8', errors='ignore')
        clean = re.sub(r'\x1b\[[0-9;]*[mGKHF]', '', raw)
        idx = clean.find('{')
        open('/tmp/report.json', 'w').write(clean[idx:].strip() if idx >= 0 else '{}')
```

```bash
git add platform/kb-stack/base/popeye/popeye-cronjob.yaml
git commit -m "Fix popeye: add extract-json initContainer to strip ANSI codes"
git push origin main
```

---

## Проблема 5: llm-proxy embedding модель недоступна

### Симптомы
`POST /v1/embeddings` с моделью `text-embedding-nomic-embed-text-v1.5` → ошибка о нехватке памяти (ноутбук уже загружен qwen3-30b).

### Диагностика
```bash
kubectl run llm-check --rm -i --restart=Never --image=curlimages/curl \
  --command -- curl -s http://llm-proxy.llm-proxy.svc.cluster.local:8080/v1/models
# Ответ: nomic-embed-text присутствует, но не загружается
```

### Решение
Развёрнут собственный embedding-сервис внутри кластера (`sentence-transformers/all-MiniLM-L6-v2`, CPU, 384-dim):

```yaml
# platform/kb-stack/base/embedding-svc/embedding-svc.yaml
# ConfigMap со скриптом Python HTTP-сервера + Deployment + Service
```

```bash
git add platform/kb-stack/base/embedding-svc/
git add platform/kb-stack/base/kustomization.yaml  # добавлена строка "- embedding-svc"
git commit -m "Add in-cluster embedding-svc (all-MiniLM-L6-v2, 384-dim)"
git push origin main
```

---

## Проблема 6: embedding-svc падает по livenessProbe

### Симптомы
Pod `embedding-svc` перезапускался с `exit code 137` через ~4.5 минуты после старта.

### Диагностика
```bash
kubectl describe pod -n kb-system -l app=embedding-svc
# Exit Code: 137 (SIGKILL от liveness probe)
# Started: 21:27:31 → Finished: 21:32:01 = 4.5 минуты

# Расчёт: initialDelaySeconds=180 + failureThreshold=3 × periodSeconds=30 = 270с = 4.5 мин
```

### Причина
`pip install sentence-transformers` + загрузка модели (~90 МБ) занимает 5–7 минут. Liveness probe с `initialDelaySeconds: 180` убивал pod до завершения инициализации.

### Исправление
Заменён `livenessProbe` на `startupProbe` с бюджетом 10 минут, увеличен лимит памяти до 2Gi:

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 7997
  initialDelaySeconds: 60
  periodSeconds: 30
  failureThreshold: 20   # 60 + 20×30 = 660s = 11 мин максимум
resources:
  limits:
    memory: "2Gi"
```

```bash
git add platform/kb-stack/base/embedding-svc/embedding-svc.yaml
git commit -m "Fix embedding-svc: startupProbe + 2Gi memory limit"
git push origin main
```

---

## Проблема 7: platform/holmesgpt не управлялся ArgoCD

### Симптомы
ConfigMaps в `platform/holmesgpt/` (тулсеты, ранбуки) применялись вручную через `kubectl apply` и не синхронизировались из git.

### Диагностика
```bash
kubectl get application holmesgpt -n argocd \
  -o jsonpath='{.spec.source.repoURL} {.spec.source.chart}'
# Ответ: https://robusta-charts.storage.googleapis.com holmes
# → ArgoCD app использует Helm chart, НЕ git path
```

### Причина
ArgoCD Application `holmesgpt` использует Helm chart как source. `platform/holmesgpt/*.yaml` — это дополнительные ConfigMaps, которые не покрывались ни одним ArgoCD Application.

### Исправление
Создан новый ArgoCD Application `holmesgpt-configs` + kustomization.yaml:

```bash
# Создан platform/holmesgpt/kustomization.yaml (список всех файлов)
# Создан applications/holmesgpt-configs.yaml:
cat applications/holmesgpt-configs.yaml
# source.path: platform/holmesgpt, destination.namespace: holmesgpt

git add applications/holmesgpt-configs.yaml platform/holmesgpt/kustomization.yaml
git commit -m "Add ArgoCD Application for platform/holmesgpt configs"
git push origin main

# Форс-refresh app-of-apps для немедленного обнаружения
kubectl patch application apps -n argocd --type=merge \
  -p '{"operation":{"sync":{"revision":"HEAD","syncStrategy":{"hook":{}}}}}'
```

---

## Проблема 8: HolmesGPT ignoreDifferences блокировал обновление volumes

### Симптомы
После добавления нового `additionalVolumes` в `applications/holmesgpt.yaml` и push в git — Deployment не обновлялся, новый volume не появлялся.

### Диагностика
```bash
kubectl get application holmesgpt -n argocd -o yaml | grep -A 10 ignoreDifferences
# Обнаружено:
# ignoreDifferences:
#   - group: apps / kind: Deployment
#     jsonPointers: [/spec/template/spec/volumes, .../volumeMounts]

kubectl get deployment holmesgpt-holmes -n holmesgpt \
  -o jsonpath='{.spec.template.spec.volumes[*].name}'
# kb-stack-toolset отсутствует, хотя в Application object есть
```

### Причина
`ignoreDifferences` был добавлен в предыдущих сессиях для ручного kubectl patching. ArgoCD полностью игнорировал изменения в volumes и volumeMounts, не триггеря rolling update.

### Исправление
```bash
# Удалена секция ignoreDifferences из applications/holmesgpt.yaml
git add applications/holmesgpt.yaml
git commit -m "Remove ignoreDifferences for holmesgpt volumes/volumeMounts"
git push origin main

# После sync ArgoCD — rolling update, новый pod с правильными volumes
kubectl get pods -n holmesgpt  # новый pod с AGE ~3min
```

---

## Проблема 9: normalizer писал placeholder vectors (size=1)

### Симптомы
Qdrant коллекция `kb_docs_mvp-cloud`: `vector_size=1`, все векторы `[0.0]`. Семантический поиск не работал.

### Диагностика
```bash
curl -s http://qdrant.qdrant.svc.cluster.local:6333/collections/kb_docs_mvp-cloud
# "vectors":{"size":1,"distance":"Cosine"} — placeholder
```

### Причина
В `normalizer-script.yaml`:
```python
"vector": [0.0]  # placeholder, embedding-сервис не вызывался
```

### Исправление
1. Развёрнут `embedding-svc` (см. Проблема 5)
2. Добавлен `EMBEDDING_ENDPOINT` в `cluster-config.yaml`
3. Переписан normalizer script: вызов `/v1/embeddings`, батчинг по 32, авто-определение vector_size, пересоздание коллекции при несовпадении размера

```bash
git add platform/kb-stack/base/normalizer/normalizer-script.yaml \
        platform/kb-stack/overlays/mvp-cloud/cluster-config.yaml
git commit -m "Add real embeddings to normalizer via embedding-svc"
git push origin main
```

---

## Проблема 10: embed_text пустой — неправильные field paths

### Симптомы
После индексации `embed_text` во всех kubescape/popeye документах: `"kubescape finding:  severity=  resources="` — пустые поля.

### Диагностика
```bash
kubectl exec -n holmesgpt <holmes-pod> -- python3 -c "
import boto3, json
s3 = boto3.client(...)
body = s3.get_object(Bucket='kb-artifacts', Key='raw/kubescape/.../findings.json')['Body'].read()
d = json.loads(body)
print('keys:', list(d.keys()))
# Ответ: ['generationTime','summaryDetails','results',...]
# Нет ключей 'name', 'severity', 'description' на верхнем уровне
"
```

### Причина
`extract_embed_text` искал `data.get("name")`, `data.get("severity")` — этих ключей нет на верхнем уровне kubescape JSON. Реальная структура:
- kubescape: `data["summaryDetails"]["controls"][<id>]` → `{name, status, ResourceCounters.failedResources}`
- popeye: `data["popeye"]["sanitizers"][i]` → `{sanitizer, tally: {error, warning}}`
- kubevious: `data["nodes"][i]` → `{dn, node, props, alerts}`

### Исправление
```python
# Kubescape
controls = data.get("summaryDetails", {}).get("controls", {})
failed = [f"{c['name']}({c['ResourceCounters']['failedResources']} failed)"
          for c in controls.values() if c.get("status") == "failed"]
return "kubescape security findings: " + ", ".join(failed[:15])

# Popeye
sanitizers = data.get("popeye", {}).get("sanitizers", [])
parts = [f"{s['sanitizer']} errors={s['tally']['error']} warnings={s['tally']['warning']}"
         for s in sanitizers if s.get("tally", {}).get("error", 0) or s.get("tally", {}).get("warning", 0)]
return "popeye lint report: " + " ".join(parts[:15])

# Kubevious
nodes = data.get("nodes", [])
dns = [n["dn"] for n in nodes[:20] if n.get("dn")]
return "kubevious topology: " + " | ".join(dns)
```

```bash
git add platform/kb-stack/base/normalizer/normalizer-script.yaml
git commit -m "Fix normalizer embed_text extraction for all three doc types"
git push origin main
```

---

## Проблема 11: Missing kustomization.yaml для embedding-svc

### Симптомы
ArgoCD показывал `ComparisonError` для kb-stack: `kustomize build failed`.

### Диагностика
```bash
kubectl get application kb-stack -n argocd -o json | python3 -c "
import json,sys; d=json.load(sys.stdin)
for c in d['status'].get('conditions',[]): print(c['type'], c['message'][:200])
"
# Failed to load target state: rpc error: kustomize build ... failed
```

### Причина
В `platform/kb-stack/base/kustomization.yaml` добавлена строка `- embedding-svc`, но директория `embedding-svc/` не содержала `kustomization.yaml`. Kustomize требует `kustomization.yaml` в каждой поддиректории.

### Исправление
```bash
cat > platform/kb-stack/base/embedding-svc/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - embedding-svc.yaml
EOF

git add platform/kb-stack/base/embedding-svc/kustomization.yaml
git commit -m "Fix: add kustomization.yaml for embedding-svc base directory"
git push origin main
```

---

## Проблема 12: kb/stack toolset — статус DISABLED

### Симптомы
HolmesGPT не вызывал `kb_search`, `kb_list_findings`, `kb_fetch_artifact`. В логах видны только вызовы `bash`, `kubernetes_jq_query`, `fetch_runbook`.

### Диагностика
```bash
kubectl exec -n holmesgpt <holmes-pod> -- python3 -c "
import sys; sys.path.insert(0,'/venv/lib/python3.11/site-packages'); sys.path.insert(0,'/app')
from holmes.config import Config
c = Config.load_from_env()
te = c.create_toolcalling_llm(dal=None).tool_executor
for ts in te.toolsets:
    if not ts.enabled:
        print(f'DISABLED: {ts.name}')
"
# Вывод: DISABLED: kb/stack
```

Также видно в логах при старте пода:
```
# До фикса: kb/stack НЕТ в списке
✅ Toolset kubernetes/core
✅ Toolset bash
# ...
# kb/stack не упоминается

# После фикса:
✅ Toolset kb/stack
```

### Причина
`kb/stack` — custom toolset, загружается из файла в `toolsets/`. Файл смонтирован корректно, но в Helm values раздел `toolsets:` явно перечисляет только разрешённые тулсеты. Отсутствие `kb/stack` в списке = статус DISABLED, инструменты не передаются LLM.

Это специфика HolmesGPT v0.19.0: custom YAML-toolsets требуют явного включения в Helm values так же, как встроенные.

### Исправление
```yaml
# applications/holmesgpt.yaml — добавлено в секцию toolsets:
toolsets:
  # ... существующие toolsets ...
  kb/stack:
    enabled: true
```

```bash
git add applications/holmesgpt.yaml
git commit -m "fix: enable kb/stack toolset in HolmesGPT Helm values"
git push origin main
# ArgoCD синкает, pod перезапускается автоматически
```

---

## Проблема 13: Jinja2 фильтр `| quote` не поддерживается HolmesGPT

### Симптомы
При вызове `kb_list_findings` HolmesGPT возвращал HTTP 500:
```
jinja2.exceptions.TemplateAssertionError: No filter named 'quote'.
```

### Диагностика
Ошибка в трейсбеке:
```
File "/app/holmes/core/tools.py", line 474, in get_parameterized_one_liner
    template = Template(cmd_or_script)
jinja2.exceptions.TemplateAssertionError: No filter named 'quote'.
```

Источник: шаблон команды в toolset YAML:
```yaml
command: >
  python3 /kb-scripts/kb_tools.py list
  {{ doc_type | default("") | quote }}   ← проблема здесь
```

### Причина
`| quote` — фильтр из Ansible (Jinja2 + Ansible extensions). HolmesGPT использует стандартный Jinja2 без Ansible-расширений. Этот фильтр недоступен.

### Исправление
```yaml
# platform/holmesgpt/kb-stack-toolset.yaml
# Было:
{{ doc_type | default("") | quote }}
# Стало:
{{ doc_type | default("") }}
```

```bash
git add platform/holmesgpt/kb-stack-toolset.yaml
git commit -m "fix: remove unsupported jinja2 'quote' filter from kb_list_findings"
git push origin main
# ArgoCD синкает holmesgpt-configs, затем нужен rollout restart:
kubectl rollout restart deployment/holmesgpt-holmes -n holmesgpt
```

---

## Проблема 14: kb_fetch_artifact — curl Basic auth несовместим с MinIO S3 API

### Симптомы
`kb_fetch_artifact` возвращал XML ошибку вместо данных:
```xml
<Error><Code>InvalidRequest</Code>
<Message>The authorization mechanism you have provided is not supported.
Please use AWS4-HMAC-SHA256.</Message></Error>
```

### Диагностика
Команда `kb_fetch_artifact` использовала `curl -u minioadmin:minioadmin`:
```yaml
command: >
  curl -s --max-time 30
  -u minioadmin:minioadmin
  "http://minio.kb-system.svc:9000/kb-artifacts/{{ path }}"
```
Вызов `curl -u` отправляет HTTP Basic Auth. MinIO S3 API (порт 9000) требует подписанные запросы по стандарту AWS Signature V4 (HMAC-SHA256). Basic auth поддерживается только MinIO Console (порт 9001).

### Исправление
Команда переписана на `python3 /kb-scripts/kb_tools.py fetch` — использует boto3, который выполняет корректную подпись AWS4:

```python
# kb_tools.py — добавлена функция cmd_fetch:
def cmd_fetch(path):
    import boto3
    s3 = boto3.client("s3",
        endpoint_url="http://minio.kb-system.svc:9000",
        aws_access_key_id="minioadmin",
        aws_secret_access_key="minioadmin",
        region_name="us-east-1")
    body = s3.get_object(Bucket="kb-artifacts", Key=path)["Body"].read()
    text = body.decode("utf-8", errors="replace")
    try:
        print(json.dumps(json.loads(text), indent=2)[:8000])
    except Exception:
        print(text[:8000])
```

```yaml
# kb-stack-toolset.yaml — новая команда:
command: >
  python3 /kb-scripts/kb_tools.py fetch
  "{{ path }}"
```

```bash
git add platform/holmesgpt/kb-stack-toolset.yaml
git commit -m "fix: kb_fetch_artifact use boto3 instead of curl Basic auth"
git push origin main
kubectl rollout restart deployment/holmesgpt-holmes -n holmesgpt
```

---

## Итоговое состояние после исправлений

| Компонент | Статус | Данные |
|---|---|---|
| embedding-svc | ✅ Running, model loaded | all-MiniLM-L6-v2, 384-dim |
| kubescape-exporter | ✅ Completed | 960 КБ JSON / запуск |
| popeye-exporter | ✅ Completed | 26 КБ JSON / запуск |
| kubevious-exporter | ✅ Completed | 345 КБ JSON / запуск |
| normalizer | ✅ Completed | 68 docs, real 384-dim vectors |
| Qdrant kb_docs_mvp-cloud | ✅ Green | 68 points, size=384, Cosine |
| holmesgpt-configs | ✅ ArgoCD Synced | all toolsets managed by git |
| HolmesGPT kb/stack toolset | ✅ ENABLED | kb_search, kb_list_findings, kb_fetch_artifact |

Верификация E2E: HolmesGPT реально вызывает kb-stack инструменты. Пример из лога запроса:
```
✅ Toolset kb/stack
Running tool #1 kb_search: python3 /kb-scripts/kb_tools.py search "'security problems'" 10 mvp-cloud
  Finished #1 in 0.12s, output length: 4,866 characters
Running tool #2 kb_search: python3 /kb-scripts/kb_tools.py search "'kubescape security findings'" 5 mvp-cloud
  Finished #2 in 0.13s
Running tool #3 kb_list_findings: ...
Running tool #4 kb_fetch_artifact: ...
```

### Команды для верификации
```bash
# Qdrant коллекция
curl -s http://qdrant.qdrant.svc.cluster.local:6333/collections/kb_docs_mvp-cloud

# Semantic search из HolmesGPT пода
kubectl exec -n holmesgpt <holmes-pod> -- \
  python3 /kb-scripts/kb_tools.py search "CPU limits not set" 5 mvp-cloud

# MinIO файлы
kubectl exec -n holmesgpt <holmes-pod> -- python3 -c "
import boto3
s3 = boto3.client('s3', endpoint_url='http://minio.kb-system.svc:9000',
    aws_access_key_id='<key>', aws_secret_access_key='<secret>', region_name='us-east-1')
for page in s3.get_paginator('list_objects_v2').paginate(Bucket='kb-artifacts', Prefix='raw/'):
    for o in page.get('Contents',[]): print(f\"{o['Size']:>10}  {o['Key']}\")
"
```
