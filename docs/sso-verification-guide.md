# SSO Stack — Руководство по верификации

Подробный пошаговый гайд по проверке всего SSO-стека:
**OpenLDAP → Authentik → Open WebUI**

Каждая команда снабжена объяснением, что она делает, и ожидаемым результатом.

---

## Общая архитектура

```
┌─────────────┐     LDAP sync      ┌─────────────────┐     OIDC/OAuth2     ┌─────────────┐
│  OpenLDAP   │ ─────────────────► │    Authentik    │ ──────────────────► │  Open WebUI │
│  (источник  │  ldap://openldap   │  (IdP / OIDC    │  /application/o/    │  (клиент)   │
│   данных)   │  .openldap.svc:389 │   Provider)     │  openwebui/         │             │
└─────────────┘                    └─────────────────┘                     └─────────────┘

Namespace:   openldap              authentik                               open-webui
```

**Переменные окружения (используются во всём гайде):**
```bash
export KUBECONFIG=/root/proj/cross/kubeconfig_6005021
```

---

## 1. Проверка OpenLDAP

### 1.1 Статус пода

```bash
kubectl get pods -n openldap
```

**Ожидаемый результат:**
```
NAME         READY   STATUS    RESTARTS   AGE
openldap-0   1/1     Running   0          Xm
```

### 1.2 Просмотр всего дерева LDAP

Команда `ldapsearch` выполняется внутри пода `openldap-0`.
Флаги: `-x` = простая аутентификация, `-H` = URL сервера, `-D` = DN для bind,
`-w` = пароль, `-b` = base DN для поиска, `(objectClass=*)` = все объекты, `dn` = выводить только DN.

```bash
kubectl exec -n openldap openldap-0 -- \
  ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=runityai,dc=local" -w LdapAdmin1234 \
  -b "dc=runityai,dc=local" "(objectClass=*)" dn
```

**Ожидаемый результат** — дерево из 7 объектов:
```
dn: dc=runityai,dc=local
dn: cn=admin,dc=runityai,dc=local
dn: ou=people,dc=runityai,dc=local
dn: ou=groups,dc=runityai,dc=local
dn: uid=admin1,ou=people,dc=runityai,dc=local
dn: uid=user1,ou=people,dc=runityai,dc=local
dn: cn=prod_runityai_admin,ou=groups,dc=runityai,dc=local
dn: cn=prod_runityai_user,ou=groups,dc=runityai,dc=local
```

### 1.3 Проверка пользователей

Фильтр `(objectClass=inetOrgPerson)` отбирает только пользователей (не группы, не OU).
Атрибуты `uid cn mail` — минимальный набор, который читает Authentik при синхронизации.

```bash
kubectl exec -n openldap openldap-0 -- \
  ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=runityai,dc=local" -w LdapAdmin1234 \
  -b "ou=people,dc=runityai,dc=local" "(objectClass=inetOrgPerson)" \
  uid cn mail uidNumber
```

**Ожидаемый результат:**
```
# admin1, people, runityai.local
dn: uid=admin1,ou=people,dc=runityai,dc=local
uid: admin1
cn: admin1
mail: admin1@runityai.local
uidNumber: 1001

# user1, people, runityai.local
dn: uid=user1,ou=people,dc=runityai,dc=local
uid: user1
cn: user1
mail: user1@runityai.local
uidNumber: 1002
```

### 1.4 Проверка групп

Фильтр `(objectClass=posixGroup)` отбирает только группы.
Важно: группы должны содержать `objectClass: uidObject` и атрибут `uid` —
это необходимо для корректной синхронизации Authentik (поле `object_uniqueness_field=uid`).

```bash
kubectl exec -n openldap openldap-0 -- \
  ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=runityai,dc=local" -w LdapAdmin1234 \
  -b "ou=groups,dc=runityai,dc=local" "(objectClass=posixGroup)"
```

