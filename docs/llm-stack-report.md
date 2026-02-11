# Архитектура LLM-стека в Kubernetes

## Обзор

Стек обеспечивает доступ Kubernetes-приложений к LLM (LMStudio), работающему на локальном ноутбуке.
Связность между кластером и ноутбуком реализована через headscale/tailscale VPN-туннель.

```
+------------------+        Tailscale VPN (DERP relay)        +-------------------+
|   Kubernetes     | <--------------------------------------> |    Ноутбук        |
|                  |   100.64.0.1 <---> 100.64.0.2            |                   |
|  +-----------+   |                                          |  +-------------+  |
|  | headscale |   |  Control plane для VPN                   |  | LMStudio    |  |
|  | :8080     |   |  (координация, раздача ключей)           |  | :1234       |  |
|  +-----------+   |                                          |  +-------------+  |
|                  |                                          |  | tailscale   |  |
|  +-----------+   |                                          |  | client      |  |
|  | llm-proxy |   |  nginx + tailscale-sidecar               |  +-------------+  |
|  | :8080     |----->  proxy_pass http://100.64.0.2:1234     +-------------------+
|  +-----------+   |
|       ^          |
|       | http://llm-proxy.llm-proxy.svc:8080/v1
|       |          |
|  +-----------+   +-----------+   +----------+
|  | holmesgpt |   | open-webui|   |  qdrant  |
|  | :5050     |   | :8080     |-->| :6333    |
|  +-----------+   +-----------+   +----------+
+------------------+
```

## Компоненты

### 1. Headscale (control plane VPN)

| Параметр | Значение |
|---|---|
| Namespace | `headscale` |
| Image | `headscale/headscale:0.23.0` |
| Service | `LoadBalancer` 89.108.100.199:8080 (NodePort 30080) |
| server_url | `http://194.58.110.52:30080` |
| DERP | Публичные серверы Tailscale (`controlplane.tailscale.com/derpmap/default`) |
| БД | SQLite (`/var/lib/headscale/db.sqlite`) на PVC |
| DNS | `tailnet.local`, MagicDNS включен |
| IP-пул | `100.64.0.0/10` (v4), `fd7a:115c:a1e0::/48` (v6) |

**Зарегистрированные ноды:**

| Hostname | User | IP | Роль |
|---|---|---|---|
| llm-proxy-* | k8s | 100.64.0.1 | Sidecar в llm-proxy поде |
| macbook-pro-tim | laptop | 100.64.0.2 | Ноутбук с LMStudio |

**Источник манифестов:** Git-репо `platform/headscale/` (файлы: configmap, deployment, namespace, pvc, service).

### 2. LLM-Proxy (nginx + tailscale sidecar)

| Параметр | Значение |
|---|---|
| Namespace | `llm-proxy` |
| Deployment | 2 контейнера: `nginx:alpine` + `tailscale/tailscale:latest` |
| Service | `ClusterIP` на порту `8080` |
| nginx proxy_pass | `http://100.64.0.2:1234` (IP ноутбука в tailscale) |

**Контейнер nginx:**
- Принимает HTTP-запросы на `:8080`
- Проксирует на tailscale IP ноутбука
- Init-контейнер подставляет IP через `envsubst` из ConfigMap
- Таймауты: `proxy_read_timeout 600s`, `proxy_send_timeout 600s`
- Health-check: `GET /health` -> `200 "ok"`

**Контейнер tailscale-sidecar:**
- `TS_USERSPACE=false` (КРИТИЧНО: при `true` не создается tun-устройство, TCP не работает)
- `TS_AUTH_ONCE=true` — авторизуется один раз, состояние хранит в Secret `tailscale-state`
- `--login-server=http://headscale.headscale.svc:8080`
- SecurityContext: `NET_ADMIN`, `NET_RAW` capabilities
- Создает интерфейс `tailscale0` с IP `100.64.0.1`

**Источник манифестов:** Git-репо `platform/llm-proxy/` (файлы: configmap, deployment, namespace, rbac, secret, service).

### 3. HolmesGPT (AI-анализ кластера)

| Параметр | Значение |
|---|---|
| Namespace | `holmesgpt` |
| Chart | `holmes` 0.6.0 (robusta-charts) |
| Image | `holmes:0.6.0` |
| Service | `ClusterIP` порт 80 -> targetPort 5050 |
| Модель | `openai/qwen3-coder-30b-a3b-instruct-mlx` |
| LLM endpoint | `http://llm-proxy.llm-proxy.svc.cluster.local:8080/v1` |

**Ключевые env vars:**

| Переменная | Значение | Примечание |
|---|---|---|
| `MODEL` | `openai/qwen3-coder-30b-a3b-instruct-mlx` | Префикс `openai/` для litellm provider |
| `OPENAI_API_BASE` | `http://llm-proxy.llm-proxy.svc.cluster.local:8080/v1` | КРИТИЧНО: именно `OPENAI_API_BASE`, не `OPENAI_BASE_URL` |
| `OPENAI_API_KEY` | `not-needed` | LMStudio не требует ключ |
| `ALLOWED_TOOLSETS` | `kubernetes/core` | Доступ к kubectl |

**Особенность:** HolmesGPT v0.6.0 в `DefaultLLM.__init__` устанавливает `self.base_url = None` и передает это в `litellm.completion(base_url=None)`. Litellm в этом случае читает `OPENAI_API_BASE` из окружения. Переменная `OPENAI_BASE_URL` НЕ работает.

