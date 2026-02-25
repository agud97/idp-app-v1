# kb-stack: Инструкция по эксплуатации

**Кластер**: mvp-cloud
**Namespace**: kb-system
**Kubeconfig**: `~/proj/cross/kubeconfig_6005021`
**GitOps repo**: `~/proj/cross/idp-app-v1`

---

## Архитектура

```
┌─────────────────────────────────────────────────────────────────┐
│                         kb-system namespace                      │
│                                                                  │
│  ┌──────────────┐  ┌────────────┐  ┌──────────────────────┐    │
│  │  kubescape   │  │   popeye   │  │ kubevious-exporter   │    │
│  │  (CronJob)   │  │  (CronJob) │  │ (CronJob + MySQL)    │    │
│  │  NSA scan    │  │  lint      │  │ topology fetch       │    │
│  └──────┬───────┘  └─────┬──────┘  └──────────┬───────────┘    │
│         │                │                    │                 │
│         └────────────────┴────────────────────┘                 │
│                          │ raw JSON                             │
│                          ▼                                      │
│                   ┌─────────────┐                               │
│                   │    MinIO    │  kb-artifacts/raw/...         │
│                   │  (S3-совм.) │  normalized/...               │
│                   └──────┬──────┘  bundles/...                  │
│                          │                                      │
│                          ▼                                      │
│                   ┌─────────────┐    ┌───────────────┐         │
│                   │ normalizer  │───▶│ embedding-svc │         │
│                   │  (CronJob)  │    │ all-MiniLM    │         │
│                   │  Python     │    │ 384-dim CPU   │         │
│                   └──────┬──────┘    └───────────────┘         │
│                          │ vectors                             │
│                          ▼                                      │
│                   ┌─────────────┐                               │
│                   │   Qdrant    │  kb_docs_mvp-cloud            │
│                   │(qdrant ns)  │  384-dim, Cosine              │
│                   └─────────────┘                               │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │   HolmesGPT           │  namespace: holmesgpt
              │   kb/stack toolset    │
              │   kb_search           │
              │   kb_list_findings    │
              │   kb_fetch_artifact   │
              └───────────────────────┘
```

### Расписание CronJob (каждые 30 минут)
| CronJob | Что делает | Результат в MinIO |
|---|---|---|
| `kubescape-exporter` | NSA security scan (~2-4 мин) | `raw/kubescape/mvp-cloud/<ts>/findings.json` |
| `popeye` | k8s lint check (~1 мин) | `raw/popeye/mvp-cloud/<ts>/report.json` |
| `kubevious-exporter` | topology fetch via API (~2-3 мин) | `raw/kubevious/mvp-cloud/<ts>/topology.json` |
| `normalizer` | embed + upsert to Qdrant | `normalized/docs/mvp-cloud/<ts>/docs.jsonl` |

### ArgoCD Applications
| App | Source | Что управляет |
|---|---|---|
| `apps` | `applications/` (recurse) | Все ArgoCD Application объекты |
| `kb-stack` | `platform/kb-stack/overlays/mvp-cloud` (kustomize) | Все ресурсы kb-system |
| `holmesgpt` | Helm chart `holmes` 0.19.0 | HolmesGPT Deployment |
| `holmesgpt-configs` | `platform/holmesgpt/` (kustomize) | HolmesGPT ConfigMaps |

---

## Проверка корректности работы

### 1. Статус подов

```bash
export K="kubectl --kubeconfig ~/proj/cross/kubeconfig_6005021"

# Все поды kb-system
$K get pods -n kb-system

# Ожидаемый результат:
# embedding-svc-xxx          1/1  Running    ← должен быть Ready
# kubevious-backend-xxx      1/1  Running
# kubevious-collector-xxx    1/1  Running
# kubevious-guard-xxx        1/1  Running
# kubevious-mysql-0          1/1  Running
# kubevious-parser-xxx       1/1  Running
# kubevious-redis-0          1/1  Running
# kubevious-ui-xxx           1/1  Running
# minio-xxx                  1/1  Running
# <последние CronJob поды>   0/1  Completed  ← должны быть Completed, не Error
```

### 2. Проверка ArgoCD sync

```bash
$K get applications -n argocd
# Все приложения должны быть Synced + Healthy
```

### 3. Проверка embedding-сервиса

