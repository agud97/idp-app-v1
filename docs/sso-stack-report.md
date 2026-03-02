# SSO Stack: OpenLDAP + Authentik + OpenWebUI — Отчёт о деплое

**Дата**: 2026-03-02
**Кластер**: k8s6005021
**Коммиты**: `03c5700` → `e63514b`

---

## Итоговая архитектура

```
Браузер (localhost:8080)
    │
    ▼ port-forward
Open WebUI pod (open-webui ns)
    │  при логине: redirect OAuth2
    ▼
Authentik Server (authentik ns)  ←──── LDAP sync ────►  OpenLDAP (openldap ns)
    │  OIDC token exchange                               dc=runityai,dc=local
    ▼                                                    ou=people: admin1, user1
Open WebUI pod                                           ou=groups: prod_runityai_admin,
    (получает groups claim)                                         prod_runityai_user
```

### Компоненты

| Компонент | Namespace | Chart / Image | Версия |
|-----------|-----------|---------------|--------|
| OpenLDAP | `openldap` | jp-gouin/openldap | 2.0.4 |
| Authentik Server | `authentik` | goauthentik.io/authentik | 2025.2.1 |
| Authentik Worker | `authentik` | — (same) | 2025.2.1 |
| PostgreSQL | `authentik` | bitnamilegacy/postgresql | 17.6.0-debian-12-r4 |
| Redis | `authentik` | bitnamilegacy/redis | 8.2.1-debian-12-r0 |
| Open WebUI | `open-webui` | open-webui chart | 5.18.0 |

### Файлы GitOps (idp-app-v1)

```
applications/
  openldap.yaml          # ArgoCD App, Helm chart jp-gouin/openldap
  authentik.yaml         # ArgoCD App, kustomize --enable-helm
  open-webui.yaml        # ArgoCD App (изменён: добавлены OIDC env vars)

platform/authentik/
  kustomization.yaml     # helmCharts: authentik 2025.2.1 + resources
  helm-values.yaml       # values: pg, redis, bootstrap, volumes
  blueprints-configmap.yaml  # ConfigMap authentik-blueprints (ldap-source + oidc-provider)
```

### LDAP структура (dc=runityai,dc=local)

```
dc=runityai,dc=local
├── ou=people
│   ├── uid=admin1  (inetOrgPerson, posixAccount)  → mail: admin1@runityai.local
│   └── uid=user1   (inetOrgPerson, posixAccount)  → mail: user1@runityai.local
└── ou=groups
    ├── cn=prod_runityai_admin  (posixGroup, uidObject)  → memberUid: admin1
    └── cn=prod_runityai_user   (posixGroup, uidObject)  → memberUid: user1
```

### Authentik blueprints (автоматически при старте)

1. **ldap-source.yaml** — создаёт LDAP Source `openldap` с user/group sync
2. **openwebui-provider.yaml** — создаёт:
   - ScopeMapping `openwebui-groups` (возвращает список групп пользователя)
   - OAuth2Provider `openwebui` (client_id=openwebui, confidential)
   - Application `openwebui` → провайдер openwebui

### OpenWebUI SSO env vars

```yaml
ENABLE_OAUTH_SIGNUP: "true"
OAUTH_PROVIDER_NAME: "Authentik"
OPENID_PROVIDER_URL: "http://authentik-server.authentik.svc.cluster.local:9000/application/o/openwebui/"
OAUTH_CLIENT_ID: "openwebui"
OAUTH_CLIENT_SECRET: valueFrom secret open-webui-sso
OAUTH_ROLES_CLAIM: "groups"
OAUTH_ADMIN_ROLES: "prod_runityai_admin"
OAUTH_ALLOWED_ROLES: "prod_runityai_admin,prod_runityai_user"
OAUTH_MERGE_ACCOUNTS_BY_EMAIL: "true"
```

---

## Проблемы и решения

### 1. Bitnami Helm repo вернул 403

**Ошибка ArgoCD:**
```
error fetching chart: failed to fetch chart:
`helm pull --repo https://charts.bitnami.com/bitnami openldap`
403 Forbidden
```

**Причина:** Bitnami перевёл Helm charts в OCI-реестр (registry-1.docker.io/bitnamicharts). Старый HTTP-репозиторий закрыт.

**Решение:** Заменил `bitnami/openldap` на `jp-gouin/openldap` из `https://jp-gouin.github.io/helm-openldap/`.

```diff
- repoURL: https://charts.bitnami.com/bitnami
- chart: openldap
- targetRevision: "0.4.6"
+ repoURL: https://jp-gouin.github.io/helm-openldap/
+ chart: openldap
+ targetRevision: "2.0.4"
```

---

### 2. Authentik chart: deprecated `service:` field