**Ожидаемый результат:**
```
# prod_runityai_admin, groups, runityai.local
dn: cn=prod_runityai_admin,ou=groups,dc=runityai,dc=local
objectClass: posixGroup
objectClass: uidObject       ← ВАЖНО: без этого Authentik не синхронизирует группы
cn: prod_runityai_admin
gidNumber: 2000
memberUid: admin1
uid: prod_runityai_admin     ← ВАЖНО: значение должно совпадать с cn

# prod_runityai_user, groups, runityai.local
dn: cn=prod_runityai_user,ou=groups,dc=runityai,dc=local
objectClass: posixGroup
objectClass: uidObject
cn: prod_runityai_user
gidNumber: 2001
memberUid: user1
uid: prod_runityai_user
```

### 1.5 Проверка bind-коннекта (тест доступности)

Проверяет, что LDAP сервер отвечает на bind-запрос. `result: 0 Success` = OK.

```bash
kubectl exec -n openldap openldap-0 -- \
  ldapwhoami -x -H ldap://localhost:389 \
  -D "cn=admin,dc=runityai,dc=local" -w LdapAdmin1234
```

**Ожидаемый результат:**
```
dn:cn=admin,dc=runityai,dc=local
```

### 1.6 Проверка доступности LDAP из namespace authentik

Подтверждает сетевую связность между namespace `authentik` и `openldap`.
Запускает временный pod `busybox` в namespace `authentik`, который выполняет `nslookup` и `nc`.

```bash
kubectl run ldap-test --rm -i --restart=Never \
  --image=busybox:1.36 -n authentik -- sh -c \
  'nslookup openldap.openldap.svc.cluster.local && \
   nc -zv openldap.openldap.svc.cluster.local 389 && echo "LDAP reachable"'
```

**Ожидаемый результат:**
```
Server:    10.96.0.10
Address:   10.96.0.10:53
Name:      openldap.openldap.svc.cluster.local
Address:   10.107.x.x
openldap.openldap.svc.cluster.local (10.107.x.x:389) open
LDAP reachable
```

---

## 2. Проверка Authentik

### 2.1 Статус всех подов

```bash
kubectl get pods -n authentik
```

**Ожидаемый результат:**
```
NAME                                    READY   STATUS    RESTARTS
authentik-postgresql-0                  1/1     Running   0
authentik-redis-master-0                1/1     Running   0
authentik-server-<hash>                 1/1     Running   0
authentik-worker-<hash>                 1/1     Running   0
```

Компоненты:
- `authentik-postgresql-0` — PostgreSQL база данных Authentik
- `authentik-redis-master-0` — Redis (сессии, кэш задач)
- `authentik-server-*` — основной веб-сервер + API
- `authentik-worker-*` — фоновый воркер (LDAP sync, blueprint apply, etc.)

### 2.2 Проверка применения Blueprint'ов

Blueprint — декларативный способ настройки Authentik через YAML-файлы.
Оба кастомных blueprint'а (LDAP source + OpenWebUI OIDC provider) должны быть в статусе `successful`.

```bash
kubectl exec -n authentik \
  $(kubectl get pods -n authentik -l app.kubernetes.io/component=server \
    -o jsonpath='{.items[0].metadata.name}') -- \
  ak shell -c "
from authentik.blueprints.models import BlueprintInstance
for b in BlueprintInstance.objects.filter(name__in=['OpenLDAP Source', 'OpenWebUI OIDC']):
    print(f'{b.name} | status={b.status} | last_applied={b.last_applied}')
"
```

**Ожидаемый результат:**
```
OpenLDAP Source  | status=successful | last_applied=2026-03-02 20:43:58...
OpenWebUI OIDC   | status=successful | last_applied=2026-03-02 20:34:09...
```

### 2.3 Проверка LDAP Source конфигурации

Читает параметры LDAP Source напрямую из базы Authentik.
Подтверждает, что blueprint применился корректно.