```bash
$K run emb-check --rm -i --restart=Never --image=curlimages/curl \
  --command -- curl -s http://embedding-svc.kb-system.svc:7997/health
# Ожидается: {"status": "ok", "vector_size": 384}
```

### 4. Проверка MinIO — наличие свежих файлов

```bash
# Получить MinIO credentials
ACCESS_KEY=$($K get secret minio-credentials -n kb-system -o jsonpath='{.data.accessKey}' | base64 -d)
SECRET_KEY=$($K get secret minio-credentials -n kb-system -o jsonpath='{.data.secretKey}' | base64 -d)

# Список файлов с размерами
$K exec -n holmesgpt deployment/holmesgpt-holmes -- python3 -c "
import boto3
s3 = boto3.client('s3', endpoint_url='http://minio.kb-system.svc:9000',
    aws_access_key_id='$ACCESS_KEY', aws_secret_access_key='$SECRET_KEY',
    region_name='us-east-1')
for page in s3.get_paginator('list_objects_v2').paginate(Bucket='kb-artifacts', Prefix='raw/'):
    for o in sorted(page.get('Contents',[]), key=lambda x: x['LastModified'], reverse=True)[:10]:
        print(f\"{o['Size']:>10}  {o['Key']}\")
"

# Нормальные размеры:
# kubescape/findings.json  > 100 КБ
# popeye/report.json       > 10 КБ
# kubevious/topology.json  > 100 КБ
# Если 0 байт — экспортёр упал, см. раздел "Диагностика"
```

### 5. Проверка Qdrant

```bash
$K run qdrant-check --rm -i --restart=Never --image=curlimages/curl \
  --command -- curl -s http://qdrant.qdrant.svc.cluster.local:6333/collections/kb_docs_mvp-cloud

# Ожидаемый результат:
# {
#   "result": {
#     "status": "green",
#     "points_count": <число>,   ← должно быть > 0
#     "config": {
#       "params": {
#         "vectors": {"size": 384, "distance": "Cosine"}  ← НЕ size:1
#       }
#     }
#   }
# }
```

### 6. Проверка semantic search из HolmesGPT пода

```bash
HOLMES_POD=$($K get pod -n holmesgpt -l app.kubernetes.io/name=holmes -o jsonpath='{.items[0].metadata.name}')

# Тест поиска
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py search "CPU memory limits not set" 5 mvp-cloud

# Тест листинга
$K exec -n holmesgpt $HOLMES_POD -- \
  python3 /kb-scripts/kb_tools.py list finding_kubescape 5 mvp-cloud

# Ожидаемый результат: строки вида
# [score=0.5xx] doc_type=finding_kubescape
#   source: raw/kubescape/mvp-cloud/20260225T.../findings.json
#   text: kubescape security findings: Non-root containers(54 failed), ...
```

### 7. Проверка HolmesGPT toolset

```bash
# Убедиться что файлы смонтированы
$K exec -n holmesgpt $HOLMES_POD -- ls -la \
  /app/holmes/plugins/toolsets/kb-stack-toolset.yaml \
  /kb-scripts/kb_tools.py

# Проверить что toolset валидный
$K exec -n holmesgpt $HOLMES_POD -- python3 -c "
import yaml
d = yaml.safe_load(open('/app/holmes/plugins/toolsets/kb-stack-toolset.yaml'))
for name, cfg in d['toolsets'].items():
    print(f'toolset: {name}')
    print(f'tools:   {[t[\"name\"] for t in cfg.get(\"tools\", [])]}')
"
# Ожидается: toolset: kb/stack, tools: ['kb_search', 'kb_list_findings', 'kb_fetch_artifact']
```

---

## Ручной запуск

### Запустить экспортёр немедленно (не ждать cron)

```bash
# Запустить все три экспортёра и normalizer
TS=$(date +%s)
$K create job -n kb-system kubescape-manual-$TS --from=cronjob/kubescape-exporter
$K create job -n kb-system popeye-manual-$TS   --from=cronjob/popeye
$K create job -n kb-system kubevious-manual-$TS --from=cronjob/kubevious-exporter

# Дождаться завершения (kubescape самый долгий, ~3-5 мин)
$K wait job -n kb-system kubescape-manual-$TS --for=condition=Complete --timeout=600s
$K wait job -n kb-system kubevious-manual-$TS --for=condition=Complete --timeout=600s

# Запустить normalizer (embed + Qdrant upsert)
$K create job -n kb-system normalizer-manual-$TS --from=cronjob/normalizer
$K wait job -n kb-system normalizer-manual-$TS --for=condition=Complete --timeout=300s

# Проверить результат
$K logs -n kb-system job/normalizer-manual-$TS | grep -v "Downloading\|━\|Collecting"
```

