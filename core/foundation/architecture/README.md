# OpenStack Architecture

Comprehensive architectural reference for OpenStack. Covers service relationships, communication patterns, authentication flow, and deployment topology.

---

## Overview

OpenStack is a collection of loosely coupled, independently deployable services that together provide Infrastructure-as-a-Service (IaaS). Each service exposes a RESTful HTTP API and is identified by a **project name** (e.g., Nova, Neutron). Services communicate with one another either synchronously over HTTP or asynchronously over a message bus (RabbitMQ).

All API requests are authenticated by Keystone. Services do not maintain their own user databases — they delegate token validation to Keystone middleware. This middleware intercepts every inbound API call, validates the bearer token, and injects the caller's identity (project, user, roles) into the request context before the service handler runs.

The data plane (actual VM traffic, storage I/O) is entirely separate from the control plane (API calls, scheduling, orchestration). A control-plane outage does not kill running instances, but prevents new operations.

---

## Service Map

| Service Name | Project | Default Port | Purpose |
|---|---|---|---|
| Identity | Keystone | 5000 | Authentication, authorization, service catalog, token issuance |
| Compute | Nova | 8774 | Virtual machine lifecycle management, scheduling |
| Networking | Neutron | 9696 | Software-defined networking, routers, security groups, floating IPs |
| Block Storage | Cinder | 8776 | Persistent block volumes, snapshots, backups |
| Image | Glance | 9292 | VM image registry, upload/download, format conversion |
| Object Storage | Swift | 8080 | Scalable object store, HTTP-accessible, eventually consistent |
| Shared Filesystem | Manila | 8786 | NFS/CIFS shared filesystem provisioning |
| Orchestration | Heat | 8004 | Stack-based resource orchestration via HOT/CFN templates |
| Container Infra | Magnum | 9511 | Kubernetes/Swarm cluster provisioning on top of Nova |
| Load Balancing | Octavia | 9876 | L4/L7 load balancer provisioning via amphora VMs or OVN |
| Key Management | Barbican | 9311 | Secrets, certificates, symmetric keys, HSM integration |
| DNS | Designate | 9001 | DNS zone and recordset management, backend driver abstraction |
| Bare Metal | Ironic | 6385 | Physical server provisioning, PXE, IPMI, Redfish |
| Dashboard | Horizon | 80/443 | Web UI over OpenStack APIs; does not have its own API |

**Placement** (port 8778) is a sub-service of the compute domain that tracks resource inventories and allocations. It is required by Nova and Blazar and may be consumed by other services.

---

## Shared Infrastructure

All OpenStack services depend on three shared infrastructure components.

### Message Queue — RabbitMQ

RabbitMQ (AMQP 0-9-1) is the default message transport. Services use **oslo.messaging** to send RPCs and notifications without coupling directly to a transport implementation.

- **RPC calls** (`cast` and `call`): intra-service communication between API and worker processes (e.g., `nova-api` → `nova-conductor` → `nova-compute`).
- **Notifications**: events emitted by services (e.g., `compute.instance.create.end`) consumed by Ceilometer, Aodh, or external consumers.

Default ports: 5672 (AMQP), 15672 (management UI), 5671 (AMQPS/TLS).

Alternative transports supported by oslo.messaging: **Kafka** (notifications only), **ZeroMQ** (experimental), **fake** (testing).

### Database — MariaDB / PostgreSQL

Each service maintains its own schema in the shared RDBMS cluster. No cross-service joins occur at runtime; services communicate via API, not by reading each other's tables.

Recommended: **MariaDB 10.6+** (Galera for HA). PostgreSQL is supported but less commonly deployed in production.

Each service uses **oslo.db** which wraps SQLAlchemy, handles connection pooling, retries, and schema migrations (Alembic).

### Cache — Memcached / Redis

Used to cache Keystone tokens, reducing per-request validation overhead. Also used by some services for rate limiting and session state.