```bash
kubectl exec -n authentik \
  $(kubectl get pods -n authentik -l app.kubernetes.io/component=server \
    -o jsonpath='{.items[0].metadata.name}') -- \
  ak shell -c "
from authentik.sources.ldap.models import LDAPSource
s = LDAPSource.objects.get(slug='openldap')
print(f'Name:               {s.name}')
print(f'Server URI:         {s.server_uri}')
print(f'Base DN:            {s.base_dn}')
print(f'Bind CN:            {s.bind_cn}')
print(f'Sync users:         {s.sync_users}')
print(f'Sync groups:        {s.sync_groups}')
print(f'Uniqueness field:   {s.object_uniqueness_field}')
print(f'Group member field: {s.group_membership_field}')
"
```

**Ожидаемый результат:**
```
Name:               OpenLDAP
Server URI:         ldap://openldap.openldap.svc.cluster.local:389
Base DN:            dc=runityai,dc=local
Bind CN:            cn=admin,dc=runityai,dc=local
Sync users:         True
Sync groups:        True
Uniqueness field:   uid
Group member field: memberUid
```

### 2.4 Запуск LDAP-синхронизации вручную

Запускает синхронизацию немедленно, не дожидаясь планировщика.
Выводит количество синхронизированных users/groups и список событий с ошибками (если есть).

```bash
kubectl exec -n authentik \
  $(kubectl get pods -n authentik -l app.kubernetes.io/component=server \
    -o jsonpath='{.items[0].metadata.name}') -- \
  ak shell -c "
from authentik.sources.ldap.models import LDAPSource
from authentik.sources.ldap.sync.users import UserLDAPSynchronizer
from authentik.sources.ldap.sync.groups import GroupLDAPSynchronizer
from authentik.sources.ldap.sync.membership import MembershipLDAPSynchronizer

source = LDAPSource.objects.get(slug='openldap')

sync_user = UserLDAPSynchronizer(source)
messages_u = sync_user.sync()
print(f'Users synced:  {sync_user.sync_to_db_count}')
for m in messages_u: print(f'  [event] {m.event}')

sync_group = GroupLDAPSynchronizer(source)
messages_g = sync_group.sync()
print(f'Groups synced: {sync_group.sync_to_db_count}')
for m in messages_g: print(f'  [event] {m.event}')

sync_mem = MembershipLDAPSynchronizer(source)
messages_m = sync_mem.sync()
print(f'Memberships synced.')
for m in messages_m: print(f'  [event] {m.event}')
"
```

**Ожидаемый результат:**
```
Users synced:  2
Groups synced: 2
Memberships synced.
```
Строк с `[event]` быть не должно (они сигнализируют об ошибках).

### 2.5 Проверка синхронизированных пользователей и групп

Читает из базы Authentik только тех пользователей, которые пришли из LDAP-источника
(путь `goauthentik.io/sources/openldap`), и выводит их группы.

```bash
kubectl exec -n authentik \
  $(kubectl get pods -n authentik -l app.kubernetes.io/component=server \
    -o jsonpath='{.items[0].metadata.name}') -- \
  ak shell -c "
from authentik.core.models import User, Group
print('=== LDAP USERS ===')
for u in User.objects.filter(path__startswith='goauthentik.io/sources/openldap'):
    grps = list(u.ak_groups.values_list('name', flat=True))
    print(f'  {u.username} | {u.email} | groups={grps}')

print()
print('=== GROUPS ===')
for g in Group.objects.filter(name__startswith='prod_runityai'):
    members = list(g.users.values_list('username', flat=True))
    ldap_uniq = g.attributes.get('ldap_uniq', 'N/A')
    print(f'  {g.name} | members={members} | ldap_uniq={ldap_uniq}')
"
```

**Ожидаемый результат:**
```
=== LDAP USERS ===
  admin1 | admin1@runityai.local | groups=['prod_runityai_admin']
  user1  | user1@runityai.local  | groups=['prod_runityai_user']

=== GROUPS ===
  prod_runityai_admin | members=['admin1'] | ldap_uniq=prod_runityai_admin
  prod_runityai_user  | members=['user1']  | ldap_uniq=prod_runityai_user
```