**Ошибка:**
```
execution error at (authentik/templates/deprectations.yaml:74:3):
`service` is deprecated. See the release notes for a list of changes.
```

**Причина:** В Authentik chart 2025.2.x поле `service.servicePort` удалено на верхнем уровне. По умолчанию уже 9000.

**Решение:** Удалил секцию `service:` из `helm-values.yaml`.

```diff
- service:
-   servicePort: 9000
```

---

### 3. jp-gouin/openldap subcharts с deprecated Ingress API

**Ошибка ArgoCD sync:**
```
The Kubernetes API could not find version "v1beta1" of extensions/Ingress
for requested resource openldap/openldap-ltb-passwd.
```

**Причина:** Subchart `ltb-passwd` и `phpldapadmin` используют `extensions/v1beta1` Ingress, удалённый в Kubernetes 1.22+. Кластер на 1.34.

**Решение:** Отключить оба subcharts в values:

```yaml
ltb-passwd:
  enabled: false
phpldapadmin:
  enabled: false
```

---

### 4. bitnami/postgresql и bitnami/redis images not found

**Ошибка:**
```
Failed to pull image "docker.io/bitnami/postgresql:15.8.0-debian-12-r18":
not found
Failed to pull image "docker.io/bitnami/redis:7.4.1-debian-12-r0":
not found
```

**Причина:** Bitnami удалил образы из публичного docker.io. Тот же паттерн что с Backstage postgresql.

**Решение:** Заменить на `bitnamilegacy` образы в helm-values.yaml:

```yaml
postgresql:
  image:
    registry: docker.io
    repository: bitnamilegacy/postgresql
    tag: "17.6.0-debian-12-r4"
redis:
  image:
    registry: docker.io
    repository: bitnamilegacy/redis
    tag: "8.2.1-debian-12-r0"
```

Для применения изменений потребовалось принудительно удалить старые поды (StatefulSets не обновились автоматически из-за CrashLoopBackOff):

```bash
kubectl --kubeconfig ~/proj/cross/kubeconfig_6005021 delete pod \
  authentik-postgresql-0 authentik-redis-master-0 -n authentik
```

---

### 5. ArgoCD кеш: не подхватывает изменения

**Симптом:** После `git push` ArgoCD продолжал показывать старую ошибку (bitnami 403, deprecated service).

**Причина:** ArgoCD кешировал манифесты источника.

**Решение:** Force hard refresh через аннотацию:

