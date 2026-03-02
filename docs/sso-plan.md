# SSO для OpenWebUI: OpenLDAP + Authentik + GitOps

## Context
OpenWebUI работает в namespace `open-webui`, доступен только через port-forward (ClusterIP).
Нужно: развернуть OpenLDAP (пользователи + группы) → Authentik (OIDC IdP) → настроить
OpenWebUI на SSO. Всё через GitOps/ArgoCD. Доступ — port-forward локально.

## Архитектура
```
Браузер → OpenWebUI (localhost:8080) → redirect → Authentik (localhost:9000)
                ↑ OIDC token exchange ↑                    ↑ LDAP sync ↑
         OpenWebUI pod (in-cluster)          OpenLDAP (openldap.openldap.svc:389)
```

**Ключевое решение для port-forward**: `AUTHENTIK_URL = http://authentik-server.authentik.svc.cluster.local:9000`
- Из pod: CoreDNS резолвит → ClusterIP → container:9000 ✓
- Из браузера: `/etc/hosts` `127.0.0.1 authentik-server.authentik.svc.cluster.local` + `port-forward 9000:9000` ✓

---

## Файлы для создания/изменения

### 1. `applications/openldap.yaml` — НОВЫЙ
ArgoCD Application, Helm chart `bitnami/openldap` v0.4.x, namespace `openldap`.

Ключевые Helm values:
```yaml
global:
  ldapDomain: "runityai.local"          # base DN: dc=runityai,dc=local
auth:
  adminPassword: "LdapAdmin1234"        # CHANGE в проде
  adminUsername: "admin"
  usernames: ["admin1", "user1"]
  passwords: ["Admin1Pass!", "User1Pass!"]
customLdifFiles:
  01-groups.ldif: |
    dn: ou=groups,dc=runityai,dc=local
    objectClass: organizationalUnit
    ou: groups

    dn: cn=prod_runityai_admin,ou=groups,dc=runityai,dc=local
    objectClass: posixGroup
    cn: prod_runityai_admin
    gidNumber: 2000
    memberUid: admin1

    dn: cn=prod_runityai_user,ou=groups,dc=runityai,dc=local
    objectClass: posixGroup
    cn: prod_runityai_user
    gidNumber: 2001
    memberUid: user1
persistence:
  enabled: true
  size: 1Gi
  storageClass: openebs-hostpath
```
Примечание: `posixGroup` + `memberUid` (UID-строка, не DN) — обходит проблему порядка
инициализации (bitnami создаёт пользователей ПОСЛЕ custom LDIF).

### 2. `platform/authentik/kustomization.yaml` — НОВЫЙ
Kustomize с helm (паттерн как в kb-stack):
```yaml
helmCharts:
  - name: authentik
    repo: https://charts.goauthentik.io
    version: "2025.2.1"
    releaseName: authentik
    namespace: authentik
    valuesFile: helm-values.yaml

resources:
  - blueprints-configmap.yaml
```

### 3. `platform/authentik/helm-values.yaml` — НОВЫЙ
```yaml
authentik:
  secret_key: "authentik-secret-key-change-me"
  postgresql:
    password: "authentik-db-pass-change-me"

postgresql:
  enabled: true
  auth:
    username: "authentik"
    password: "authentik-db-pass-change-me"
    database: "authentik"
  primary:
    persistence:
      size: 5Gi
      storageClass: openebs-hostpath

redis:
  enabled: true
  auth:
    enabled: false
  master:
    persistence:
      size: 1Gi
      storageClass: openebs-hostpath

server:
  env:
    - name: AUTHENTIK_BOOTSTRAP_PASSWORD
      value: "AuthentikAdmin123!"    # первичный пароль admin
    - name: AUTHENTIK_BOOTSTRAP_EMAIL
      value: "admin@runityai.local"
    - name: AUTHENTIK_URL
      value: "http://authentik-server.authentik.svc.cluster.local:9000"
  volumes:
    - name: blueprints-custom
      configMap:
        name: authentik-blueprints
  volumeMounts:
    - name: blueprints-custom
      mountPath: /blueprints/custom

worker:
  volumes:
    - name: blueprints-custom
      configMap:
        name: authentik-blueprints
  volumeMounts:
    - name: blueprints-custom
      mountPath: /blueprints/custom

service:
  servicePort: 9000    # container port = service port (для консистентности URLs)

blueprints:
  configMaps: []       # управляем монтированием вручную через volumes выше
```