### 2.6 Проверка OAuth2 Provider конфигурации

Читает параметры OIDC-провайдера `openwebui` из базы Authentik.

```bash
kubectl exec -n authentik \
  $(kubectl get pods -n authentik -l app.kubernetes.io/component=server \
    -o jsonpath='{.items[0].metadata.name}') -- \
  ak shell -c "
from authentik.providers.oauth2.models import OAuth2Provider
p = OAuth2Provider.objects.get(name='openwebui')
print(f'Name:                     {p.name}')
print(f'Client ID:                {p.client_id}')
print(f'Client type:              {p.client_type}')
print(f'Redirect URIs:            {p.redirect_uris}')
print(f'Include claims in token:  {p.include_claims_in_id_token}')
scopes = list(p.property_mappings.values_list('name', flat=True))
print(f'Scopes/Mappings:          {scopes}')
"
```

**Ожидаемый результат:**
```
Name:                     openwebui
Client ID:                openwebui
Client type:              confidential
Redirect URIs:            [RedirectURI(matching_mode=STRICT, url='http://localhost:8080/oauth/oidc/callback')]
Include claims in token:  True
Scopes/Mappings:          ["authentik default OAuth Mapping: OpenID 'openid'",
                           'openwebui-groups',
                           "authentik default OAuth Mapping: OpenID 'email'",
                           "authentik default OAuth Mapping: OpenID 'profile'"]
```

### 2.7 Проверка OIDC Discovery Endpoint (port-forward)

**Шаг 1.** Установить port-forward к сервису Authentik.
Сервис `authentik-server` слушает на порту `80` (контейнер использует порт `9000`).

```bash
kubectl port-forward svc/authentik-server 19000:80 -n authentik &
```

**Шаг 2.** Запросить OIDC discovery document.
Это стандартный endpoint RFC 8414 — его открывает каждый OIDC Provider.
По нему клиент (Open WebUI) узнаёт URL'ы authorization, token, userinfo endpoints.

```bash
curl -s http://localhost:19000/application/o/openwebui/.well-known/openid-configuration \
  | python3 -m json.tool
```

**Ожидаемый результат (полный документ):**
```json
{
    "issuer": "http://localhost:19000/application/o/openwebui/",
    "authorization_endpoint": "http://localhost:19000/application/o/authorize/",
    "token_endpoint": "http://localhost:19000/application/o/token/",
    "userinfo_endpoint": "http://localhost:19000/application/o/userinfo/",
    "end_session_endpoint": "http://localhost:19000/application/o/openwebui/end-session/",
    "introspection_endpoint": "http://localhost:19000/application/o/introspect/",
    "revocation_endpoint": "http://localhost:19000/application/o/revoke/",
    "device_authorization_endpoint": "http://localhost:19000/application/o/device/",
    "response_types_supported": ["code", "id_token", "id_token token", "code token",
                                  "code id_token", "code id_token token"],
    "response_modes_supported": ["query", "fragment", "form_post"],
    "jwks_uri": "http://localhost:19000/application/o/openwebui/jwks/",
    "grant_types_supported": ["authorization_code", "refresh_token", "implicit",
                               "client_credentials", "password",
                               "urn:ietf:params:oauth:grant-type:device_code"],
    "id_token_signing_alg_values_supported": ["RS256"],
    "subject_types_supported": ["public"],
    "token_endpoint_auth_methods_supported": ["client_secret_post", "client_secret_basic"],
    "scopes_supported": ["openid", "groups", "email", "profile"],
    "claims_supported": ["sub", "iss", "aud", "exp", "iat", "auth_time", "acr", "amr",
                         "nonce", "email", "email_verified", "name", "given_name",
                         "preferred_username", "nickname", "groups"],
    "code_challenge_methods_supported": ["plain", "S256"]
}
```

Ключевые проверки:
- `scopes_supported` содержит `"groups"` — Open WebUI получит роли пользователя
- `claims_supported` содержит `"groups"` — группы будут в ID-токене
- `id_token_signing_alg_values_supported`: `"RS256"` — токены подписаны RSA

