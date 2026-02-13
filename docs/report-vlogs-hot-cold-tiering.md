# Отчёт: Hot/Cold Storage Tiering для VictoriaLogs

**Дата:** 2026-02-13
**Статус:** MVP реализован и проверен

## Цель

Проверить механику partition detach/move/attach в VictoriaLogs для реализации hot/cold tiering данных. В рамках MVP создан второй StorageClass (эмуляция "медленного" хранилища на том же диске), PVC для cold storage, и CronJob для автоматической миграции старых партиций.

## Предусловия

| Компонент | Версия | Примечание |
|-----------|--------|------------|
| VictoriaLogs | v1.45.0 | Поддержка partition API (`/internal/partition/list`, `detach`, `attach`) |
| Helm chart | `victoria-logs-single` v0.11.26 | Поддержка `server.extraVolumes` / `server.extraVolumeMounts` |
| OpenEBS | `localpv-provisioner` v4.1.0 | Provisioner `openebs.io/local`, поддержка нескольких hostpath StorageClass |
| ArgoCD | selfHeal: true | Изменения в `applications/` через git push |

## Что было сделано

### 1. StorageClass `openebs-hostpath-cold`

**Файл:** `platform/victorialogs/storageclass-cold.yaml`
**Применение:** `kubectl apply` (не через ArgoCD — platform-манифест)

Создан новый StorageClass с provisioner `openebs.io/local` и `BasePath: /var/openebs/cold`. OpenEBS provisioner автоматически подхватывает любой StorageClass со своим provisioner — отдельная конфигурация в chart не требуется.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-hostpath-cold
  annotations:
    cas.openebs.io/config: |
      - name: StorageType
        value: "hostpath"
      - name: BasePath
        value: "/var/openebs/cold"
    openebs.io/cas-type: local
provisioner: openebs.io/local
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

**Результат:**
```
$ kubectl get sc
NAME                         PROVISIONER        VOLUMEBINDINGMODE
openebs-hostpath (default)   openebs.io/local   WaitForFirstConsumer
openebs-hostpath-cold        openebs.io/local   WaitForFirstConsumer
```

### 2. PVC `vlogs-cold-storage`

**Файл:** `platform/victorialogs/pvc-cold.yaml`
**Применение:** `kubectl apply`

PVC на 50Gi в namespace `victorialogs`. Использует `WaitForFirstConsumer` — binding происходит только при подключении к pod'у, что гарантирует размещение на той же ноде, где запущен VictoriaLogs.

**Результат:**
```
$ kubectl get pvc -n victorialogs
NAME                                                        STATUS   STORAGECLASS
server-volume-victoria-logs-victoria-logs-single-server-0   Bound    openebs-hostpath
vlogs-cold-storage                                          Bound    openebs-hostpath-cold
```

### 3. Монтирование cold volume в pod VictoriaLogs

**Файл:** `applications/victoria-logs.yaml`
**Применение:** git push → ArgoCD sync

Добавлены `extraVolumes` и `extraVolumeMounts` в helm values:

```yaml
server:
  extraVolumes:
    - name: cold-storage
      persistentVolumeClaim:
        claimName: vlogs-cold-storage
  extraVolumeMounts:
    - name: cold-storage
      mountPath: /vlogs-cold
```

**Результат:** Pod пересоздан ArgoCD'ом с двумя mount'ами:
- `/storage` — hot (основной PVC, `openebs-hostpath`)
- `/vlogs-cold` — cold (новый PVC, `openebs-hostpath-cold`)

```
$ kubectl exec -n victorialogs victoria-logs-victoria-logs-single-server-0 -- mount | grep -E "/storage|/vlogs-cold"
/dev/sda4 on /storage type xfs (rw,...)
/dev/sda4 on /vlogs-cold type xfs (rw,...)
```

> Оба volume на одном физическом диске (`/dev/sda4`) — это ожидаемое поведение для MVP-эмуляции. В production cold storage будет на отдельном диске/NFS.

### 4. CronJob для миграции партиций

**Файл:** `platform/victorialogs/partition-mover-cronjob.yaml`
**Применение:** `kubectl apply`

CronJob `vlogs-partition-mover` запускается ежедневно в 02:00 UTC. Использует `bitnami/kubectl` для выполнения `kubectl exec` в pod VictoriaLogs.

**Алгоритм для каждой партиции (кроме сегодняшней):**

1. Проверить, не является ли уже symlink'ом (skip если да)
2. `POST /internal/partition/detach?name=YYYYMMDD` — отключить партицию
3. `mv /storage/partitions/YYYYMMDD /vlogs-cold/YYYYMMDD` — переместить данные
4. `ln -s /vlogs-cold/YYYYMMDD /storage/partitions/YYYYMMDD` — создать symlink
5. `POST /internal/partition/attach?name=YYYYMMDD` — переподключить партицию