### Проследить за выполнением экспортёра

```bash
# Логи kubescape (initContainer scanner + main container uploader)
$K logs -n kb-system <pod-name> -c scanner   # initContainer
$K logs -n kb-system <pod-name> -c uploader  # main container

# Логи popeye
$K logs -n kb-system <pod-name> -c popeye        # initContainer
$K logs -n kb-system <pod-name> -c extract-json  # initContainer
$K logs -n kb-system <pod-name> -c uploader      # main container

# Логи kubevious
$K logs -n kb-system <pod-name> -c get-snapshot-id  # initContainer
$K logs -n kb-system <pod-name> -c fetcher          # initContainer
$K logs -n kb-system <pod-name> -c uploader         # main container
```

---

## Как изменять конфигурацию

> Всё через git → ArgoCD синкает автоматически.

### Изменить расписание CronJob

```bash
# Файл: platform/kb-stack/overlays/mvp-cloud/cluster-config.yaml
# Поле: SCHEDULE: "*/30 * * * *"
# После изменения:
git add platform/kb-stack/overlays/mvp-cloud/cluster-config.yaml
git commit -m "Update schedule"
git push origin main
```

### Добавить новый тулсет для HolmesGPT

```bash
# 1. Создать файл platform/holmesgpt/my-toolset.yaml
# 2. Добавить в platform/holmesgpt/kustomization.yaml
# 3. Добавить в applications/holmesgpt.yaml (additionalVolumes + additionalVolumeMounts)
# 4. Push в git — ArgoCD синкнет оба приложения
git add platform/holmesgpt/my-toolset.yaml \
        platform/holmesgpt/kustomization.yaml \
        applications/holmesgpt.yaml
git commit -m "Add new HolmesGPT toolset"
git push origin main
```

### Изменить параметры embedding-сервиса

```bash
# Файл: platform/kb-stack/base/embedding-svc/embedding-svc.yaml
# Переменные: EMBEDDING_MODEL, PORT
# После изменения — ArgoCD обновит Deployment
git add platform/kb-stack/base/embedding-svc/embedding-svc.yaml
git commit && git push
```

### Принудительный ArgoCD sync (если не синкается автоматически)

```bash
# Форс-обновить конкретное приложение
$K annotate application kb-stack -n argocd argocd.argoproj.io/refresh=hard --overwrite

# Форс-обновить app-of-apps (чтобы он подхватил новые Application файлы)
$K patch application apps -n argocd --type=merge \
  -p '{"operation":{"sync":{"revision":"HEAD","syncStrategy":{"hook":{}}}}}'
```

---

## Диагностика типичных проблем

### Экспортёр завершился с ошибкой

```bash
# Найти последний pod экспортёра
$K get pods -n kb-system -l app=kubescape-exporter --sort-by=.metadata.creationTimestamp | tail -3

# Посмотреть все контейнеры (initContainers + containers)
$K describe pod -n kb-system <pod-name> | grep -A 3 "State:\|Exit Code:\|Reason:"

# Логи initContainer
$K logs -n kb-system <pod-name> -c scanner
```

### Файлы в MinIO = 0 байт

Причины:
1. Экспортёр упал в initContainer — смотри логи `-c scanner` / `-c popeye` / `-c fetcher`
2. MinIO недоступен — проверь `$K get pods -n kb-system -l app=minio`
3. RBAC проблема — `$K logs -n kb-system <pod-name> -c uploader`

```bash
# Проверить что MinIO отвечает
$K run minio-ping --rm -i --restart=Never -n kb-system --image=curlimages/curl \
  --command -- curl -s -o /dev/null -w "%{http_code}" \
  http://minio.kb-system.svc:9000/minio/health/live
# Ожидается: 200
```

### Qdrant: 0 points или size=1