### 2.8 Проверка OIDC Discovery из namespace open-webui (in-cluster)

Подтверждает, что Open WebUI pod реально может дойти до Authentik по внутреннему имени.
Сервис `authentik-server` в namespace `authentik` слушает на порту **80** (не 9000).

```bash
kubectl exec -n open-webui open-webui-0 -- \
  curl -sv --max-time 10 \
  "http://authentik-server.authentik.svc.cluster.local/application/o/openwebui/.well-known/openid-configuration" \
  2>&1 | grep -E "< HTTP|issuer|scopes_supported"
```

**Ожидаемый результат:**
```
< HTTP/1.1 200 OK
    "issuer": "http://authentik-server.authentik.svc.cluster.local/application/o/openwebui/",
    "scopes_supported": ["openid", "groups", "email", "profile"],
```

> **Важно:** URL должен использовать порт 80 (по умолчанию для HTTP), а НЕ 9000.
> Порт 9000 — это контейнерный порт, не сервисный.

---

## 3. Проверка Open WebUI

### 3.1 Статус пода и наличие секрета

```bash
kubectl get pods -n open-webui
kubectl get secret open-webui-sso -n open-webui
```

**Ожидаемый результат:**
```
NAME          READY   STATUS    RESTARTS
open-webui-0  1/1     Running   0

NAME             TYPE     DATA   AGE
open-webui-sso   Opaque   1      Xm
```

### 3.2 Проверка содержимого секрета

Секрет `open-webui-sso` создан вручную (не через ArgoCD) и содержит client_secret для OAuth2.
Значение должно совпадать с `client_secret` в Blueprint'е Authentik.

```bash
kubectl get secret open-webui-sso -n open-webui \
  -o jsonpath='{.data.OAUTH_CLIENT_SECRET}' | base64 -d
echo
```

**Ожидаемый результат:**
```
openwebui-client-secret-change-me
```

### 3.3 Проверка SSO env vars в поде

Все переменные окружения для SSO должны присутствовать в running-поде.

```bash
kubectl exec -n open-webui open-webui-0 -- env \
  | grep -E "OAUTH|OPENID|ENABLE_OAUTH" | sort
```

**Ожидаемый результат:**
```
ENABLE_OAUTH_SIGNUP=true
OAUTH_ADMIN_ROLES=prod_runityai_admin
OAUTH_ALLOWED_ROLES=prod_runityai_admin,prod_runityai_user
OAUTH_CLIENT_ID=openwebui
OAUTH_CLIENT_SECRET=openwebui-client-secret-change-me
OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true
OAUTH_PROVIDER_NAME=Authentik
OAUTH_ROLES_CLAIM=groups
OPENID_PROVIDER_URL=http://authentik-server.authentik.svc.cluster.local/application/o/openwebui/
```

Расшифровка переменных:
| Переменная | Значение | Описание |
|---|---|---|
| `ENABLE_OAUTH_SIGNUP` | `true` | Разрешить создание аккаунта при первом SSO-логине |
| `OAUTH_PROVIDER_NAME` | `Authentik` | Название кнопки на странице логина |
| `OPENID_PROVIDER_URL` | `http://authentik-server.../application/o/openwebui/` | Base URL OIDC-провайдера (без `.well-known/...`) |
| `OAUTH_CLIENT_ID` | `openwebui` | client_id из Authentik OAuth2 Provider |
| `OAUTH_CLIENT_SECRET` | `openwebui-client-secret-change-me` | client_secret (из Secret) |
| `OAUTH_ROLES_CLAIM` | `groups` | Поле в JWT-токене, из которого читаются роли |
| `OAUTH_ADMIN_ROLES` | `prod_runityai_admin` | Роль для получения admin-прав в Open WebUI |
| `OAUTH_ALLOWED_ROLES` | `prod_runityai_admin,prod_runityai_user` | Только эти роли могут войти |
| `OAUTH_MERGE_ACCOUNTS_BY_EMAIL` | `true` | Объединять аккаунты с одинаковым email |