**Обработка ошибок:** при неудачном move выполняется повторный attach оригинальной партиции.

**RBAC:** ServiceAccount `vlogs-partition-mover` с правами `pods/exec` в namespace `victorialogs`.

## Верификация

### Тест 1: Partition API

Проверена работоспособность всех endpoint'ов partition API изнутри pod'а:

```
$ kubectl exec -n victorialogs victoria-logs-victoria-logs-single-server-0 -- \
    wget -qO- http://127.0.0.1:9428/internal/partition/list
["20260205","20260206","20260207","20260208","20260209","20260210","20260211","20260212","20260213"]
```

> **Замечание:** `localhost` не работает (connection refused), необходимо использовать `127.0.0.1`. Это связано с особенностями DNS-резолва в минималистичном контейнере VictoriaLogs.

### Тест 2: Ручная миграция одной партиции

Выполнена полная цепочка detach → mv → symlink → attach для партиции `20260205`:

```
# Detach
$ wget --post-data='' 'http://127.0.0.1:9428/internal/partition/detach?name=20260205'
HTTP/1.1 200 OK

# Move + symlink
$ mv /storage/partitions/20260205 /vlogs-cold/20260205
$ ln -s /vlogs-cold/20260205 /storage/partitions/20260205

# Verify symlink
$ ls -la /storage/partitions/20260205
lrwxrwxrwx  20260205 -> /vlogs-cold/20260205

# Re-attach
$ wget --post-data='' 'http://127.0.0.1:9428/internal/partition/attach?name=20260205'
HTTP/1.1 200 OK

# Partition list — 20260205 снова доступна
["20260205","20260206","20260207","20260208","20260209","20260210","20260211","20260212","20260213"]
```

**Вывод:** VictoriaLogs корректно читает данные через symlink'и. Partition API (detach/attach) работает штатно.

### Тест 3: Запуск CronJob вручную

Создан тестовый job из CronJob'а:

```
$ kubectl create job -n victorialogs vlogs-partition-mover-test2 --from=cronjob/vlogs-partition-mover
```

**Логи выполнения:**
```
=== VictoriaLogs Partition Mover ===
Today: 20260213
Current partitions: ["20260205","20260206","20260207","20260208","20260209","20260210","20260211","20260212","20260213"]
SKIP 20260205 (already on cold storage)
MOVING 20260206 to cold storage...
  Detached 20260206
  Moved to /vlogs-cold/20260206, symlink created
  Re-attached 20260206
MOVING 20260207 to cold storage...
  Detached 20260207
  Moved to /vlogs-cold/20260207, symlink created
  Re-attached 20260207
MOVING 20260208 to cold storage...
  Detached 20260208
  Moved to /vlogs-cold/20260208, symlink created
  Re-attached 20260208
MOVING 20260209 to cold storage...
  Detached 20260209
  Moved to /vlogs-cold/20260209, symlink created
  Re-attached 20260209
MOVING 20260210 to cold storage...
  Detached 20260210
  Moved to /vlogs-cold/20260210, symlink created
  Re-attached 20260210
MOVING 20260211 to cold storage...
  Detached 20260211
  Moved to /vlogs-cold/20260211, symlink created
  Re-attached 20260211
MOVING 20260212 to cold storage...
  Detached 20260212
  Moved to /vlogs-cold/20260212, symlink created
  Re-attached 20260212
SKIP 20260213 (today)

=== Summary ===
Moved: 7 | Skipped: 2 | Errors: 0
Final partitions: ["20260205","20260206","20260207","20260208","20260209","20260210","20260211","20260212","20260213"]
```

**Результат:**
- 7 партиций перемещены на cold storage
- 1 пропущена (20260205 — уже на cold после ручного теста)
- 1 пропущена (20260213 — сегодняшний день)
- 0 ошибок
- Все 9 партиций доступны в финальном списке

### Тест 4: Запрос данных с cold storage

Выполнен LogsQL-запрос к данным из партиции 20260210 (находится на cold storage через symlink):

```
$ wget -qO- 'http://127.0.0.1:9428/select/logsql/query?query=*&limit=3&start=2026-02-10T00:00:00Z&end=2026-02-10T23:59:59Z'
```

Результат: 3 записи успешно возвращены (логи crossplane из `crossplane-system` namespace). Данные полностью доступны через symlink.

### Тест 5: Финальное состояние файловой системы

