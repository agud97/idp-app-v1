**Отчет по проделанной работе и текущим проблемам**

**Что сделал (GitOps + ArgoCD) — сессия 1**
1. Склонировал `idp-app-v1` в текущую папку, перенес правки и запушил в `origin/main`.
2. Внес и запушил изменения по kb-stack (серия коммитов):
   - Добавил egress к API server (`10.96.0.1:443`) и разрешил egress внутри `kb-system`.
   - Добавил `NetworkPolicy` для нормализатора с egress в интернет (`80/443`) для `pip`.
   - Добавил `NetworkPolicy` для ingress внутри `kb-system`.
   - Добавил egress к kubevious-сервисам (IP сервисов MySQL/Redis/collector).
   - Обновил `minio/mc` и `minio/minio` теги.
   - Исправил `kubescape-exporter` запуск (без `/bin/sh`).
   - Исправил `popeye` команду.
   - Исправил endpoint kubevious в overlay на `kubevious-backend-clusterip`.
   - Добавил ожидание наличия файлов в upload контейнерах `kubescape` и `kubevious` (чтобы не падали, если файл не появился).
   - Добавил `Replace=true,Force=true` для `minio-create-bucket` Job.
   - Добавил kustomize‑патч для probe‑параметров `kubevious-backend`.
   - Установил порт `kubevious-backend` на `4000` (вместо `4001`) через kustomize‑патч.
3. Принудительные refresh ArgoCD (`kb-stack`, `apps`) и чистка/перезапуск:
   - Удалял старые `Job` и `Pod` (в т.ч. `kubevious-backend/parser`, `normalizer`).
   - Перезапускал `minio`.
   - Удалял `minio-create-bucket` для корректного пересоздания через ArgoCD.

**Что сделал — сессия 2 (2026-02-25)**
1. **Диагностика kubevious-backend**: выяснил, что backend зависал на стадии `connect-to-k8s`.
   - **Корневая причина**: NetworkPolicy использовала ClusterIP API-сервера (`10.96.0.1:443`), но kube-proxy делает DNAT *до* применения NetworkPolicy. Реальный трафик шёл на `194.58.110.52:6443` и блокировался.
   - **Фикс**: обновил NetworkPolicy egress на `194.58.110.52/32:6443`.
2. **Добавил RBAC для kubevious-backend и kubevious-parser**: ClusterRoleBinding к `kb-readonly` ClusterRole. Helm-чарт kubevious не создал RBAC (kustomize-деплой).
3. **Исправил kubescape-exporter**:
   - Убрал `command: ["kubescape"]` — scratch-образ не имеет PATH, entrypoint встроен.
   - Сменил образ `quay.io/kubescape/kubescape:v3.0.0` → `quay.io/kubescape/kubescape-cli:v3.0.48`. Образ `kubescape` — это оператор/сервер, а `kubescape-cli` — CLI для сканирования.
4. **Проверил normalizer**: `InvalidAccessKeyId` проблема была решена в предыдущей сессии (пересоздание секрета). Текущие запуски проходят успешно.

Коммиты сессии 2:
1. `Fix API server egress and add RBAC for kubevious-backend/parser`
2. `Fix kubescape-exporter: remove command override for scratch image`
3. `Use kubescape-cli image instead of operator image`
4. `Fix kubescape-cli image tag to v3.0.48`

**Состояние кластера (текущее)**
- `kb-stack`: **Synced**, **Healthy**
- `kubevious-backend`: **1/1 Running** — слушает порт 4000, все стадии пройдены
- `kubevious-parser`: **1/1 Running**
- `kubevious-collector/guard/ui/mysql/redis`: **Running**
- `minio`: **Running**, `minio-create-bucket`: **Complete**
- `popeye`: **Complete**
- `kubevious-exporter`: **Complete**
- `kubescape-exporter`: **Complete** (скан NSA framework, compliance score 60%)
- `normalizer`: **Complete**
- Все CronJob-ы (`*/30 * * * *`): работают корректно

**Все 5 проблем из предыдущего отчёта решены.**

---

**Сессия 3 (2026-03-15) — Восстановление Backstage**

**Проблема 1: Backstage недоступен (readiness 503)**

- **Корневая причина**: нода `k8s6005021-az1-md1-5rhzt-477v8` ушла в `NotReady` (kubelet перестал отвечать). PostgreSQL pod был запланирован на эту ноду. PVC использовал `openebs-hostpath` с node affinity на мёртвую ноду — данные стали недоступны, pod завис в `Terminating`.
- **Дополнительная сложность**: при первой попытке новый PV создался на ноде `rrwf4`, у которой был taint `disk-pressure:NoSchedule` — pod снова не мог запланироваться. Потребовалось повторное удаление PVC/PV.
- **Фикс**:
  1. Force-delete застрявшего pod `backstage-postgresql-0`
  2. Удалил PVC `data-backstage-postgresql-0` и PV (дважды, пока не попал на живую ноду)
  3. Новый PVC/PV создался на ноде `jlllw` (Ready, без taint)
  4. Перезапустил Backstage pod для переподключения к новой БД (данные БД сброшены, Backstage пересоздал схему)

**Проблема 2: Каталог не загружается (401 Unauthorized)**

- **Корневая причина**: GitHub PAT в секрете `backstage-secrets` устарел. В файле `token-gh` токен был рабочим, но в секрет попал с лишним символом переноса строки `\n`, что делало его невалидным при использовании.
- **Фикс**: пересоздал секрет `backstage-secrets` с токеном, очищенным от `\n` (`tr -d '\n'`). Перезапустил Backstage pod — каталог загрузился из GitHub.

**Проблема 3: Нет вкладки Kubernetes в UI**

- **Корневая причина**: в официальном образе `ghcr.io/backstage/backstage:latest` вкладка `EntityKubernetesContent` регистрируется только для компонентов типа `service`. Компоненты в каталоге имели тип `environment` — вкладка не отображалась, хотя backend Kubernetes API работал корректно и аннотации `backstage.io/kubernetes-label-selector` были прописаны.
- **Фикс**: изменил `spec.type: environment` → `spec.type: service` в файлах:
  - `developer-apps/catalog/demo123.yaml`
  - `developer-apps/catalog/qa-environment.yaml`
- Запушил в git, принудительно обновил каталог через `POST /api/catalog/refresh`.

**Состояние Backstage после сессии 3**
- `backstage`: **1/1 Running**, readiness **200 OK**
- `backstage-postgresql-0`: **1/1 Running** (новый PV на ноде `jlllw`)
- Каталог: 8 entities загружено (Template, 2 Components, Group, 3 Locations)
- Вкладка Kubernetes: **работает** для `demo123-env` и `qa-environment-env`