- **Memcached**: simplest, most common. Stateless, no persistence.
- **Redis**: supports persistence and Pub/Sub; used by some deployments for distributed locking (Tooz).

---

## API Patterns

### RESTful Design

OpenStack APIs follow REST conventions:

```
GET    /v2/servers           # list resources
POST   /v2/servers           # create resource
GET    /v2/servers/{id}      # show resource
PUT    /v2/servers/{id}      # full update
PATCH  /v2/servers/{id}      # partial update
DELETE /v2/servers/{id}      # delete resource
```

All APIs return JSON. Request bodies are JSON. Content-Type is `application/json`.

### Microversions

Nova, Cinder, Glance, and Manila use **microversions** to add backwards-compatible API changes without a new major version.

- Client requests a microversion via header: `X-OpenStack-Nova-Microversion: 2.95`
- Server returns the version it honored in the response header.
- If no header is sent, the server responds at the minimum supported microversion.
- Use `openstack microversion list --service compute` to query the supported range.

### Request / Response Format

Typical request:

```http
POST /v2/servers HTTP/1.1
Host: nova.example.com:8774
X-Auth-Token: gAAAAABl...
Content-Type: application/json

{
  "server": {
    "name": "web-01",
    "flavorRef": "m1.small",
    "imageRef": "5e4c5e4c-...",
    "networks": [{"uuid": "a1b2c3..."}]
  }
}
```

Typical response:

```http
HTTP/1.1 202 Accepted
Content-Type: application/json

{
  "server": {
    "id": "7b3e9f2a-...",
    "status": "BUILD",
    "links": [{"rel": "self", "href": "http://nova.example.com:8774/v2/servers/7b3e9f2a-..."}]
  }
}
```

Error responses use standard HTTP status codes (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 500 Internal Server Error) with a JSON body:

```json
{
  "badRequest": {
    "message": "Invalid flavorRef provided.",
    "code": 400
  }
}
```

---

## Authentication Flow

Every API call goes through this sequence:

```
Client                  Keystone               Target Service
  |                        |                        |
  |-- POST /v3/auth/tokens -->                       |
  |   (credentials/app cred)                        |
  |<-- 201 + X-Subject-Token + service catalog --   |
  |                        |                        |
  |-- GET /v2/servers ----+-----------------------> |
  |   (X-Auth-Token: <token>)                       |
  |                        |                 [keystonemiddleware]
  |                        |<-- GET /v3/auth/tokens/{token} (cache miss)
  |                        |-- 200 + token data -->  |
  |                        |                 [inject: project_id, roles, user_id]
  |                        |                 [service handler runs]
  |<---------------------------------------- 200 response
```

**keystonemiddleware** (`keystonemiddleware.auth_token`) sits in the WSGI pipeline of every service. It:

1. Reads the `X-Auth-Token` header.
2. Looks up the token in its local cache (Memcached).
3. On cache miss, calls `GET /v3/auth/tokens/{token_id}` on Keystone.
4. Validates expiry, project scope, and domain.
5. Injects `HTTP_X_PROJECT_ID`, `HTTP_X_USER_ID`, `HTTP_X_ROLES`, etc. into the WSGI environment.
6. The service's policy engine then enforces RBAC against those roles.

Token types:
- **Fernet tokens** (default since Ocata): cryptographically signed, no server-side persistence, ~250 bytes. Rotation via `keystone-manage fernet_rotate`.
- **JWS tokens**: JWT-based alternative to Fernet.

---

## Service Catalog

The service catalog is returned inside the authentication response token. It lists every registered endpoint.

```json
{
  "catalog": [
    {
      "type": "compute",
      "name": "nova",
      "endpoints": [
        {
          "interface": "public",
          "region": "RegionOne",
          "url": "https://nova.example.com:8774/v2.1"
        },
        {
          "interface": "internal",
          "region": "RegionOne",
          "url": "http://10.0.0.10:8774/v2.1"
        },
        {
          "interface": "admin",
          "region": "RegionOne",
          "url": "http://10.0.0.10:8774/v2.1"
        }
      ]
    }
  ]
}
```