**RAG/Qdrant:** HolmesGPT **не имеет интеграции с qdrant или RAG**. Контекст для анализа получается не из векторной БД, а через live-выполнение kubectl-команд. Доступные toolsets (`kubernetes/core`, `helm`, `aws`, `docker`, `confluence`, `slab`, `internet`) — все CLI-based. `RunbookManager` — простой regex-матчинг по issue ID/name, не vector retrieval. Qdrant используется только Open WebUI.

**API-эндпоинты:**
- `POST /api/chat` — свободный чат с LLM (имеет доступ к kubectl)
- `POST /api/investigate` — расследование инцидентов
- `POST /api/workload_health_check` — проверка здоровья workload
- `POST /api/conversation` — контекстный диалог по issue
- `GET /api/model` — текущая модель

**Custom Runbooks (база знаний для расследований):**

HolmesGPT поддерживает runbooks — инструкции, которые автоматически подключаются при расследовании инцидентов через `/api/investigate`. Runbooks матчатся по regex-паттернам к названию issue.

Реализация:
- `RunbookManager` загружает runbooks из `/app/holmes/plugins/runbooks/*.yaml`
- Встроенные runbooks: `jira.yaml` (Jira issues), `general.yaml` (KubeSchedulerDown, KubeControllerManagerDown)
- Custom runbooks загружены через ConfigMap `custom-runbooks` в namespace `holmesgpt`, примонтированный как subPath к `/app/holmes/plugins/runbooks/custom-runbooks.yaml`

Custom runbooks (3 шт.):

| Runbook | Regex-паттерн (name) | Описание |
|---|---|---|
| Tailscale/LLM-proxy | `.*tailscale.*\|.*llm-proxy.*\|.*vpn.*` | Проверка TS_USERSPACE, tailscale0, peer connectivity |
| Pod failures | `.*CrashLoopBackOff.*\|.*OOMKilled.*\|.*ImagePullBackOff.*` | Диагностика типичных pod failures |
| LLM connectivity | `.*holmesgpt.*\|.*litellm.*\|.*openai.*\|.*lmstudio.*` | Проверка цепочки holmesgpt → llm-proxy → tailscale → LMStudio |

Итого загружено 5 runbooks (2 builtin + 3 custom).

**Проверено 2026-02-11:**
- `/api/investigate` с title "llm-proxy connectivity timeout" — runbook tailscale/llm-proxy автоматически подобран и загружен в instructions
- HolmesGPT выполнил 9 kubectl-команд для расследования, следуя инструкциям из runbook
- `/api/chat` — runbooks не используются (by design), но LLM самостоятельно анализирует проблему через kubectl

**Важно:** Helm chart holmes 0.6.0 **не поддерживает** `additionalVolumes`/`extraVolumes` в values. Custom runbooks монтируются через прямой kubectl patch деплоймента. ArgoCD Application настроена с `ignoreDifferences` для volumes/volumeMounts.

**Важно:** ConfigMap `custom-runbooks` хранится в `platform/holmesgpt/custom-runbooks-configmap.yaml`, но НЕ управляется ArgoCD (Application holmesgpt указывает на Helm chart). При изменении runbooks: обновить файл в git и применить вручную `kubectl apply -f platform/holmesgpt/custom-runbooks-configmap.yaml`.

**Источник:** Helm chart из `https://robusta-charts.storage.googleapis.com`, ArgoCD Application в `applications/holmesgpt.yaml`.

### 4. Open WebUI (веб-интерфейс для чата)

| Параметр | Значение |
|---|---|
| Namespace | `open-webui` |
| Chart | `open-webui` 5.18.0 |
| Image | `ghcr.io/open-webui/open-webui:0.5.16` |
| Service | `ClusterIP` порт 80 -> 8080 |
| StatefulSet | `open-webui-0` + deployment `open-webui-pipelines` |
| Storage | 5Gi (data) + 2Gi (pipelines), `openebs-hostpath` |

**Ключевые env vars:**

| Переменная | Значение |
|---|---|
| `OPENAI_API_BASE_URLS` | `http://open-webui-pipelines...svc:9099;http://llm-proxy.llm-proxy.svc.cluster.local:8080/v1` |
| `OPENAI_API_KEY` | `not-needed` |
| `VECTOR_DB` | `qdrant` |
| `QDRANT_URI` | `http://qdrant.qdrant.svc.cluster.local:6333` |
| `RAG_EMBEDDING_ENGINE` | (пусто — используется встроенный) |
| `RAG_EMBEDDING_MODEL` | `sentence-transformers/all-MiniLM-L6-v2` |
| `ENABLE_OLLAMA_API` | `False` |

**Источник:** Helm chart из `https://helm.openwebui.com`, ArgoCD Application в `applications/open-webui.yaml`.

### 5. Qdrant (векторная БД для RAG)

| Параметр | Значение |
|---|---|
| Namespace | `qdrant` |
| Chart | `qdrant` 0.10.1 |
| Service | `ClusterIP` порты 6333 (REST) + 6334 (gRPC) |
| StatefulSet | `qdrant-0` |
| Storage | 5Gi, `openebs-hostpath` |

Используется Open WebUI для RAG (Retrieval-Augmented Generation). Эмбеддинги генерируются моделью `sentence-transformers/all-MiniLM-L6-v2` (локально в open-webui, без LLM).

