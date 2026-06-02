# Keystone Architecture

## Keystone Components

Keystone runs as a **WSGI application** served by Apache httpd (via mod_wsgi or mod_proxy_uwsgi) or Nginx. There is no separate daemon; the process model is entirely managed by the web server.

```
                 ┌─────────────────────────────────────────┐
                 │             Apache httpd                 │
                 │  ┌─────────────────────────────────────┐│
                 │  │     keystone WSGI application        ││
                 │  │  ┌──────────┐  ┌──────────────────┐ ││
                 │  │  │  Router  │→ │  Auth Handlers   │ ││
                 │  │  └──────────┘  └──────────────────┘ ││
                 │  │  ┌──────────────────────────────────┐││
                 │  │  │         oslo.policy RBAC          │││
                 │  │  └──────────────────────────────────┘││
                 │  └─────────────────────────────────────┘│
                 └─────────────────────────────────────────┘
                           │                  │
                    ┌──────┴──────┐    ┌──────┴──────┐
                    │  Database   │    │  Memcached  │
                    │ (MariaDB/PG)│    │   (cache)   │
                    └─────────────┘    └─────────────┘
```

**Key processes and files:**

- `/etc/keystone/keystone.conf` — primary configuration file
- `/etc/keystone/policy.yaml` — RBAC policy overrides (empty by default; defaults are in code)
- `/etc/keystone/fernet-keys/` — Fernet token encryption keys (directory)
- `/etc/keystone/credential-keys/` — Credential encryption keys (separate directory)
- `/var/log/keystone/keystone.log` — application log
- WSGI entrypoint: `keystone.server.wsgi` (for Apache)

**API endpoints** (default ports):
- Public API: `http://10.0.0.10:5000/v3` (Identity v3 only; v2 was removed in Queens)
- Admin API: historically on port 35357, now merged — both public and admin use port 5000 in modern deployments

## Identity Model

### Domains

A **domain** is the highest-level isolation boundary. Every user, group, and project lives within exactly one domain. The `Default` domain always exists.

- `Default` domain: contains legacy resources created before domain-awareness
- Custom domains (e.g., `acme-corp`, `ldap-users`) enable separate identity stores per domain
- Domain admins can manage only their domain's resources
- Domain names must be unique within a Keystone deployment

### Projects

A **project** (historically called "tenant") is the unit of resource ownership. All OpenStack resources (VMs, volumes, networks, etc.) are scoped to a project.

- Projects belong to exactly one domain
- Projects can be **nested** (hierarchical multitenancy): a project can have a parent project
- Nested projects share the same domain as the root project
- Project tree depth is limited to avoid performance issues (typically ≤ 5 levels)
- `is_domain=True` projects represent domains in the hierarchical model

### Users

A **user** is an identity that can authenticate to Keystone. Users belong to one domain and can be assigned roles in multiple projects/domains.

- Users authenticate with: password, application credential, token, or federated assertion
- A user with `enabled=False` cannot authenticate
- Passwords are stored hashed (bcrypt by default) in the SQL identity backend
- LDAP-backed users are authenticated against the LDAP directory; Keystone does not store their passwords

### Groups

A **group** is a collection of users within a domain. Role assignments can be made to groups: every member of the group inherits those role assignments.

- Group membership is managed through `openstack group add user`
- Federated users can be auto-assigned to groups via mapping rules
- Groups cannot span domains

### Roles

A **role** is a named marker that expresses permissions within a scope (project, domain, or system). Keystone ships with default roles:

| Role | Scope | Purpose |
|---|---|---|
| `admin` | system, domain, project | Full administrative access at that scope |
| `member` | project | Standard project user — create/use resources |
| `reader` | project, domain, system | Read-only access |

**Implied roles**: Roles can imply other roles. The default hierarchy is:
```
admin → member → reader
```
Assigning `admin` automatically grants `member` and `reader` access.

### Role Assignments

A **role assignment** binds a (user or group) + a role + a scope (project, domain, or system).

- `openstack role add --project engineering --user alice member`
- `openstack role add --domain Default --user bob admin`
- `openstack role add --system all --user svcaccount reader`

Role assignments are stored in the `assignment` table. The effective assignments include both direct and inherited assignments.

**Inherited assignments**: A role assignment on a domain with `--inherited` propagates to all projects in that domain. Useful for domain-wide access.

## RBAC Enforcement Model

Keystone 2025.x uses **oslo.policy** with a **scope-aware, default-safe policy model** introduced in Wallaby and fully enforced in later releases.

### Scopes

| Scope | Description | Token type |
|---|---|---|
| `system` scope | Operations that affect the entire cloud (e.g., managing service catalog) | `system`-scoped token |
| `domain` scope | Operations within one domain (e.g., managing domain users) | `domain`-scoped token |
| `project` scope | Operations within one project (e.g., launching VMs) | `project`-scoped token |

A token carries exactly one scope. Multi-scoped operations require multiple tokens.

### Policy Enforcement

- **Personas**: `system admin`, `system reader`, `domain admin`, `domain reader`, `project admin`, `project member`, `project reader`
- `enforce_scope = True` in `[oslo_policy]` enables scope checking (mandatory in 2025.1+)
- `enforce_new_defaults = True` uses the new role hierarchy (replaces legacy `is_admin_project`)
- Policy YAML overrides: `/etc/keystone/policy.yaml` — only specify deviations from defaults