### Endpoint Interface Types

| Interface | Audience | Network |
|---|---|---|
| `public` | External clients, end users | Public / internet-reachable |
| `internal` | Inter-service communication | Management/internal network |
| `admin` | Operators, administrative operations | Management/admin network |

Services use the `internal` interface when calling other services. External clients (CLI, dashboard, applications) use `public`. Some services expose additional admin-only endpoints — historically on port +1 (e.g., Keystone admin was 35357; consolidated to 5000 since Stein).

### Service Discovery

Clients call `openstack endpoint list` to see the catalog. Services look up peer endpoint URLs at startup from the catalog, configured in their config files under `[<service>] www_authenticate_uri` and similar options. Do not hardcode service URLs — always derive from the catalog.

---

## Oslo Libraries

Oslo ("OpenStack Library") is the set of shared Python libraries used across all services to avoid duplicating common infrastructure code.

### oslo.config

Parses configuration files and CLI options. Every OpenStack service uses oslo.config to load its `*.conf` file.

- INI format with `[SECTION]` headings and `key = value` pairs.
- Options declared in code; unknown options in config files are ignored (with a warning).
- Config generator: `oslo-config-generator --config-file etc/nova/nova-config-generator.conf`

### oslo.messaging

Abstracts the message transport (RabbitMQ, Kafka, fake). Provides:

- `RPCClient` / `RPCServer`: request-reply RPC.
- `Notifier` / `NotificationListener`: event notifications.
- Transport URL format: `rabbit://user:pass@host:5672/vhost`

### oslo.db

SQLAlchemy wrapper providing:

- Connection URL management.
- Automatic retry on deadlocks (`@oslo_db.api.wrap_db_retry`).
- Schema migration via Alembic (`nova-manage db sync`).
- Reader/writer session routing for Galera active-active clusters.

### oslo.policy

RBAC policy enforcement:

- Reads `policy.yaml` (or `policy.json` legacy) from the service config dir.
- Rules expressed as role checks, token attribute checks, or logical combinations.
- Enforced via `policy.enforce(credentials, action, target)` calls in service code.
- `oslopolicy-checker` CLI for testing rules without running the service.

### oslo.log

Logging configuration wrapper:

- Sets up Python `logging` consistently across services.
- Adds request IDs (`req-<uuid>`) and instance IDs to log context for correlation.
- Configured via `[DEFAULT]` section (`log_file`, `log_dir`, `debug`, `verbose`).

---

## Typical Deployment Topology

### Node Types

**Controller Node**
Runs: API services, schedulers, conductors, Keystone, Glance, Cinder API, Neutron server, Heat API, Horizon, MariaDB, RabbitMQ, Memcached.
Does not run: hypervisor, data-plane networking agents (except in all-in-one).

**Compute Node**
Runs: `nova-compute`, Neutron L2 agent (OVS or OVN), `nova-virtlogd`, libvirt/KVM.
Does not run: database, message queue, API services.

**Network Node** (traditional OVS model)
Runs: `neutron-l3-agent`, `neutron-dhcp-agent`, `neutron-metadata-agent`, OVS.
In OVN deployments, routing is distributed; a dedicated network node is optional.

**Storage Node**
Runs: Cinder volume service + backend driver, Swift storage server (account/container/object), or Ceph OSDs (external to OpenStack but providing RBD backend).

### Minimum Production Topology

```
[ Controller x3 ] -- HA pair, Galera cluster, RabbitMQ cluster, HAProxy/VIP
[ Compute x N   ] -- Hypervisor nodes, scales horizontally
[ Storage x N   ] -- Ceph cluster or Cinder LVM/NFS (scales separately)
```