**Проверено 2026-02-10:** Загрузка тестового документа через Open WebUI API подтвердила полную работоспособность связки:

| Параметр | Значение |
|---|---|
| Коллекции после загрузки | 2 (`open-webui_<knowledge-id>`, `open-webui_file-<file-id>`) |
| Статус коллекций | `green` |
| Размер векторов | 384 (соответствует модели `all-MiniLM-L6-v2`) |
| Метрика расстояния | Cosine |
| Points в каждой коллекции | 1 |

Процесс работы RAG-пайплайна:
1. Файл загружается в open-webui (`POST /api/v1/files/`)
2. Создается Knowledge Base (`POST /api/v1/knowledge/create`)
3. Файл добавляется в Knowledge Base (`POST /api/v1/knowledge/{id}/file/add`)
4. Open-webui генерирует эмбеддинги моделью `sentence-transformers/all-MiniLM-L6-v2` (384-мерные вектора, локально в поде)
5. Вектора записываются в qdrant — создаются 2 коллекции (по файлу и по knowledge base)
6. При чат-запросе с включенным RAG open-webui ищет релевантные чанки в qdrant и передает их как контекст в LLM

**Источник:** Helm chart из `https://qdrant.github.io/qdrant-helm`, ArgoCD Application в `applications/qdrant.yaml`.

## Потоки данных

### Чат через Open WebUI
```
Браузер -> open-webui (:80)
  -> llm-proxy.llm-proxy.svc:8080/v1/chat/completions
    -> nginx proxy_pass -> 100.64.0.2:1234 (через tailscale0)
      -> [DERP relay / direct] -> ноутбук -> LMStudio
```

### RAG в Open WebUI
```
Браузер -> open-webui (:80)
  -> sentence-transformers (локально в поде) -> embedding
  -> qdrant.qdrant.svc:6333 (поиск по векторам)
  -> llm-proxy -> LMStudio (генерация ответа с контекстом)
```

### Расследование инцидентов через HolmesGPT
```
API-запрос -> holmesgpt-holmes.holmesgpt.svc:80/api/investigate
  -> holmesgpt выполняет kubectl (RBAC: ClusterRole с read-доступом)
  -> litellm.completion() -> OPENAI_API_BASE -> llm-proxy
    -> nginx -> tailscale -> LMStudio
  -> LLM анализирует вывод kubectl и возвращает результат
```

## GitOps-структура

```
idp-app-v1/
  applications/           # ArgoCD Application ресурсы (применяются вручную)
    headscale.yaml        # -> platform/headscale/ (git)
    llm-proxy.yaml        # -> platform/llm-proxy/ (git)
    holmesgpt.yaml        # -> Helm chart holmes 0.6.0
    open-webui.yaml       # -> Helm chart open-webui 5.18.0
    qdrant.yaml           # -> Helm chart qdrant 0.10.1
  platform/               # Raw-манифесты, отслеживаются ArgoCD
    headscale/
      configmap.yaml
      deployment.yaml
      namespace.yaml
      pvc.yaml
      service.yaml
    llm-proxy/
      configmap.yaml      # nginx.conf + LAPTOP_TAILSCALE_IP
      deployment.yaml     # nginx + tailscale-sidecar
      namespace.yaml
      rbac.yaml           # ServiceAccount + RBAC для tailscale
      secret.yaml         # TS_AUTHKEY (placeholder, заполняется вручную)
      service.yaml
```

**Важно:** Файлы в `applications/` НЕ отслеживаются никаким app-of-apps. При изменении ArgoCD Application нужно делать `kubectl apply -f`.

## Команды для проверки

### Общее состояние
```bash
export KUBECONFIG=~/proj/cross/kubeconfig_6005021

# Все поды стека
kubectl get pods -n headscale -n llm-proxy -n holmesgpt -n open-webui -n qdrant
# или по отдельности:
kubectl get pods -n headscale
kubectl get pods -n llm-proxy
kubectl get pods -n holmesgpt
kubectl get pods -n open-webui
kubectl get pods -n qdrant

# ArgoCD статус всех приложений
kubectl get application -n argocd
```

### Headscale — ноды и связность
```bash
# Список зарегистрированных нод
kubectl exec -n headscale deploy/headscale -- headscale nodes list

# Список пользователей
kubectl exec -n headscale deploy/headscale -- headscale users list

# Проверить конфигурацию
kubectl get configmap -n headscale headscale-config -o jsonpath='{.data.config\.yaml}'
```

### Tailscale sidecar в llm-proxy — VPN-туннель
```bash
# Статус tailscale (ноды и их состояние)
kubectl exec -n llm-proxy deploy/llm-proxy -c tailscale-sidecar -- tailscale status

# Ping ноутбука через tailscale
kubectl exec -n llm-proxy deploy/llm-proxy -c tailscale-sidecar -- tailscale ping 100.64.0.2

# КРИТИЧНО: проверить наличие tun-устройства (должен быть tailscale0)
kubectl exec -n llm-proxy deploy/llm-proxy -c tailscale-sidecar -- ip addr

# Проверить маршруты (должен быть маршрут через tailscale0)
kubectl exec -n llm-proxy deploy/llm-proxy -c tailscale-sidecar -- ip route

# Прямой HTTP-запрос к LMStudio через tailscale
kubectl exec -n llm-proxy deploy/llm-proxy -c tailscale-sidecar -- wget -q -O- --timeout=10 http://100.64.0.2:1234/v1/models
```