### 3.4 Проверка через API /api/config

Open WebUI — SPA (Single Page Application). Статическая страница `/` не содержит данных о SSO.
Вместо этого нужно вызвать API-эндпоинт, который используется frontend'ом при загрузке.

```bash
# Установить port-forward (если ещё не запущен)
kubectl port-forward svc/open-webui 18080:80 -n open-webui &

# Запросить конфигурацию
curl -s http://localhost:18080/api/config | python3 -m json.tool
```

**Ожидаемый результат:**
```json
{
    "status": true,
    "name": "Open WebUI",
    "version": "0.5.16",
    "oauth": {
        "providers": {
            "oidc": "Authentik"
        }
    },
    "features": {
        "auth": true,
        "enable_signup": false,
        "enable_login_form": true
    }
}
```

Поле `"oauth": {"providers": {"oidc": "Authentik"}}` подтверждает:
- SSO сконфигурирован
- Кнопка "Continue with Authentik" появится на странице `/auth`

### 3.5 Проверка страницы авторизации

Open WebUI загружает страницу авторизации как SPA, кнопка SSO рендерится через JS.
Однако API `/api/config` выше уже подтверждает наличие SSO провайдера.

Для визуальной проверки достаточно открыть `http://localhost:18080/auth` в браузере.
На странице будет присутствовать кнопка `Continue with Authentik`.

---

## 4. Проверка всего SSO-флоу (end-to-end)

### 4.1 Диаграмма потока авторизации

```
Браузер          Open WebUI              Authentik                OpenLDAP
   │                  │                      │                        │
   │  GET /auth        │                      │                        │
   │─────────────────►│                      │                        │
   │                  │  GET /.well-known/   │                        │
   │                  │  openid-config       │                        │
   │                  │─────────────────────►│                        │
   │                  │◄─────────────────────│                        │
   │  302 redirect    │                      │                        │
   │◄─────────────────│                      │                        │
   │                  │                      │                        │
   │  GET /authorize? │                      │                        │
   │  client_id=...   │                      │                        │
   │─────────────────────────────────────────►                        │
   │  HTML: login form│                      │                        │
   │◄─────────────────────────────────────────                        │
   │                  │                      │                        │
   │  POST credentials│                      │                        │
   │─────────────────────────────────────────►                        │
   │                  │                      │  LDAP bind+search       │
   │                  │                      │───────────────────────►│
   │                  │                      │◄───────────────────────│
   │  302 ?code=...   │                      │                        │
   │◄─────────────────────────────────────────                        │
   │                  │                      │                        │
   │  GET /oauth/oidc/callback?code=...      │                        │
   │─────────────────►│                      │                        │
   │                  │  POST /token         │                        │
   │                  │  code exchange       │                        │
   │                  │─────────────────────►│                        │
   │                  │  access_token +      │                        │
   │                  │  id_token (JWT)      │                        │
   │                  │◄─────────────────────│                        │
   │                  │  GET /userinfo       │                        │
   │                  │─────────────────────►│                        │
   │                  │  {groups: [...]}     │                        │
   │                  │◄─────────────────────│                        │
   │  Logged in!      │                      │                        │
   │◄─────────────────│                      │                        │
```

### 4.2 Ручная проверка токена (curl OIDC flow)

Эмулирует Client Credentials flow для проверки token endpoint (не Authorization Code flow,
но позволяет убедиться, что client_id/secret принимаются).

```bash
# Запустить port-forward
kubectl port-forward svc/authentik-server 19000:80 -n authentik &

# Запросить токен через client_credentials
curl -s -X POST http://localhost:19000/application/o/token/ \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=openwebui" \
  -d "client_secret=openwebui-client-secret-change-me" \
  -d "scope=openid" | python3 -m json.tool
```

**Ожидаемый результат:**
```json
{
    "access_token": "...",
    "expires_in": 3600,
    "token_type": "Bearer",
    "scope": "openid"
}
```