### 4. `platform/authentik/blueprints-configmap.yaml` — НОВЫЙ
ConfigMap `authentik-blueprints` в namespace `authentik`, с двумя файлами:

**`ldap-source.yaml`** — LDAP источник:
```yaml
version: 1
metadata:
  name: OpenLDAP Source
  labels:
    blueprints.goauthentik.io/instantiate: "true"
entries:
  - model: authentik_sources_ldap.ldapsource
    state: present
    identifiers:
      slug: openldap
    attrs:
      name: OpenLDAP
      server_uri: "ldap://openldap.openldap.svc.cluster.local:389"
      bind_cn: "cn=admin,dc=runityai,dc=local"
      bind_password: "LdapAdmin1234"   # синхронизировать с openldap values
      base_dn: "dc=runityai,dc=local"
      user_object_filter: "(objectClass=inetOrgPerson)"
      group_object_filter: "(objectClass=posixGroup)"
      group_membership_field: "memberUid"
      object_uniqueness_field: "uid"
      sync_users: true
      sync_groups: true
      user_path_template: "goauthentik.io/sources/%(slug)s"
```

**`openwebui-provider.yaml`** — OAuth2 провайдер + Application + groups scope:
```yaml
version: 1
metadata:
  name: OpenWebUI OIDC
  labels:
    blueprints.goauthentik.io/instantiate: "true"
entries:
  # Scope mapping для групп
  - model: authentik_providers_oauth2.scopemapping
    state: present
    identifiers:
      name: "openwebui-groups"
    attrs:
      name: "openwebui-groups"
      scope_name: "groups"
      description: "User groups for OpenWebUI"
      expression: |
        return list(request.user.ak_groups.values_list("name", flat=True))

  # OAuth2/OIDC провайдер
  - model: authentik_providers_oauth2.oauth2provider
    state: present
    identifiers:
      name: "openwebui"
    attrs:
      name: "openwebui"
      authorization_flow: !Find [authentik_flows.flow, [slug, default-provider-authorization-implicit-consent]]
      client_id: "openwebui"
      client_secret: "openwebui-client-secret-change-me"
      client_type: "confidential"
      redirect_uris: "http://localhost:8080/oauth/oidc/callback"
      include_claims_in_id_token: true
      signing_key: !Find [authentik_crypto.certificatekeypair, [name, "authentik Self-signed Certificate"]]
      property_mappings:
        - !Find [authentik_providers_oauth2.scopemapping, [scope_name, openid]]
        - !Find [authentik_providers_oauth2.scopemapping, [scope_name, email]]
        - !Find [authentik_providers_oauth2.scopemapping, [scope_name, profile]]
        - !Find [authentik_providers_oauth2.scopemapping, [name, openwebui-groups]]

  # Application в Authentik
  - model: authentik_core.application
    state: present
    identifiers:
      slug: openwebui
    attrs:
      name: "OpenWebUI"
      slug: "openwebui"
      provider: !Find [authentik_providers_oauth2.oauth2provider, [name, openwebui]]
```

### 5. `applications/authentik.yaml` — НОВЫЙ
ArgoCD Application, kustomize (с `--enable-helm`), path `platform/authentik/`:
```yaml
source:
  repoURL: https://github.com/agud97/idp-app-v1.git
  path: platform/authentik
  targetRevision: HEAD
  kustomize:
    buildOptions: --enable-helm
destination:
  namespace: authentik
syncPolicy:
  automated: {prune: true, selfHeal: true}
  syncOptions: [CreateNamespace=true]
```