### LLM-Proxy — nginx прокси
```bash
# Проверить nginx конфигурацию
kubectl get configmap -n llm-proxy llm-proxy-config -o yaml

# Проверить доступность через nginx (изнутри пода)
kubectl exec -n llm-proxy deploy/llm-proxy -c nginx -- wget -q -O- --timeout=10 http://127.0.0.1:8080/v1/models

# Проверить доступность через сервис (из другого пода)
kubectl run test-llm --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 10 http://llm-proxy.llm-proxy.svc:8080/v1/models

# Логи nginx (ошибки upstream timed out = проблема с tailscale)
kubectl logs -n llm-proxy deploy/llm-proxy -c nginx --tail=20

# Логи tailscale sidecar
kubectl logs -n llm-proxy deploy/llm-proxy -c tailscale-sidecar --tail=20
```

### HolmesGPT — AI-анализ
```bash
# Проверить какая модель настроена
kubectl run test-model --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 10 http://holmesgpt-holmes.holmesgpt.svc:80/api/model

# Отправить тестовый чат-запрос
kubectl run test-chat --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 120 -X POST http://holmesgpt-holmes.holmesgpt.svc:80/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"ask": "What namespaces exist in this cluster?"}'

# Расследование конкретного инцидента
kubectl run test-inv --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 120 -X POST http://holmesgpt-holmes.holmesgpt.svc:80/api/investigate \
  -H 'Content-Type: application/json' \
  -d '{"source":"manual","title":"Test","description":"Check pod health in default namespace","subject":{},"context":{}}'

# Проверить env vars (ключевой: OPENAI_API_BASE)
kubectl get deployment -n holmesgpt holmesgpt-holmes \
  -o jsonpath='{range .spec.template.spec.containers[0].env[*]}{.name}={.value}{"\n"}{end}'

# Логи holmesgpt
kubectl logs -n holmesgpt deploy/holmesgpt-holmes --tail=30
```

### Open WebUI — веб-интерфейс
```bash
# Проверить env vars (ключевой: OPENAI_API_BASE_URLS)
kubectl get statefulset -n open-webui open-webui \
  -o jsonpath='{range .spec.template.spec.containers[0].env[*]}{.name}={.value}{"\n"}{end}'

# Логи (искать "Connection error" для проблем с llm-proxy)
kubectl logs -n open-webui open-webui-0 -c open-webui --tail=30

# Проверить что pipelines работают
kubectl logs -n open-webui deploy/open-webui-pipelines --tail=10
```

### Qdrant — векторная БД
```bash
# Проверить здоровье
kubectl run test-qdrant --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 5 http://qdrant.qdrant.svc:6333/healthz

# Список коллекций
kubectl run test-qdrant2 --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 5 http://qdrant.qdrant.svc:6333/collections

# Детали конкретной коллекции (подставить имя из списка)
kubectl run test-qdrant3 --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 5 http://qdrant.qdrant.svc:6333/collections/<collection-name>

# Поиск точек в коллекции (проверить что эмбеддинги записаны)
kubectl run test-qdrant4 --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 5 -X POST http://qdrant.qdrant.svc:6333/collections/<collection-name>/points/scroll \
  -H 'Content-Type: application/json' -d '{"limit": 5, "with_payload": true}'
```

### Проверка RAG-пайплайна (open-webui + qdrant)
```bash
# Получить API-токен: залогиниться в open-webui UI, открыть Settings -> Account -> API Keys

# 1. Загрузить тестовый файл
kubectl run test-upload --rm -it --restart=Never --image=curlimages/curl -- sh -c '
echo "Test document content for RAG verification." > /tmp/test.txt
curl -s --max-time 30 -X POST http://open-webui.open-webui.svc:80/api/v1/files/ \
  -H "Authorization: Bearer <TOKEN>" \
  -F "file=@/tmp/test.txt"'

# 2. Создать Knowledge Base (подставить file_id из ответа шага 1)
kubectl run test-kb --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 30 -X POST http://open-webui.open-webui.svc:80/api/v1/knowledge/create \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Test KB","description":"test"}'

# 3. Добавить файл в KB (подставить knowledge_id и file_id)
kubectl run test-add --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 60 -X POST http://open-webui.open-webui.svc:80/api/v1/knowledge/<knowledge-id>/file/add \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"file_id":"<file-id>"}'

# 4. Проверить что коллекции появились в qdrant
kubectl run test-verify --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 5 http://qdrant.qdrant.svc:6333/collections

# Ожидаемый результат: коллекции open-webui_<knowledge-id> и open-webui_file-<file-id>
# со status=green, vectors size=384 (all-MiniLM-L6-v2), points_count >= 1
```

## Инструкция: Custom Runbooks для HolmesGPT

### Что такое runbooks

Runbooks — это инструкции, которые автоматически подключаются к расследованию инцидента через `/api/investigate`. Когда HolmesGPT получает issue, `RunbookManager` сопоставляет название issue с regex-паттернами в runbooks. Если есть совпадение — инструкции из runbook передаются LLM как часть промпта.

**Важно:**
- Runbooks работают **только** с `/api/investigate` (regex-матчинг по `title`/`issue_name`)
- `/api/chat` **не** использует runbooks (свободный чат, без матчинга)
- `/api/workload_health_check` имеет баг без Robusta — не использовать