```
$ ls -la /storage/partitions/
lrwxrwxrwx  20260205 -> /vlogs-cold/20260205
lrwxrwxrwx  20260206 -> /vlogs-cold/20260206
lrwxrwxrwx  20260207 -> /vlogs-cold/20260207
lrwxrwxrwx  20260208 -> /vlogs-cold/20260208
lrwxrwxrwx  20260209 -> /vlogs-cold/20260209
lrwxrwxrwx  20260210 -> /vlogs-cold/20260210
lrwxrwxrwx  20260211 -> /vlogs-cold/20260211
lrwxrwxrwx  20260212 -> /vlogs-cold/20260212
drwxrwsr-x  20260213  (real directory — today's hot partition)

$ ls /vlogs-cold/
20260205  20260206  20260207  20260208  20260209  20260210  20260211  20260212
```

## Архитектура (итог)

```
┌─────────────────────────────────────────────────────────┐
│  VictoriaLogs Pod                                       │
│                                                         │
│  /storage/partitions/                                   │
│  ├── 20260205 → /vlogs-cold/20260205  (symlink)        │
│  ├── 20260206 → /vlogs-cold/20260206  (symlink)        │
│  ├── ...                                                │
│  ├── 20260212 → /vlogs-cold/20260212  (symlink)        │
│  └── 20260213/  (real dir — today's hot partition)      │
│                                                         │
│  /vlogs-cold/              ← PVC: openebs-hostpath-cold │
│  ├── 20260205/                                          │
│  ├── 20260206/                                          │
│  └── ...                                                │
│                                                         │
│  Partition API (localhost:9428)                          │
│  ├── /internal/partition/list                            │
│  ├── /internal/partition/detach?name=YYYYMMDD            │
│  └── /internal/partition/attach?name=YYYYMMDD            │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  CronJob: vlogs-partition-mover (daily 02:00 UTC)       │
│  Image: bitnami/kubectl:latest                          │
│  Method: kubectl exec → detach → mv → ln -s → attach   │
│  RBAC: pods/exec in victorialogs namespace              │
└─────────────────────────────────────────────────────────┘
```

## Файлы

| Файл | Действие | Применение |
|------|----------|------------|
| `platform/victorialogs/storageclass-cold.yaml` | Создан | `kubectl apply` |
| `platform/victorialogs/pvc-cold.yaml` | Создан | `kubectl apply` |
| `applications/victoria-logs.yaml` | Изменён (`extraVolumes`) | git push → ArgoCD |
| `platform/victorialogs/partition-mover-cronjob.yaml` | Создан | `kubectl apply` |

## Ключевые находки

1. **Partition API работает.** Endpoint'ы `detach`/`attach` принимают параметр `name=YYYYMMDD` через POST. Partition list возвращает JSON-массив.

2. **Symlink'и поддерживаются.** VictoriaLogs корректно читает данные партиций через symbolic links. После attach партиции через symlink данные доступны для LogsQL-запросов.

3. **OpenEBS поддерживает несколько StorageClass.** Достаточно создать новый SC с тем же provisioner'ом (`openebs.io/local`) и другим `BasePath`. OpenEBS dynamic provisioner подхватывает его автоматически.

4. **`WaitForFirstConsumer` обеспечивает co-location.** Оба PVC (hot и cold) автоматически привязались к одной ноде при подключении к pod'у.

5. **`localhost` vs `127.0.0.1`.** В контейнере VictoriaLogs `localhost` не резолвится — нужно использовать `127.0.0.1` для обращения к partition API.

## Риски и ограничения

| Риск | Статус | Комментарий |
|------|--------|-------------|
| Symlink'и не поддерживаются VictoriaLogs | **Опровергнут** | Работает штатно |
| PVC на разных нодах | **Митигирован** | `WaitForFirstConsumer` гарантирует co-location |
| Потеря symlink'ов при рестарте pod'а | **Низкий риск** | Symlink'и хранятся на PV, не в ephemeral storage |
| Auto-attach при старте VictoriaLogs | **Требует проверки** | При рестарте pod'а VictoriaLogs должен автоматически подхватить партиции из `/storage/partitions/` (включая symlink'и). Если нет — потребуется init-container для повторного attach |
| Реальная разница в скорости | **Не применимо** | MVP на одном диске — разница в latency отсутствует |

## Следующие шаги (за рамками MVP)

1. **Проверка поведения при рестарте pod'а** — убедиться что VictoriaLogs при старте автоматически attach'ит партиции через symlink'и
2. **Реальный cold storage** — NFS, S3-backed volume, или отдельный медленный диск
3. **Мониторинг** — алерт при ошибках CronJob'а, метрики размера hot/cold storage
4. **Настройка retention** — разные retention period для hot (7 дней) и cold (90 дней)