```ini
# /etc/keystone.conf (oslo_policy section)
[oslo_policy]
enforce_scope = true
enforce_new_defaults = true
```

## Token Providers

### Fernet Tokens (Default)

Fernet tokens are **bearer tokens** using symmetric encryption (AES-128-CBC + HMAC-SHA256 via the `cryptography` library). They are:

- **Not persisted** in the database — Keystone does not maintain a token table
- **Self-contained**: the token payload encodes the user ID, project/domain/system scope, expiry, and audit IDs
- **Bounded in size**: typically 200–300 bytes, safe for HTTP headers
- **Rotatable**: keys are rotated without invalidating existing tokens (using staged/primary/secondary key scheme)

Fernet token payload fields:
- Version byte (0x80)
- Timestamp (8 bytes, seconds since epoch)
- IV (16 bytes, random)
- Ciphertext (variable, AES-CBC encrypted payload)
- HMAC (32 bytes, SHA-256)

Token expiry is enforced during validation by checking the embedded timestamp against `[token] expiration` (default: 3600 seconds).

### JWS Tokens

JWS (JSON Web Signature) tokens are an alternative provider introduced in Stein:

- Asymmetric signing: **ES256** (ECDSA with P-256 curve and SHA-256)
- **Not persisted** like Fernet
- Public keys distributed to all Keystone nodes for validation
- Larger than Fernet tokens (JWS overhead adds base64url + JSON structure)
- Key setup: `keystone-manage jws_setup`

Choose Fernet for most deployments. JWS is useful when you need asymmetric verification (e.g., offline token inspection).

### Token Revocation

Since tokens are not stored, revocation works via **revocation events** persisted in the DB. On validation, Keystone checks the token's audit chain against the revocation list. The revocation list is cached.

- `openstack token revoke <token_id>` — revoke a specific token
- Fernet tokens have a short expiry (default 1 hour), which limits the revocation window

## Application Credentials

Application credentials allow users to create **delegated credentials** tied to their role assignments at the time of creation. They are used by automation scripts and services without exposing the user's password.

- Bound to the user and the project at creation time
- Can have an expiry (`--expiration 2025-12-31T00:00:00`)
- Can be restricted to specific roles (subset of user's roles)
- Can be restricted to specific API operations via `--access-rules`
- **Cannot be used to create new application credentials** (prevents privilege escalation)
- Stored encrypted in the `application_credential` table

```bash
openstack application credential create deploy-robot \
  --role member \
  --expiration "2025-12-31T00:00:00" \
  --description "CI/CD pipeline credential"
```

The response includes a `secret` (shown once). Store it immediately.

## Service Catalog

The service catalog is the registry of OpenStack API endpoints. Every other service registers itself here after deployment.

### Services

A **service** represents a type of API:

| Type | Name | Description |
|---|---|---|
| `identity` | keystone | Identity service |
| `compute` | nova | Compute service |
| `network` | neutron | Networking service |
| `volume` / `volumev3` | cinder | Block storage |
| `image` | glance | Image service |
| `object-store` | swift | Object storage |
| `placement` | placement | Placement API |
| `orchestration` | heat | Orchestration |
| `load-balancer` | octavia | Load balancing |

### Endpoints

Each service has up to three **endpoints** per region:

| Interface | Purpose | Consumers |
|---|---|---|
| `public` | External-facing URL | End users, CLI clients |
| `internal` | Internal network URL | Inter-service communication |
| `admin` | Administrative operations | Operators only |

Example endpoint set for Nova in region `RegionOne`:
- public: `http://10.0.0.10:8774/v2.1`
- internal: `http://10.0.0.11:8774/v2.1`
- admin: `http://10.0.0.11:8774/v2.1`

### Regions

Regions represent independent OpenStack deployments sharing the same Keystone. Multi-region setups register separate endpoint sets per region.

## Credential Storage

Keystone stores user credentials (e.g., EC2-compatible access/secret key pairs) in the `credential` table. All credential blobs are **AES-256-GCM encrypted** using keys from `/etc/keystone/credential-keys/`.

- Credential keys are separate from Fernet keys
- Setup: `keystone-manage credential_setup`
- Rotation: `keystone-manage credential_rotate`
- After rotation, re-encrypt existing credentials: `keystone-manage credential_migrate`

## Trust Delegation

A **trust** allows a trustor (user A) to delegate a subset of their roles on a project to a trustee (user B or service account). Used heavily by Heat for stack operations.

- Trusts are scoped to a single project
- The delegated roles must be a subset of the trustor's roles
- Trusts can be **impersonating** (the trustee acts as the trustor) or **non-impersonating**
- Optional: `remaining_uses` limits the number of times the trust can be consumed
- `allow_redelegation = True` permits the trustee to further delegate (disabled by default)

```bash
# Create a trust (trustor: alice delegates member role on engineering to heat-svc)
openstack trust create \
  --trustor alice \
  --trustee heat-svc \
  --role member \
  --project engineering \
  --impersonation
```