### Архитектура

```
RunbookManager
  └── load_builtin_runbooks()
        └── сканирует /app/holmes/plugins/runbooks/*.yaml
              ├── jira.yaml          (builtin — Jira issues)
              ├── general.yaml       (builtin — KubeSchedulerDown, KubeControllerManagerDown)
              └── custom-runbooks.yaml  (custom — из ConfigMap)
```

Каждый runbook содержит:
- `match.issue_name` — regex-паттерн для сопоставления с названием issue
- `match.source` — regex-паттерн для источника (опционально)
- `instructions` — текстовые инструкции для LLM

### Формат runbook

```yaml
runbooks:
  - match:
      issue_name: ".*паттерн1.*|.*паттерн2.*"   # regex для названия issue
    instructions: >
      Текст инструкции для LLM. Описать шаги диагностики,
      на что обратить внимание, какие команды выполнить.
  - match:
      issue_name: ".*другой_паттерн.*"
      source: "prometheus"                        # опционально: regex для источника
    instructions: >
      Другие инструкции.
```

### Как добавить новый runbook

**Шаг 1.** Отредактировать файл в git:

```bash
vim ~/proj/cross/idp-app-v1/platform/holmesgpt/custom-runbooks-configmap.yaml
```

Добавить новый блок в `data.custom-runbooks.yaml.runbooks`:

```yaml
      - match:
          issue_name: ".*my-new-pattern.*"
        instructions: >
          Описание шагов диагностики для этого типа инцидентов.
```

**Шаг 2.** Применить ConfigMap в кластер:

```bash
export KUBECONFIG=~/proj/cross/kubeconfig_6005021
kubectl apply -f ~/proj/cross/idp-app-v1/platform/holmesgpt/custom-runbooks-configmap.yaml
```

**Шаг 3.** Перезапустить pod holmesgpt (runbooks загружаются при старте):

```bash
kubectl rollout restart deployment holmesgpt-holmes -n holmesgpt
kubectl rollout status deployment holmesgpt-holmes -n holmesgpt
```

**Шаг 4.** Закоммитить изменения:

```bash
cd ~/proj/cross/idp-app-v1
git add platform/holmesgpt/custom-runbooks-configmap.yaml
git commit -m "update custom runbooks for HolmesGPT"
git push
```

### Как проверить что runbooks загружены

**Способ 1.** Проверить логи при старте (runbooks загружаются из yaml-файлов):

```bash
kubectl logs -n holmesgpt deploy/holmesgpt-holmes --tail=50 | grep -i runbook
```

**Способ 2.** Проверить что файл примонтирован:

```bash
kubectl exec -n holmesgpt deploy/holmesgpt-holmes -- cat /app/holmes/plugins/runbooks/custom-runbooks.yaml
```

**Способ 3.** Проверить что volume и volumeMount на месте:

```bash
# Volumes (должен быть custom-runbooks -> configmap custom-runbooks)
kubectl get deployment holmesgpt-holmes -n holmesgpt \
  -o jsonpath='{range .spec.template.spec.volumes[*]}{.name}: {.configMap.name}{"\n"}{end}'

# VolumeMounts (должен быть subPath custom-runbooks.yaml)
kubectl get deployment holmesgpt-holmes -n holmesgpt \
  -o jsonpath='{range .spec.template.spec.containers[0].volumeMounts[*]}{.name} -> {.mountPath} (subPath: {.subPath}){"\n"}{end}'
```

### Как убедиться что runbooks используются при расследовании

**Тест 1.** Отправить investigate-запрос с title, матчащим паттерн runbook:

```bash
# Port-forward (или использовать kubectl run)
kubectl port-forward -n holmesgpt svc/holmesgpt-holmes 8088:80 &

# Отправить запрос с title, матчащим ".*llm-proxy.*"
curl -s -X POST http://localhost:8088/api/investigate \
  -H "Content-Type: application/json" \
  -d '{
    "source": "manual",
    "source_instance_id": "test",
    "title": "llm-proxy upstream timeout",
    "description": "Test investigation",
    "subject": {"namespace": "llm-proxy", "kind": "deployment", "name": "llm-proxy"},
    "context": {}
  }' --max-time 300 | python3 -m json.tool
```

**Что проверять в ответе:**
- Поле `instructions` — должно содержать текст из runbook (не пустой массив)
- Поле `tool_calls` — список kubectl-команд, которые выполнил HolmesGPT
- Поле `analysis` — итоговый анализ от LLM

**Тест 2.** Через kubectl run (без port-forward):

```bash
kubectl run test-investigate --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 300 -X POST http://holmesgpt-holmes.holmesgpt.svc:80/api/investigate \
  -H 'Content-Type: application/json' \
  -d '{"source":"manual","source_instance_id":"test","title":"llm-proxy upstream timeout","description":"Test","subject":{"namespace":"llm-proxy"},"context":{}}'
```

**Тест 3.** Быстрая проверка матчинга (title НЕ матчит ни один паттерн):

```bash
curl -s -X POST http://localhost:8088/api/investigate \
  -H "Content-Type: application/json" \
  -d '{
    "source": "manual",
    "source_instance_id": "test",
    "title": "random unrelated issue",
    "description": "Test",
    "subject": {},
    "context": {}
  }' --max-time 300 | python3 -c "
import json, sys
data = json.load(sys.stdin)
instructions = data.get('instructions', [])
print('Instructions count:', len(instructions))
if instructions:
    for i, inst in enumerate(instructions):
        print(f'  [{i}]: {inst[:200]}')
else:
    print('  (empty — no runbook matched, as expected)')
"
```