### 4.3 Проверка JWKS (ключи для верификации токенов)

JWKS — JSON Web Key Set. Open WebUI использует эти публичные ключи для проверки
подписи JWT-токенов, полученных от Authentik.

```bash
curl -s http://localhost:19000/application/o/openwebui/jwks/ | python3 -m json.tool
```

**Ожидаемый результат:**
```json
{
    "keys": [
        {
            "kty": "RSA",
            "use": "sig",
            "alg": "RS256",
            "kid": "...",
            "n": "...",
            "e": "AQAB"
        }
    ]
}
```

RSA-ключ присутствует — токены будут верифицированы корректно.

### 4.4 Проверка содержимого ID-токена

Получает Authorization Code через resource owner password grant (для тестирования),
затем декодирует JWT без проверки подписи, чтобы посмотреть claims.

> **Примечание:** `password` grant должен быть включён в Authentik.
> Это только для отладки — в production используется `authorization_code`.

```bash
# Получить токен через password grant
RESPONSE=$(curl -s -X POST http://localhost:19000/application/o/token/ \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=openwebui" \
  -d "client_secret=openwebui-client-secret-change-me" \
  -d "username=admin1" \
  -d "password=Admin1Pass!" \
  -d "scope=openid email profile groups")

echo "$RESPONSE" | python3 -m json.tool

# Декодировать JWT (только payload, без проверки подписи)
ID_TOKEN=$(echo "$RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['id_token'])")
echo "$ID_TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool
```

**Ожидаемый payload:**
```json
{
    "iss": "http://localhost:19000/application/o/openwebui/",
    "sub": "...",
    "aud": "openwebui",
    "email": "admin1@runityai.local",
    "email_verified": true,
    "name": "admin1",
    "preferred_username": "admin1",
    "groups": ["prod_runityai_admin"]
}
```

Поле `"groups": ["prod_runityai_admin"]` — именно его читает Open WebUI через
`OAUTH_ROLES_CLAIM=groups` для назначения прав.

---

## 5. Проверка ArgoCD Application статусов

Все три приложения должны быть `Synced` + `Healthy`.

```bash
kubectl get applications -n argocd openldap authentik open-webui \
  -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'
```

**Ожидаемый результат:**
```
NAME        SYNC     HEALTH
openldap    Synced   Healthy
authentik   Synced   Healthy
open-webui  Synced   Healthy
```

---

## 6. Быстрый чеклист (все команды одним блоком)

```bash
export KUBECONFIG=/root/proj/cross/kubeconfig_6005021

# ── ArgoCD ────────────────────────────────────────────────────────────────
kubectl get applications -n argocd openldap authentik open-webui \
  -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'

# ── Pods ──────────────────────────────────────────────────────────────────
kubectl get pods -n openldap && kubectl get pods -n authentik && kubectl get pods -n open-webui

# ── LDAP: tree ────────────────────────────────────────────────────────────
kubectl exec -n openldap openldap-0 -- \
  ldapsearch -x -H ldap://localhost:389 -D "cn=admin,dc=runityai,dc=local" \
  -w LdapAdmin1234 -b "dc=runityai,dc=local" "(objectClass=*)" dn

# ── LDAP: groups (check uidObject + uid) ──────────────────────────────────
kubectl exec -n openldap openldap-0 -- \
  ldapsearch -x -H ldap://localhost:389 -D "cn=admin,dc=runityai,dc=local" \
  -w LdapAdmin1234 -b "ou=groups,dc=runityai,dc=local" "(objectClass=posixGroup)"

# ── Authentik: blueprints status ──────────────────────────────────────────
kubectl exec -n authentik \
  $(kubectl get pods -n authentik -l app.kubernetes.io/component=server -o jsonpath='{.items[0].metadata.name}') -- \
  ak shell -c "from authentik.blueprints.models import BlueprintInstance; \
  [print(b.name, b.status) for b in BlueprintInstance.objects.filter(name__in=['OpenLDAP Source','OpenWebUI OIDC'])]"

# ── Authentik: synced users & groups ──────────────────────────────────────
kubectl exec -n authentik \
  $(kubectl get pods -n authentik -l app.kubernetes.io/component=server -o jsonpath='{.items[0].metadata.name}') -- \
  ak shell -c "
from authentik.core.models import User, Group
[print(u.username, list(u.ak_groups.values_list('name',flat=True))) for u in User.objects.filter(path__startswith='goauthentik.io/sources/openldap')]
[print(g.name, list(g.users.values_list('username',flat=True))) for g in Group.objects.filter(name__startswith='prod_runityai')]
"

# ── Authentik: OIDC discovery (port-forward) ──────────────────────────────
kubectl port-forward svc/authentik-server 19000:80 -n authentik &
sleep 3
curl -s http://localhost:19000/application/o/openwebui/.well-known/openid-configuration \
  | python3 -m json.tool | grep -E "issuer|scopes_supported|claims_supported"

# ── Open WebUI: OIDC env vars ─────────────────────────────────────────────
kubectl exec -n open-webui open-webui-0 -- env | grep -E "OAUTH|OPENID|ENABLE_OAUTH" | sort

# ── Open WebUI: API config (SSO provider visible) ─────────────────────────
kubectl port-forward svc/open-webui 18080:80 -n open-webui &
sleep 3
curl -s http://localhost:18080/api/config | python3 -m json.tool | grep -A5 oauth
```