```bash
# Причина: normalizer не запустился или embedding-svc недоступен
$K logs -n kb-system <normalizer-pod> | grep -v "Downloading\|━" | tail -20

# Проверить embedding-svc
$K run emb-test --rm -i --restart=Never --image=curlimages/curl \
  --command -- sh -c '
    curl -s http://embedding-svc.kb-system.svc:7997/health
    echo ""
    curl -s -X POST http://embedding-svc.kb-system.svc:7997/v1/embeddings \
      -H "Content-Type: application/json" \
      -d "{\"input\":\"test\",\"model\":\"all-MiniLM-L6-v2\"}" | head -c 200
  '
```

### embedding-svc в CrashLoopBackOff

```bash
$K describe pod -n kb-system -l app=embedding-svc
# Проверить Exit Code:
# 137 = OOMKilled или убит startupProbe (интернет недоступен для pip install)
# 1   = ошибка в скрипте

$K logs -n kb-system deployment/embedding-svc --previous | tail -20
```

### HolmesGPT не видит toolset

```bash
HOLMES_POD=$($K get pod -n holmesgpt -l app.kubernetes.io/name=holmes -o jsonpath='{.items[0].metadata.name}')

# Проверить что volume смонтирован
$K exec -n holmesgpt $HOLMES_POD -- ls /app/holmes/plugins/toolsets/

# Если файла нет — проверить ArgoCD sync
$K get application holmesgpt holmesgpt-configs -n argocd

# Проверить volumes в Deployment
$K get deployment holmesgpt-holmes -n holmesgpt \
  -o jsonpath='{.spec.template.spec.volumes[*].name}'
```

### Сброс и переиндексация Qdrant

```bash
# Удалить коллекцию и переиндексировать заново
$K run qdrant-reset --rm -i --restart=Never --image=curlimages/curl \
  --command -- curl -s -X DELETE \
  http://qdrant.qdrant.svc.cluster.local:6333/collections/kb_docs_mvp-cloud

# Запустить normalizer — создаст коллекцию заново
$K create job -n kb-system normalizer-reindex-$(date +%s) --from=cronjob/normalizer
```

---

## Мониторинг

### Проверить что все CronJob'ы успешно выполнялись

```bash
# Последние 5 выполнений каждого CronJob
for cj in kubescape-exporter popeye kubevious-exporter normalizer; do
  echo "=== $cj ==="
  $K get pods -n kb-system -l app=$cj \
    --sort-by=.metadata.creationTimestamp \
    -o custom-columns="NAME:.metadata.name,STATUS:.status.phase,AGE:.metadata.creationTimestamp" \
    | tail -5
done
```

### Быстрая проверка состояния всего стека

```bash
echo "=== ArgoCD Apps ==="
$K get applications -n argocd --no-headers | grep -E "(kb-stack|holmesgpt)"

echo "=== kb-system pods ==="
$K get pods -n kb-system --no-headers | grep -v "Completed"

echo "=== Qdrant collection ==="
$K run q --rm -i --restart=Never --image=curlimages/curl \
  --command -- curl -s http://qdrant.qdrant.svc.cluster.local:6333/collections/kb_docs_mvp-cloud \
  | grep -o '"points_count":[0-9]*\|"size":[0-9]*'

echo "=== Embedding-svc ==="
$K run e --rm -i --restart=Never -n kb-system --image=curlimages/curl \
  --command -- curl -s http://embedding-svc.kb-system.svc:7997/health
```

---

## Ключевые адреса сервисов (внутри кластера)

| Сервис | Адрес |
|---|---|
| MinIO S3 API | `http://minio.kb-system.svc:9000` |
| Qdrant HTTP API | `http://qdrant.qdrant.svc.cluster.local:6333` |
| Embedding Service | `http://embedding-svc.kb-system.svc:7997` |
| Kubevious Backend | `http://kubevious-backend-clusterip.kb-system.svc:4000` |
| HolmesGPT | `http://holmesgpt-holmes.holmesgpt.svc` |
| LLM Proxy | `http://llm-proxy.llm-proxy.svc.cluster.local:8080` |

## MinIO credentials

```bash
$K get secret minio-credentials -n kb-system -o json | python3 -c "
import json,sys,base64
d=json.load(sys.stdin)
for k,v in d['data'].items(): print(k,'=', base64.b64decode(v).decode())
"
```