### Текущие runbooks

| # | Паттерн (issue_name) | Когда срабатывает | Инструкции |
|---|---|---|---|
| 1 | `jira` (source) | Jira issues | Builtin (jira.yaml) |
| 2 | `KubeSchedulerDown\|KubeControllerManagerDown` | Prometheus alerts | Builtin (general.yaml) |
| 3 | `.*tailscale.*\|.*llm-proxy.*\|.*vpn.*` | Проблемы VPN/прокси | Проверить TS_USERSPACE, tailscale0, peer connectivity |
| 4 | `.*CrashLoopBackOff.*\|.*OOMKilled.*\|.*ImagePullBackOff.*` | Pod failures | Проверить events, ресурсы, логи, image pull secrets |
| 5 | `.*holmesgpt.*\|.*litellm.*\|.*openai.*\|.*lmstudio.*` | LLM connectivity | Проверить OPENAI_API_BASE, llm-proxy, tailscale, 403 errors |

### Рекомендации

1. **Regex-паттерны делать широкими.** Лучше перестраховаться и заматчить лишний issue, чем пропустить нужный. Инструкции не навредят, если issue не связан.

2. **Инструкции писать как чеклист.** LLM лучше следует пронумерованным шагам, чем свободному тексту.

3. **Указывать конкретные команды и значения.** Вместо "проверьте конфигурацию" писать "проверьте что TS_USERSPACE=false в deployment llm-proxy".

4. **Тестировать каждый новый runbook.** Отправить investigate-запрос с title, который должен матчить паттерн, и убедиться что `instructions` в ответе не пустой.

5. **Не дублировать builtin runbooks.** Проверить существующие паттерны перед добавлением.

6. **Помнить про перезапуск.** После `kubectl apply` ConfigMap обновляется, но HolmesGPT загружает runbooks при старте. Нужен `kubectl rollout restart`.

7. **ArgoCD не управляет ConfigMap.** Файл в git — только бэкап. Применять вручную через `kubectl apply -f`.

8. **Ограничение Helm chart 0.6.0.** Chart не поддерживает `additionalVolumes`/`extraVolumes`. Volume примонтирован через прямой kubectl patch. ArgoCD Application настроена с `ignoreDifferences` для volumes/volumeMounts чтобы не перезаписывать патч. Если ArgoCD пересоздаст deployment — патч потеряется. В этом случае повторить:

```bash
kubectl patch deployment holmesgpt-holmes -n holmesgpt --type=json -p='[
  {"op":"add","path":"/spec/template/spec/volumes/-","value":{"name":"custom-runbooks","configMap":{"name":"custom-runbooks"}}},
  {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts/-","value":{"name":"custom-runbooks","mountPath":"/app/holmes/plugins/runbooks/custom-runbooks.yaml","subPath":"custom-runbooks.yaml"}}
]'
```

## Инструкция: Переключение на другую LLM

### Текущая схема

```
HolmesGPT  ──→  llm-proxy (nginx) ──→  tailscale ──→  LMStudio (ноутбук)
Open WebUI ──→  llm-proxy (nginx) ──→  tailscale ──→  LMStudio (ноутбук)
```

llm-proxy нужен только для доступа к LLM через tailscale VPN. Если LLM доступна напрямую из кластера (интернет или внутренняя сеть) — llm-proxy можно обойти.

---

### Вариант A: Публичная LLM (OpenAI, Anthropic, Google и др.)

LLM доступна через интернет, llm-proxy **не нужен**.

```
HolmesGPT  ──→  https://api.openai.com/v1
Open WebUI ──→  https://api.openai.com/v1
```

#### A.1. HolmesGPT

Отредактировать `applications/holmesgpt.yaml`:

```yaml
    helm:
      values: |
        additionalEnvVars:
          - name: MODEL
            value: "openai/gpt-4o"                          # litellm provider/model
          - name: OPENAI_API_BASE
            value: "https://api.openai.com/v1"               # endpoint провайдера
          - name: OPENAI_API_KEY
            value: "sk-..."                                   # реальный API-ключ
```

**Формат MODEL для litellm:**

| Провайдер | MODEL | OPENAI_API_BASE |
|---|---|---|
| OpenAI | `openai/gpt-4o` | `https://api.openai.com/v1` |
| Anthropic | `anthropic/claude-sonnet-4-5-20250929` | не нужен (litellm знает endpoint) |
| Google Gemini | `gemini/gemini-2.0-flash` | не нужен |
| Azure OpenAI | `azure/my-deployment` | `https://xxx.openai.azure.com` |

> **Примечание:** Для Anthropic и Google litellm использует свои env vars:
> - Anthropic: `ANTHROPIC_API_KEY`
> - Google: `GEMINI_API_KEY` или `GOOGLE_API_KEY`

Пример для Anthropic:

```yaml
        additionalEnvVars:
          - name: MODEL
            value: "anthropic/claude-sonnet-4-5-20250929"
          - name: ANTHROPIC_API_KEY
            value: "sk-ant-..."
```

Применить:

```bash
cd ~/proj/cross/idp-app-v1
vim applications/holmesgpt.yaml                    # внести изменения
kubectl apply -f applications/holmesgpt.yaml       # применить в ArgoCD
```

