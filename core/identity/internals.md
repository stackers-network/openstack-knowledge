# Keystone Internals

## Token Lifecycle

### Issue

1. Client sends credentials (password, token, application credential, or federated assertion) to `POST /v3/auth/tokens`
2. Keystone routes the request through the auth plugin chain (see Auth Middleware Pipeline below)
3. Auth plugins validate the credentials against the appropriate identity backend
4. Keystone resolves the requested scope (project, domain, or system)
5. RBAC check: the user must have at least one role assignment in the requested scope
6. Token payload is assembled: user ID, scope (project/domain/system ID), audit IDs, expiry, catalog (if requested)
7. For Fernet: payload is serialized (msgpack), encrypted with the primary Fernet key, base64url-encoded
8. For JWS: payload is JSON-encoded, signed with the primary EC private key
9. Token is returned in the `X-Subject-Token` response header

### Validate

1. Another service (or Keystone itself) sends `GET /v3/auth/tokens` with `X-Auth-Token` (caller's token) and `X-Subject-Token` (token to validate)
2. Keystone base64url-decodes the Fernet token and tries decryption with each active Fernet key (primary first, then secondaries)
3. If decryption succeeds, the payload is deserialized and the embedded expiry is checked
4. Revocation check: the token's audit ID is checked against cached revocation events
5. If the subject token is a project-scoped token, Keystone verifies the project still exists and is enabled
6. If validated, Keystone returns a `200 OK` with full token details in the response body
7. Result is cached in Memcached keyed by the token hash (configurable via `[cache]`)

### Revoke

1. Client sends `DELETE /v3/auth/tokens` with `X-Subject-Token`
2. Keystone creates a **revocation event** record in the `revocation_event` table containing:
   - `audit_id` (audit chain ID from the token)
   - `revoked_at` timestamp
3. Revocation events are **not** the token itself — Keystone never stores issued Fernet tokens
4. Revocation events are cached and distributed via Memcached
5. Events expire from the DB after `[token] expiration` seconds (no need to keep them longer than the token lifetime)

### Token Expiry vs. Revocation

```
Token issued          Token expires (default 1h)
    │                         │
    ▼                         ▼
────●─────────────────────────●──────────────────→ time
         │
         ▼
    Token revoked:
    revocation_event inserted;
    all validation attempts return 401 until natural expiry
```

## Fernet Token Format and Key Rotation

### Fernet Token Binary Layout

Fernet is a standard from the `cryptography` Python library. A Keystone Fernet token consists of:

```
0x80           — Version (1 byte)
timestamp      — 64-bit big-endian UNIX timestamp (8 bytes)
IV             — AES-CBC initialization vector (16 bytes, random)
ciphertext     — AES-128-CBC encrypted msgpack payload (variable)
HMAC           — SHA-256 HMAC over all preceding bytes (32 bytes)
```

The whole structure is base64url-encoded (no padding). The final token looks like:

```
gAAAAABm...  (200-300 characters typical)
```

### Fernet Payload (Before Encryption)

Keystone serializes the token payload as msgpack before encryption. Payload fields depend on scope:

**Project-scoped Fernet payload:**
```
[version, user_id, methods, project_id, expires_at, audit_ids, ...]
```

**Domain-scoped:**
```
[version, user_id, methods, domain_id, expires_at, audit_ids, ...]
```

**System-scoped:**
```
[version, user_id, methods, "all", expires_at, audit_ids, ...]
```

### Key Repository Structure

```
/etc/keystone/fernet-keys/
├── 0      ← staged key (next primary after rotation)
├── 1      ← primary key (signs new tokens)
├── 2      ← secondary key (validates recent tokens)
└── 3      ← secondary key (validates older tokens)
```

Keys are named with integer filenames. The **highest-numbered key** is always the **primary** (used to encrypt new tokens). Key `0` is the **staged key** (generated but not yet used to sign). All other keys are secondaries used only for decryption/validation.

### Key Rotation Mechanism

Given `max_active_keys = 5`:

**Before rotation:**
```
0 (staged), 1 (primary), 2 (secondary), 3 (secondary)
```

**After rotation:**
```
0 (new staged), 2 (secondary), 3 (secondary), 4 (new primary ← was staged key 0)
```

Key `1` (old primary) becomes a secondary. Old secondary `1` is renamed. Eventually the oldest secondaries are pruned when the count exceeds `max_active_keys - 2`.

**Rotation frequency recommendation:**
- Rotate at most every `(max_active_keys - 2) * token_expiration / 2` seconds
- With `max_active_keys = 5` and `expiration = 3600s`:  `(5-2) * 3600 / 2 = 5400s ≈ 90 minutes`
- A common production setting is daily rotation with `max_active_keys = 5`

**Multi-node key synchronization:**
All Keystone nodes must have identical key repositories. A token encrypted by node A must be decryptable by node B. Use:
- Ansible to distribute keys after each rotation
- Shared NFS mount (avoid for HA latency reasons)
- Dedicated key management job (cron or systemd timer)

## Auth Middleware Pipeline

### keystonemiddleware.auth_token

Every OpenStack service (Nova, Neutron, Glance, etc.) has `keystonemiddleware.auth_token` in its WSGI pipeline. This middleware intercepts every API request and:

1. Extracts `X-Auth-Token` from the request header
2. Checks the local token cache (Memcached) for a cached validation result
3. If not cached, calls Keystone `GET /v3/auth/tokens` with the token
4. Parses the response and injects request environment variables:
   - `HTTP_X_USER_ID`
   - `HTTP_X_USER_NAME`
   - `HTTP_X_PROJECT_ID`
   - `HTTP_X_PROJECT_NAME`
   - `HTTP_X_DOMAIN_ID`
   - `HTTP_X_ROLES` (comma-separated)
   - `HTTP_X_SERVICE_TOKEN` (if a service token was also provided)
5. Passes the enriched request to the downstream WSGI application

If validation fails, the middleware returns `401 Unauthorized`.

### keystonemiddleware Configuration (paste.ini)

```ini
# /etc/nova/paste.ini (example)
[pipeline:main]
pipeline = cors http_proxy_to_wsgi request_id faultwrapper keystonecontext authtoken ratelimit osapi_compute_app_v21

[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
```

```ini
# /etc/nova/nova.conf (keystone_authtoken section)
[keystone_authtoken]
www_authenticate_uri = http://10.0.0.10:5000
auth_url = http://10.0.0.11:5000
memcached_servers = 10.0.0.10:11211,10.0.0.11:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova_service_pass
# Cache validated tokens for 300 seconds
token_cache_time = 300
# Reject requests with no token (set false only for public endpoints)
delay_auth_decision = false
```

### Keystone Auth Plugins (Server-Side)

On the Keystone server side, auth is handled by **auth plugins** loaded via the `[auth]` config section:

```ini
[auth]
methods = password,token,application_credential,totp,mapped,oauth1
```

Each method maps to a plugin class:

| Method | Plugin | Description |
|---|---|---|
| `password` | `keystone.auth.plugins.password.Password` | Username + password |
| `token` | `keystone.auth.plugins.token.Token` | Re-authenticate with existing token |
| `application_credential` | `keystone.auth.plugins.application_credential.ApplicationCredential` | App credential ID + secret |
| `totp` | `keystone.auth.plugins.totp.TOTP` | Time-based one-time password (MFA) |
| `mapped` | `keystone.auth.plugins.mapped.Mapped` | Federated identity (SAML, OIDC) |
| `oauth1` | `keystone.auth.plugins.oauth1.OAuth` | OAuth 1.0a |

Plugins are chained: a request can specify multiple methods (e.g., `password` + `totp` for MFA).

## Identity Backend

### SQL Backend (Default)

The SQL driver stores users and groups in the MariaDB/PostgreSQL database.

- Tables: `user`, `group`, `group_user`, `local_user`, `password`
- Password hashing: bcrypt (configured via `[identity] password_hash_algorithm`)
- User attributes stored as JSON in the `extra` column (for non-standard attributes)

```ini
[identity]
driver = sql
```

### LDAP Backend

Keystone can delegate user/group reads (and optionally writes) to an LDAP/AD directory.

```ini
[identity]
driver = ldap

[ldap]
url = ldap://10.0.0.20
user = cn=admin,dc=example,dc=com
password = ldap_bind_pass
suffix = dc=example,dc=com

user_tree_dn = ou=Users,dc=example,dc=com
user_objectclass = inetOrgPerson
user_id_attribute = cn
user_name_attribute = cn
user_mail_attribute = mail
user_pass_attribute = userPassword
user_filter = (memberOf=cn=openstack-users,ou=Groups,dc=example,dc=com)

group_tree_dn = ou=Groups,dc=example,dc=com
group_objectclass = groupOfNames
group_id_attribute = cn
group_name_attribute = cn
group_member_attribute = member
group_filter =

# Read-only LDAP (recommended — manage users in LDAP, not via Keystone API)
user_allow_create = false
user_allow_update = false
user_allow_delete = false
group_allow_create = false
group_allow_update = false
group_allow_delete = false

# TLS
use_tls = true
tls_cacertfile = /etc/keystone/ldap-ca.pem
tls_req_cert = demand
```

**Per-domain LDAP backends**: Configure separate LDAP directories per domain by placing per-domain config in `/etc/keystone/domains/keystone.{domain_name}.conf`.

```ini
# /etc/keystone/domains/keystone.acme-corp.conf
[identity]
driver = ldap

[ldap]
url = ldap://10.0.0.21
suffix = dc=acme-corp,dc=com
...
```

Enable domain-specific backends:

```ini
# /etc/keystone/keystone.conf
[identity]
domain_specific_drivers_enabled = true
domain_config_dir = /etc/keystone/domains
```

## Assignment Backend

The assignment driver stores role assignments between actors (users, groups) and targets (projects, domains, system).

- **Only `sql` is supported** in modern Keystone
- Tables: `assignment`, `implied_role`, `system_assignment`
- Role inheritance is computed at query time via recursive traversal of the project tree

```ini
[assignment]
driver = sql
```

## Resource Backend

The resource driver stores domains and projects.

- Only `sql` is supported
- Tables: `project` (used for both projects and domains; `is_domain` flag distinguishes them)

```ini
[resource]
driver = sql
cache_time = 600
```

## Catalog Backend

The catalog driver stores services and endpoints.

- **`sql`** (default): endpoints stored in DB, mutable via API
- **`templated`**: endpoints defined in a static template file (`/etc/keystone/default_catalog.templates`), read-only, no API management. Rarely used in production.

```ini
[catalog]
driver = sql
```

## Credential Encryption

All user credentials stored in the `credential` table are encrypted using **AES-256-GCM** with keys from `/etc/keystone/credential-keys/`.

The key repository has the same staged/primary/secondary structure as Fernet keys but is managed by separate commands:

```bash
# Setup (once)
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

# Rotate keys
keystone-manage credential_rotate --keystone-user keystone --keystone-group keystone

# Re-encrypt all stored credentials with new primary key
keystone-manage credential_migrate --keystone-user keystone --keystone-group keystone
```

After `credential_rotate`, the old primary becomes a secondary and can still decrypt existing blobs. Run `credential_migrate` to re-encrypt all blobs so the old key can be safely removed.

## Token Validation Caching

Keystonemiddleware caches token validation results in Memcached to avoid hitting Keystone on every API request.

### Cache Keys

Keystonemiddleware stores validation responses using a hash of the token as the cache key. Multiple services sharing the same Memcached cluster will share cached results.

### Cache Configuration

```ini
# In each service's .conf file (e.g., nova.conf, neutron.conf)
[keystone_authtoken]
memcached_servers = 10.0.0.10:11211,10.0.0.11:11211
token_cache_time = 300       # seconds
revocation_cache_time = 10   # how often to fetch revocation list
```

### Cache Invalidation

When a token is revoked:
- The revocation event is added to the DB
- `keystonemiddleware` periodically fetches the revocation list (every `revocation_cache_time` seconds)
- Until the revocation list is refreshed, a revoked token may still pass validation in the cache window

This is a **known, intentional trade-off**: the revocation propagation delay equals `min(token_cache_time, revocation_cache_time)`.

### Server-Side Token Caching

Keystone itself caches project/domain resolution, user lookups, and role assignments:

```ini
[cache]
enabled = true
backend = dogpile.cache.pymemcache
backend_argument = url:10.0.0.10:11211
expiration_time = 600

# Enable per-subsystem caches
[token]
cache_on_issue = true

[resource]
cache_time = 3600

[role]
cache_time = 3600
```

## Request Flow Through Keystone

```
Client
  │
  ▼  POST /v3/auth/tokens  {auth: {identity: {methods: [password], ...}, scope: {...}}}
  ▼
Apache httpd (mod_wsgi)
  │
  ▼
keystone WSGI router
  │
  ▼
auth controller
  │
  ├─→ [password plugin]
  │       └─→ identity backend (SQL/LDAP) — validate username + password hash
  │
  ├─→ scope resolution
  │       └─→ resource backend — resolve project/domain
  │
  ├─→ role assignment check
  │       └─→ assignment backend — user has role in scope?
  │
  ├─→ token assembly
  │       └─→ fernet provider — encrypt payload
  │
  └─→ 201 Created, X-Subject-Token: gAAAAABm...
```