---

## Inter-Service Communication

### Synchronous — REST API

Services call each other's APIs over HTTP when the caller needs an immediate response. Examples:

- Nova calls Neutron to create a port when launching an instance.
- Nova calls Cinder to attach a volume.
- Nova calls Glance to get image metadata before scheduling.
- Keystone is called by every service's middleware on token validation (cache miss).

Calls go over the `internal` endpoint interface (management network). Libraries: `keystoneauth1` handles token acquisition and endpoint URL lookup automatically.

### Asynchronous — RPC over Message Bus

Within a single service, components communicate via oslo.messaging RPC over RabbitMQ. This decouples the API tier from worker tiers and enables horizontal scaling.

| Service | Async path |
|---|---|
| Nova | `nova-api` → RabbitMQ → `nova-conductor` → `nova-compute` |
| Cinder | `cinder-api` → RabbitMQ → `cinder-scheduler` → `cinder-volume` |
| Neutron | `neutron-server` → RabbitMQ → `neutron-l3-agent` / `neutron-dhcp-agent` |
| Heat | `heat-api` → RabbitMQ → `heat-engine` |

Call types:
- `cast`: fire-and-forget. API returns 202 Accepted; work happens asynchronously.
- `call`: blocking RPC. Caller waits for return value. Used for synchronous work within a component.

---

## Architecture Diagram

```
                        ┌─────────────────────────────────────────────────────┐
                        │                  KEYSTONE (5000)                     │
                        │    Auth · Token Validation · Service Catalog         │
                        └──────────────────────┬──────────────────────────────┘
                                               │  keystonemiddleware (every service)
                         ┌─────────────────────┼─────────────────────────────┐
                         │                     │                             │
              ┌──────────▼──────┐   ┌──────────▼──────┐   ┌─────────────────▼───┐
              │  NOVA API (8774) │   │NEUTRON API (9696)│   │  GLANCE API (9292)  │
              │  nova-scheduler  │   │ neutron-server   │   │  glance-registry    │
              │  nova-conductor  │   │                  │   │                     │
              └──────────┬──────┘   └──────────┬───────┘   └─────────────────────┘
                         │                     │
              ┌──────────▼──────────────────────▼──────────────────────────────────┐
              │                      RABBITMQ  (5672)                               │
              │              Message Bus — RPC + Notifications                      │
              └──────┬────────────────┬──────────────────┬──────────────────────────┘
                     │                │                  │
          ┌──────────▼───┐  ┌─────────▼────┐   ┌────────▼────────────┐
          │ nova-compute  │  │ neutron-l3-  │   │  nova-conductor     │
          │  (libvirt/KVM)│  │ agent/dhcp   │   │  (DB orchestration) │
          └──────────────┘  └─────────┬────┘   └────────┬────────────┘
                                      │                  │
              ┌────────────────────────▼──────────────────▼──────────────────────┐
              │                  MARIADB / POSTGRESQL                              │
              │        nova · neutron · glance · cinder · keystone · ...           │
              └────────────────────────────────────────────────────────────────────┘

Additional services sit alongside Nova/Neutron:
  CINDER (8776) · SWIFT (8080) · MANILA (8786) · HEAT (8004)
  MAGNUM (9511) · OCTAVIA (9876) · BARBICAN (9311)
  DESIGNATE (9001) · IRONIC (6385)

All services:
  - Authenticate inbound requests via Keystone middleware
  - Publish events to RabbitMQ (oslo.messaging notifications)
  - Persist state to MariaDB (oslo.db / SQLAlchemy)
  - Cache tokens in Memcached
```

---

## Cross-Reference

- Deployment methods → `core/foundation/deployment-overview/`
- Configuration patterns → `core/foundation/configuration/`
- Keystone internals → `core/identity/`
- Nova scheduling and compute → `core/compute/`
- Neutron ML2 and OVN → `core/networking/`