Дождаться пересоздания пода:

```bash
kubectl get pods -n holmesgpt -w
```

Проверить:

```bash
kubectl run test-model --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 10 http://holmesgpt-holmes.holmesgpt.svc:80/api/model

kubectl run test-chat --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 120 -X POST http://holmesgpt-holmes.holmesgpt.svc:80/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"ask": "List all namespaces in this cluster"}'
```

#### A.2. Open WebUI

Отредактировать `applications/open-webui.yaml`:

```yaml
    helm:
      values: |
        ollama:
          enabled: false
        openaiBaseApiUrls:
          - https://api.openai.com/v1                        # endpoint провайдера
        extraEnvVars:
          - name: OPENAI_API_KEY
            value: "sk-..."                                   # реальный API-ключ
          - name: VECTOR_DB
            value: "qdrant"
          - name: QDRANT_URI
            value: "http://qdrant.qdrant.svc.cluster.local:6333"
          - name: RAG_EMBEDDING_ENGINE
            value: ""
          - name: RAG_EMBEDDING_MODEL
            value: "sentence-transformers/all-MiniLM-L6-v2"
```

Применить:

```bash
vim applications/open-webui.yaml
kubectl apply -f applications/open-webui.yaml
kubectl get pods -n open-webui -w
```

Проверить — открыть веб-интерфейс и отправить сообщение.

#### A.3. Несколько провайдеров в Open WebUI

Open WebUI поддерживает несколько LLM-эндпоинтов одновременно:

```yaml
        openaiBaseApiUrls:
          - https://api.openai.com/v1
          - https://api.anthropic.com/v1
        extraEnvVars:
          - name: OPENAI_API_KEYS
            value: "sk-openai-...;sk-ant-..."                 # через точку с запятой
```

#### A.4. Безопасное хранение API-ключей

Вместо хранения ключей в открытом виде в yaml, использовать Kubernetes Secret:

```bash
# Создать секрет
kubectl create secret generic llm-api-keys -n holmesgpt \
  --from-literal=OPENAI_API_KEY=sk-...

# В holmesgpt.yaml сослаться через valueFrom:
```

```yaml
        additionalEnvVars:
          - name: MODEL
            value: "openai/gpt-4o"
          - name: OPENAI_API_BASE
            value: "https://api.openai.com/v1"
          - name: OPENAI_API_KEY
            valueFrom:
              secretKeyRef:
                name: llm-api-keys
                key: OPENAI_API_KEY
```

> **Важно:** Добавить `ignoreDifferences` для Secret в ArgoCD Application, чтобы ArgoCD не перезаписывал секрет:
> ```yaml
>   ignoreDifferences:
>     - group: ""
>       kind: Secret
>       jsonPointers:
>         - /data
> ```

---

### Вариант B: On-prem LLM, доступная из кластера по сети

LLM развёрнута на сервере, доступном из кластера напрямую (без VPN). Например: vLLM, Ollama, text-generation-inference на внутреннем сервере.

```
HolmesGPT  ──→  http://192.168.1.100:8000/v1
Open WebUI ──→  http://192.168.1.100:8000/v1
```

#### B.1. HolmesGPT

```yaml
        additionalEnvVars:
          - name: MODEL
            value: "openai/my-model-name"                    # openai/ prefix для litellm
          - name: OPENAI_API_BASE
            value: "http://192.168.1.100:8000/v1"            # адрес LLM-сервера
          - name: OPENAI_API_KEY
            value: "not-needed"                               # или реальный ключ
```

> **Формат MODEL:** Для любой OpenAI-совместимой LLM (vLLM, Ollama, LMStudio, text-generation-inference) используйте префикс `openai/` + имя модели. litellm будет обращаться к `OPENAI_API_BASE`.

#### B.2. Open WebUI

```yaml
        openaiBaseApiUrls:
          - http://192.168.1.100:8000/v1
        extraEnvVars:
          - name: OPENAI_API_KEY
            value: "not-needed"
```

#### B.3. Проверка связности перед переключением

```bash
# Проверить что LLM доступна из кластера
kubectl run test-llm --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 10 http://192.168.1.100:8000/v1/models

# Если таймаут — LLM недоступна из кластера, нужен VPN или другой маршрут
```

---

### Вариант C: On-prem LLM через tailscale (текущая схема)

LLM на ноутбуке/сервере за NAT, доступ только через headscale/tailscale VPN.

```
HolmesGPT  ──→  llm-proxy ──→  tailscale ──→  LLM (100.64.0.x)
Open WebUI ──→  llm-proxy ──→  tailscale ──→  LLM (100.64.0.x)
```

#### C.1. Сменить целевой IP/порт LLM

Если LLM переехала на другой tailscale-хост:

```bash
# Обновить IP в ConfigMap llm-proxy
kubectl edit configmap llm-proxy-config -n llm-proxy
# Изменить LAPTOP_TAILSCALE_IP на новый IP
```

Или отредактировать в git `platform/llm-proxy/configmap.yaml` и запушить — ArgoCD подхватит.

После изменения ConfigMap — перезапустить pod:

```bash
kubectl rollout restart deployment llm-proxy -n llm-proxy
```

#### C.2. Сменить модель

Если LLM та же (LMStudio/Ollama), но загружена другая модель:

```yaml
# В applications/holmesgpt.yaml — обновить MODEL
          - name: MODEL
            value: "openai/new-model-name"
```