### 6. `applications/open-webui.yaml` — ИЗМЕНИТЬ
Добавить в `extraEnvVars`:
```yaml
- name: ENABLE_OAUTH_SIGNUP
  value: "true"
- name: OAUTH_PROVIDER_NAME
  value: "Authentik"
- name: OPENID_PROVIDER_URL
  value: "http://authentik-server.authentik.svc.cluster.local:9000/application/o/openwebui/"
- name: OAUTH_CLIENT_ID
  value: "openwebui"
- name: OAUTH_CLIENT_SECRET
  valueFrom:
    secretKeyRef:
      name: open-webui-sso
      key: OAUTH_CLIENT_SECRET
- name: OAUTH_ROLES_CLAIM
  value: "groups"
- name: OAUTH_ADMIN_ROLES
  value: "prod_runityai_admin"
- name: OAUTH_ALLOWED_ROLES
  value: "prod_runityai_admin,prod_runityai_user"
- name: OAUTH_MERGE_ACCOUNTS_BY_EMAIL
  value: "true"
```

---

## Секреты (создать вручную через kubectl)

```bash
# OpenWebUI — клиентский секрет для OIDC
kubectl create secret generic open-webui-sso -n open-webui \
  --from-literal=OAUTH_CLIENT_SECRET="openwebui-client-secret-change-me"
```
(значение должно совпадать с `client_secret` в blueprint `openwebui-provider.yaml`)

---

## Порядок деплоя

1. Создать секрет `open-webui-sso` в `open-webui` (выше)
2. `git add . && git commit && git push` → ArgoCD автоматически:
   - Деплоит OpenLDAP (namespace `openldap`)
   - Деплоит Authentik + ConfigMap с blueprints (namespace `authentik`)
   - Обновляет OpenWebUI (новые env vars)
3. Подождать готовности Authentik (~3-5 минут), blueprints применятся автоматически

---

## Верификация

### Шаг 1 — /etc/hosts + port-forward
```bash
# На локальной машине
echo "127.0.0.1 authentik-server.authentik.svc.cluster.local" >> /etc/hosts

# Два терминала:
kubectl --kubeconfig ~/proj/cross/kubeconfig_6005021 \
  port-forward svc/authentik-server -n authentik 9000:9000

kubectl --kubeconfig ~/proj/cross/kubeconfig_6005021 \
  port-forward svc/open-webui -n open-webui 8080:80
```

### Шаг 2 — Проверить Authentik
1. Открыть `http://authentik-server.authentik.svc.cluster.local:9000`
2. Войти: `admin@runityai.local` / `AuthentikAdmin123!`
3. Directory → Sources → `OpenLDAP` → "Run sync" → убедиться что admin1, user1 синхронизированы
4. Directory → Groups → убедиться что группы `prod_runityai_admin`, `prod_runityai_user` появились
5. Applications → Applications → `OpenWebUI` — должен быть

### Шаг 3 — Проверить SSO в OpenWebUI
1. Открыть `http://localhost:8080`
2. На странице логина должна появиться кнопка "Sign in with Authentik"
3. Войти как `admin1` (группа `prod_runityai_admin`) → роль `admin` в OpenWebUI
4. Войти как `user1` (группа `prod_runityai_user`) → роль `user` в OpenWebUI

---

## Потенциальные проблемы и решения

| Проблема | Решение |
|----------|---------|
| Blueprint `!Find` не может найти default flow | После первого запуска Authentik ждать ~2 минуты, затем manual sync Authentik app в ArgoCD |
| LDAP sync fails: connection refused | Проверить `kubectl get svc -n openldap`, OpenLDAP должен быть Running |
| Authentik pod не стартует (blueprints ConfigMap не существует) | Kustomize создаёт ConfigMap и Deployment вместе — должно работать |
| `redirect_uri mismatch` в Authentik | Проверить что port-forward на 8080, добавить нужный URI в blueprint |
| OpenWebUI не показывает кнопку SSO | Проверить env vars в pod: `kubectl exec -n open-webui open-webui-0 -- env | grep OAUTH` |