---

## 7. Диагностика проблем

### Authentik worker не обрабатывает blueprint

```bash
# Проверить логи worker'а
kubectl logs -n authentik \
  $(kubectl get pods -n authentik -l app.kubernetes.io/component=worker -o jsonpath='{.items[0].metadata.name}') \
  --tail=50 | grep -i "blueprint\|error\|exception"
```

### LDAP-синхронизация не видит группы (0 groups synced)

Причина: в группе нет `uid` атрибута, а `object_uniqueness_field=uid` в LDAP Source.
Решение: добавить `objectClass: uidObject` и `uid: <group-name>` в LDIF.

```bash
# Добавить uidObject к группе (если не было изначально)
kubectl exec -n openldap openldap-0 -- \
  ldapmodify -x -H ldap://localhost:389 \
  -D "cn=admin,dc=runityai,dc=local" -w LdapAdmin1234 <<'LDIF'
dn: cn=prod_runityai_admin,ou=groups,dc=runityai,dc=local
changetype: modify
add: objectClass
objectClass: uidObject
-
add: uid
uid: prod_runityai_admin
LDIF
```

### Blueprint в статусе `error`

```bash
# Посмотреть детали ошибки blueprint'а
kubectl exec -n authentik \
  $(kubectl get pods -n authentik -l app.kubernetes.io/component=server -o jsonpath='{.items[0].metadata.name}') -- \
  ak shell -c "
from authentik.blueprints.models import BlueprintInstance
b = BlueprintInstance.objects.get(name='OpenLDAP Source')
print(b.status, b.metadata)
from authentik.blueprints.v1.importer import Importer
importer = Importer(b.retrieve(), {})
valid, logs = importer.validate()
print('Valid:', valid)
for l in logs: print(l.event)
"
```

### ConfigMap с blueprint'ами не обновился в поде

Kubelet обновляет mounted ConfigMap с задержкой ~60 секунд.
Проверить текущее содержимое файла в поде:

```bash
kubectl exec -n authentik \
  $(kubectl get pods -n authentik -l app.kubernetes.io/component=worker -o jsonpath='{.items[0].metadata.name}') -- \
  ls -la /blueprints/custom/

kubectl exec -n authentik \
  $(kubectl get pods -n authentik -l app.kubernetes.io/component=worker -o jsonpath='{.items[0].metadata.name}') -- \
  cat /blueprints/custom/ldap-source.yaml
```

---

*Дата создания: 2026-03-03*
*Стек: OpenLDAP 2.0.4 (jp-gouin), Authentik 2025.2.1, Open WebUI 5.18.0*