> LMStudio обычно игнорирует имя модели в запросе и использует загруженную. Но для корректного логирования лучше указывать актуальное имя.

#### C.3. Добавить новый tailscale-хост с LLM

1. Зарегистрировать хост в headscale:
```bash
kubectl exec -n headscale deploy/headscale -- headscale preauthkeys create --user laptop --reusable --expiration 24h
# Использовать полученный ключ для подключения tailscale на новом хосте
```

2. Проверить что хост появился:
```bash
kubectl exec -n headscale deploy/headscale -- headscale nodes list
```

3. Обновить IP в ConfigMap (см. C.1)

---

### Вариант D: Переключение между режимами (VPN ↔ Интернет)

#### D.1. С tailscale (LMStudio) на публичный API (OpenAI)

1. Обновить `applications/holmesgpt.yaml` — указать OpenAI endpoint и ключ (см. вариант A.1)
2. Обновить `applications/open-webui.yaml` — указать OpenAI endpoint (см. вариант A.2)
3. Применить оба:
```bash
kubectl apply -f applications/holmesgpt.yaml
kubectl apply -f applications/open-webui.yaml
```
4. llm-proxy можно оставить (не мешает) или удалить приложение из ArgoCD

#### D.2. С публичного API обратно на tailscale (LMStudio)

1. Убедиться что llm-proxy работает и tailscale-туннель активен:
```bash
kubectl exec -n llm-proxy deploy/llm-proxy -c tailscale-sidecar -- tailscale status
kubectl exec -n llm-proxy deploy/llm-proxy -c tailscale-sidecar -- wget -q -O- --timeout=10 http://100.64.0.2:1234/v1/models
```

2. Вернуть в `applications/holmesgpt.yaml`:
```yaml
          - name: MODEL
            value: "openai/qwen3-coder-30b-a3b-instruct-mlx"
          - name: OPENAI_API_BASE
            value: "http://llm-proxy.llm-proxy.svc.cluster.local:8080/v1"
          - name: OPENAI_API_KEY
            value: "not-needed"
```

3. Вернуть в `applications/open-webui.yaml`:
```yaml
        openaiBaseApiUrls:
          - http://llm-proxy.llm-proxy.svc.cluster.local:8080/v1
        extraEnvVars:
          - name: OPENAI_API_KEY
            value: "not-needed"
```

4. Применить:
```bash
kubectl apply -f applications/holmesgpt.yaml
kubectl apply -f applications/open-webui.yaml
```

---

### Чеклист после переключения LLM

```bash
export KUBECONFIG=~/proj/cross/kubeconfig_6005021

# 1. ArgoCD — всё в sync?
kubectl get applications -n argocd

# 2. Поды живы?
kubectl get pods -n holmesgpt
kubectl get pods -n open-webui

# 3. HolmesGPT — правильная модель?
kubectl run test-model --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 10 http://holmesgpt-holmes.holmesgpt.svc:80/api/model

# 4. HolmesGPT — тестовый запрос
kubectl run test-chat --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s --max-time 120 -X POST http://holmesgpt-holmes.holmesgpt.svc:80/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"ask": "List all namespaces"}'

# 5. Open WebUI — открыть в браузере и отправить тестовое сообщение

# 6. Env vars корректны?
kubectl get deployment -n holmesgpt holmesgpt-holmes \
  -o jsonpath='{range .spec.template.spec.containers[0].env[*]}{.name}={.value}{"\n"}{end}'

# 7. Логи без ошибок?
kubectl logs -n holmesgpt deploy/holmesgpt-holmes --tail=20
kubectl logs -n open-webui open-webui-0 -c open-webui --tail=20

# 8. Custom runbooks на месте? (после переключения holmesgpt)
kubectl exec -n holmesgpt deploy/holmesgpt-holmes -- ls /app/holmes/plugins/runbooks/
# Ожидаемый результат: custom-runbooks.yaml, general.yaml, jira.yaml
```

## Известные проблемы и решения

### 1. tailscale ping работает, но curl/wget таймаутится
**Причина:** `TS_USERSPACE=true` — нет tun-устройства, TCP-трафик не маршрутизируется через tailscale.
**Диагностика:** `ip addr` в sidecar — нет `tailscale0`.
**Решение:** `TS_USERSPACE=false` + capabilities `NET_ADMIN`, `NET_RAW`.

### 2. holmesgpt возвращает 403 "unsupported_country_region_territory"
**Причина:** litellm идет на api.openai.com вместо llm-proxy.
**Диагностика:** `OPENAI_BASE_URL` в env вместо `OPENAI_API_BASE`.
**Решение:** litellm читает `OPENAI_API_BASE`, а не `OPENAI_BASE_URL`. HolmesGPT передает `base_url=None` в litellm, поэтому env var — единственный способ задать endpoint.

### 3. kubectl exec периодически падает с EOF
**Причина:** Нестабильное соединение с API-сервером кластера.
**Решение:** Использовать `kubectl run --rm -it --image=curlimages/curl` для тестов вместо exec.

### 4. DERP relay "does not know about peer"
**Причина:** Ноутбук оффлайн в headscale или tailscale-клиент не подключен.
**Диагностика:** `headscale nodes list` — проверить статус Connected.
**Решение:** Перезапустить tailscale-клиент на ноутбуке, убедиться что `--login-server` указывает на правильный headscale URL.