```bash
kubectl --kubeconfig ~/proj/cross/kubeconfig_6005021 -n argocd patch application openldap \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
kubectl --kubeconfig ~/proj/cross/kubeconfig_6005021 -n argocd patch application authentik \
  --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

---

### 6. Blueprint `OpenLDAP Source`: user_property_mappings пусто

**Ошибка при apply blueprint:**
```
Entry invalid: Serializer errors {
  'user_property_mappings': ["When 'Sync users' is enabled,
  'users property mappings' cannot be empty."]
}
```

**Причина:** В Authentik 2025.x `user_property_mappings` и `group_property_mappings` обязательны при включённом sync. В первоначальном blueprint они не были указаны.

**Диагностика:**
```bash
kubectl exec -n authentik authentik-worker-78fc549b4f-4rxhc -- \
  ak shell -c "
from authentik.blueprints.v1.importer import Importer
content = open('/blueprints/custom/ldap-source.yaml').read()
importer = Importer.from_string(content)
valid, logs = importer.validate()
for log in logs: print(log.event[:300])
"
```

**Доступные LDAP mappings:**
```bash
kubectl exec -n authentik authentik-worker-78fc549b4f-4rxhc -- \
  ak shell -c "
from authentik.sources.ldap.models import LDAPSourcePropertyMapping
for m in LDAPSourcePropertyMapping.objects.all().order_by('name'):
    print(m.name)
"
# authentik default OpenLDAP Mapping: uid
# authentik default OpenLDAP Mapping: cn
# authentik default LDAP Mapping: mail
# authentik default LDAP Mapping: Name
# authentik default LDAP Mapping: DN to User Path
```

**Решение:** Добавить `user_property_mappings` и `group_property_mappings` в blueprint через `!Find`:

```yaml
user_property_mappings:
  - !Find [authentik_sources_ldap.ldapsourcepropertymapping, [name, "authentik default OpenLDAP Mapping: uid"]]
  - !Find [authentik_sources_ldap.ldapsourcepropertymapping, [name, "authentik default OpenLDAP Mapping: cn"]]
  - !Find [authentik_sources_ldap.ldapsourcepropertymapping, [name, "authentik default LDAP Mapping: mail"]]
  - !Find [authentik_sources_ldap.ldapsourcepropertymapping, [name, "authentik default LDAP Mapping: Name"]]
  - !Find [authentik_sources_ldap.ldapsourcepropertymapping, [name, "authentik default LDAP Mapping: DN to User Path"]]
group_property_mappings:
  - !Find [authentik_sources_ldap.ldapsourcepropertymapping, [name, "authentik default OpenLDAP Mapping: cn"]]
```

---

### 7. Blueprint `OpenWebUI OIDC`: redirect_uris формат и invalidation_flow

**Ошибка:**
```
'invalidation_flow': ['This field is required.']
'redirect_uris': {'non_field_errors': ['Expected a list of items but got type "str".']}
```

**Причина:** В Authentik 2025.2.x:
- `redirect_uris` изменился с простой строки на список объектов `{url, matching_mode}`
- `invalidation_flow` стало обязательным полем OAuth2Provider

**Доступные flows:**
```bash
kubectl exec -n authentik authentik-worker-78fc549b4f-4rxhc -- \
  ak shell -c "
from authentik.flows.models import Flow
for f in Flow.objects.filter(designation='invalidation'):
    print(f.slug, f.name)
# default-provider-invalidation-flow  Logged out of application
"
```

**Решение:**
```yaml
# было:
redirect_uris: "http://localhost:8080/oauth/oidc/callback"

# стало:
redirect_uris:
  - url: "http://localhost:8080/oauth/oidc/callback"
    matching_mode: strict
invalidation_flow: !Find [authentik_flows.flow, [slug, default-provider-invalidation-flow]]
```

---

### 8. ConfigMap mount: задержка sync

**Симптом:** После `kubectl apply -f blueprints-configmap.yaml` содержимое файлов в поде обновлялось не сразу.

**Причина:** Kubernetes ConfigMap volumes обновляются с задержкой (kubelet sync period, по умолчанию ~1 минута).

**Паттерн ожидания:**
```bash
# После apply ConfigMap — ждём sync
sleep 25
kubectl exec -n authentik authentik-worker-78fc549b4f-4rxhc -- \
  cat /blueprints/custom/ldap-source.yaml | grep "user_property_mappings"
# если нет — ждём ещё
```

**Альтернатива:** Применять изменения напрямую через `ak shell`, не дожидаясь sync:
```bash
kubectl exec -n authentik authentik-worker-78fc549b4f-4rxhc -- \
  ak shell -c "
from authentik.blueprints.v1.importer import Importer
importer = Importer.from_string(open('/blueprints/custom/ldap-source.yaml').read())
valid, _ = importer.validate()
if valid: importer.apply()
"
```

---

### 9. LDAP Group Sync: posixGroup не имеет `uid` атрибута

**Симптом:** После sync — users появились, groups появились, но memberships пустые.

**Причина:** В Authentik `object_uniqueness_field: uid` используется для двух целей:
1. Как уникальный ключ объекта при синхронизации
2. Как поле для сопоставления `memberUid` с пользователями (хранится в `attributes.ldap_uniq`)

`posixGroup` objectClass не включает атрибут `uid` → GroupLDAPSynchronizer пропускал группы:
```python
if not attributes.get(self._source.object_uniqueness_field):
    self.message("Uniqueness field not found/not set in attributes: ...")
    continue  # группа пропускается
```

**Диагностика:**
```bash
# Прямой запуск синхронизатора групп
kubectl exec -n authentik authentik-worker-78fc549b4f-4rxhc -- \
  ak shell -c "
from authentik.sources.ldap.sync.groups import GroupLDAPSynchronizer
from authentik.sources.ldap.models import LDAPSource
source = LDAPSource.objects.get(slug='openldap')
syncer = GroupLDAPSynchronizer(source)
pages = list(syncer.get_objects())
print('Pages:', len(pages), 'items:', len(pages[0]))
cnt = syncer.sync(pages[0])
print('Synced:', cnt)  # → 0
"
```

**Попытка 1 (неверная):** Изменить `object_uniqueness_field: uid` → `entryUUID`.
_Результат:_ Группы начали синхронизироваться, но memberships перестали работать — `memberUid: admin1` не совпадал с `ldap_uniq = <UUID>`.

**Решение (правильное):** Добавить `uid` к группам в LDAP через вспомогательный objectClass `uidObject`:

```bash
# Добавление uid к группам через ldapmodify (live fix на работающем поде)
kubectl exec -n openldap openldap-0 -- \
  bash -c "ldapmodify -x -H ldap://localhost:389 \
    -D 'cn=admin,dc=runityai,dc=local' -w LdapAdmin1234 <<'LDIF'
dn: cn=prod_runityai_admin,ou=groups,dc=runityai,dc=local
changetype: modify
add: objectClass
objectClass: uidObject
-
add: uid
uid: prod_runityai_admin

dn: cn=prod_runityai_user,ou=groups,dc=runityai,dc=local
changetype: modify
add: objectClass
objectClass: uidObject
-
add: uid
uid: prod_runityai_user
LDIF"
```

Также обновлён LDIF в chart values (для будущих pod restarts):
```yaml
dn: cn=prod_runityai_admin,ou=groups,dc=runityai,dc=local
objectClass: posixGroup
objectClass: uidObject   # <- добавлено
cn: prod_runityai_admin
uid: prod_runityai_admin # <- добавлено
```

**Финальный sync через ak shell:**
```bash
kubectl exec -n authentik authentik-worker-78fc549b4f-4rxhc -- \
  ak shell -c "
from authentik.sources.ldap.models import LDAPSource
from authentik.sources.ldap.sync.users import UserLDAPSynchronizer
from authentik.sources.ldap.sync.groups import GroupLDAPSynchronizer
from authentik.sources.ldap.sync.membership import MembershipLDAPSynchronizer

source = LDAPSource.objects.get(slug='openldap')
u_count = sum(UserLDAPSynchronizer(source).sync(p) for p in UserLDAPSynchronizer(source).get_objects())
g_count = sum(GroupLDAPSynchronizer(source).sync(p) for p in GroupLDAPSynchronizer(source).get_objects())
m = MembershipLDAPSynchronizer(source)
m_count = sum(m.sync(p) for p in m.get_objects())
print(f'Users: {u_count}, Groups: {g_count}, Memberships: {m_count}')
# → Users: 2, Groups: 2, Memberships: 4
"
```

---

### 10. Дублирование групп при смене uniqueness_field

**Симптом:** После возврата с `entryUUID` обратно на `uid` в БД оказалось 4 группы (по 2 дубликата).

**Причина:** При каждом изменении `object_uniqueness_field` Authentik создаёт новые группы с новым `ldap_uniq` ключом, не удаляя старые.

**Диагностика:**
```bash
kubectl exec -n authentik authentik-worker-78fc549b4f-4rxhc -- \
  ak shell -c "
from authentik.core.models import Group
for g in Group.objects.filter(name__startswith='prod_runityai'):
    print(g.pk, g.name, g.attributes.get('ldap_uniq'), list(g.users.values_list('username', flat=True)))
"
# 5ead4c72 prod_runityai_admin 4433a640-aac0-... []        <- entryUUID, пустая
# 80e675fa prod_runityai_admin prod_runityai_admin ['admin1'] <- uid, правильная
```

**Решение:** Удалить группы с UUID-based `ldap_uniq` и без членов:
```bash
kubectl exec -n authentik authentik-worker-78fc549b4f-4rxhc -- \
  ak shell -c "
import re
from authentik.core.models import Group
uuid_re = re.compile(r'^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$')
for g in Group.objects.filter(name__startswith='prod_runityai'):
    ldap_uniq = g.attributes.get('ldap_uniq', '')
    if not g.users.exists() and uuid_re.match(str(ldap_uniq)):
        g.delete()
        print(f'Deleted: {g.name} ({ldap_uniq})')
"
```

---

## Итоговый статус

```
ArgoCD:
  openldap:  Synced ✅  Healthy ✅
  authentik: Synced ✅  Healthy ✅
  open-webui: Synced ✅ Healthy ✅

Authentik:
  LDAP Source openldap: server=ldap://openldap.openldap.svc.cluster.local:389
  OAuth2 Provider openwebui: client_id=openwebui
  Application OpenWebUI: slug=openwebui

LDAP Sync:
  Users: admin1 (groups=[prod_runityai_admin])
         user1  (groups=[prod_runityai_user])
  Groups: prod_runityai_admin (members=[admin1])
          prod_runityai_user  (members=[user1])

Secrets:
  open-webui-sso (namespace open-webui) → OAUTH_CLIENT_SECRET
```

## Верификация SSO (port-forward)

```bash
# 1. /etc/hosts (локально, один раз)
echo "127.0.0.1 authentik-server.authentik.svc.cluster.local" >> /etc/hosts

# 2. Port-forwards (два терминала)
kubectl --kubeconfig ~/proj/cross/kubeconfig_6005021 \
  port-forward svc/authentik-server -n authentik 9000:80

kubectl --kubeconfig ~/proj/cross/kubeconfig_6005021 \
  port-forward svc/open-webui -n open-webui 8080:80

# 3. Проверка
# Authentik admin: http://authentik-server.authentik.svc.cluster.local:9000
#   логин: akadmin / AuthentikAdmin123!
#   Directory → Sources → OpenLDAP → Run sync
#   Applications → Applications → OpenWebUI ✓

# OpenWebUI: http://localhost:8080
#   кнопка "Sign in with Authentik" ✓
#   admin1 / Admin1Pass! → роль admin
#   user1  / User1Pass!  → роль user
```
