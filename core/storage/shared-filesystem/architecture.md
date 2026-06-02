# Manila Architecture

## Service Topology

Manila is a distributed service composed of four processes. They communicate over oslo.messaging (RabbitMQ by default) and all read/write a shared MariaDB database.

```
             ┌──────────────────────────────────────────────────────────┐
             │                     Manila API Tier                       │
             │   ┌──────────────────────────────────────────────────┐   │
             │   │             manila-api (WSGI)                     │   │
             │   │   ┌─────────────┐   ┌──────────────────────────┐ │   │
             │   │   │  REST Router│→  │  Request Handler + Auth  │ │   │
             │   │   └─────────────┘   └──────────────────────────┘ │   │
             │   └──────────────────────────────────────────────────┘   │
             └──────────────────────┬───────────────────────────────────┘
                                    │ oslo.messaging RPC
                 ┌──────────────────┼──────────────────────┐
                 │                  │                       │
      ┌──────────▼──────────┐  ┌────▼─────────────┐  ┌────▼────────────────┐
      │  manila-scheduler   │  │  manila-share     │  │  manila-data        │
      │                     │  │  (one per         │  │                     │
      │  Selects a share    │  │   backend node)   │  │  Handles cross-     │
      │  service node for   │  │                   │  │  backend data copy  │
      │  new share requests │  │  Drives the       │  │  during migration   │
      │                     │  │  backend driver   │  │                     │
      └─────────────────────┘  └──────────┬────────┘  └─────────────────────┘
                                           │
                            ┌──────────────┼──────────────┐
                            │              │               │
                     ┌──────▼───┐   ┌──────▼───┐   ┌──────▼───┐
                     │  NFS/    │   │  CephFS  │   │  NetApp  │
                     │  Samba   │   │  driver  │   │  driver  │
                     │  driver  │   └──────────┘   └──────────┘
                     └──────────┘
```

### manila-api

The API service is a **WSGI application** (served by Apache httpd or uWSGI) that:

- Exposes the Manila REST API at port `8786` (v2 only; v1 was removed in Stein)
- Validates requests against Keystone tokens via `keystonemiddleware`
- Enforces RBAC using `oslo.policy`
- Writes metadata to the database (share records, access rules, type definitions)
- Publishes RPC calls to `manila-scheduler` (for creates) and `manila-share` (for other operations)
- Multiple instances can run behind a load balancer for HA

Key files:
- `/etc/manila/manila.conf` — primary configuration
- `/etc/manila/policy.yaml` — RBAC overrides (defaults are in code)
- `/var/log/manila/manila-api.log` — API log

### manila-scheduler

The scheduler selects **which manila-share node** should handle a new share creation. It:

- Receives `create_share` RPC calls from `manila-api`
- Runs a filter/weigh pipeline against the live capability reports from share nodes
- Dispatches the request to the selected `manila-share` node via RPC
- Does **not** communicate with the storage backend directly

The scheduler uses a pluggable filter pipeline (see [internals.md](internals.md) for filter details). Capability data is periodically reported by each `manila-share` node into the database, and the scheduler reads this to make placement decisions.

### manila-share

The share service is the **backend driver executor**. There is typically one `manila-share` process per storage backend or per host. It:

- Receives RPC calls from the scheduler (create) and the API (delete, extend, access, snapshot, etc.)
- Instantiates and calls the configured storage driver
- Manages the **share server** lifecycle when `driver_handles_share_servers = True`
- Reports capacity and capability statistics back to the scheduler periodically
- Runs as a long-lived daemon; one process can manage one or more backends (using the `enabled_share_backends` config option)

Key files:
- `/var/log/manila/manila-share.log` — share service log

### manila-data

The data service handles **data-plane operations** that require copying share contents between backends. It:

- Executes `migration_start` / `migration_complete` tasks for shares being migrated between incompatible backends
- Mounts both source and destination shares locally and performs a byte-level copy
- Tracks progress and reports it through the migration API
- Runs independently of `manila-share`; typically one instance per cloud

The data service is **optional** if you only use driver-assisted migration (where the storage hardware handles the copy natively), but is required for generic host-assisted migration.

## Share Types

A **share type** is a named set of capabilities that represents a class of storage. Share types serve two purposes:

1. **Scheduling**: The extra_specs on a share type are matched against backend capability reports to route shares to the correct backend.
2. **Driver behavior**: Certain extra_specs directly configure driver behavior (e.g., `driver_handles_share_servers`, `snapshot_support`).

### Required Extra Spec

Every share type must set `driver_handles_share_servers` (DHSS):

```
driver_handles_share_servers = True   # DHSS=true mode
driver_handles_share_servers = False  # DHSS=false mode
```

This is not optional — it controls a fundamental operating mode difference (see Driver Modes below).

### Common Extra Specs

| Key | Example Value | Meaning |
|---|---|---|
| `driver_handles_share_servers` | `True` / `False` | DHSS mode (required) |
| `snapshot_support` | `True` / `False` | Whether the backend supports snapshots |
| `create_share_from_snapshot_support` | `True` / `False` | Whether shares can be created from snapshots |
| `revert_to_snapshot_support` | `True` / `False` | Whether a share can be reverted in-place to a snapshot |
| `mount_snapshot_support` | `True` / `False` | Whether snapshots can be mounted directly |
| `replication_type` | `readable` / `writeable` / `dr` | Replication model supported |
| `availability_zones` | `az1,az2` | Restrict share type to specific AZs |
| `provisioning:max_share_size` | `1024` | Maximum share size in GiB |

### Default Share Type

Manila ships with a configurable default share type. When a user creates a share without specifying a type, the default is used. Configure it in `manila.conf`:

```ini
[DEFAULT]
default_share_type = default
```

## Driver Modes: DHSS=true vs DHSS=false

This is the most important architectural distinction in Manila. It determines who manages the network infrastructure that exports the shares.

### DHSS=true (Driver Handles Share Servers)

In this mode, Manila **manages the full lifecycle of share server infrastructure**:

- Manila creates a **share server** (typically a VM or network appliance) per share network association
- Manila allocates Neutron ports on the tenant's network for the share server
- The share server exports shares directly onto the tenant's Neutron network
- Tenants can use their own isolated networks; Manila bridges them via the share server

```
Tenant Network (Neutron)
    10.0.1.0/24
         │
    ┌────┴─────────────────────────────┐
    │       Share Server (VM/appliance) │
    │   manila-share manages lifecycle  │
    │   exports: 10.0.1.10:/vol/share1  │
    └───────────────────────────────────┘
         │
    ┌────┴────────────────┐
    │  Instance A          │  mounts: 10.0.1.10:/vol/share1
    │  Instance B          │  mounts: 10.0.1.10:/vol/share1
    └─────────────────────┘
```

**Share network required**: Before creating a share with DHSS=true, the tenant must create a share network that references their Neutron network and subnet.

**Drivers that use DHSS=true**: Generic (reference) driver, Hitachi NAS, some ONTAP configurations.

### DHSS=false (Backend Manages Its Own Network)

In this mode, Manila **does not manage share servers**. The storage backend already has a pre-configured NFS/CIFS server reachable on the tenant network. Manila only instructs the backend to create/delete exports and manage access rules.

```
Pre-existing storage backend
    (NetApp, Pure, CephFS, existing NFS server)
         │ exports via fixed IP, e.g. 192.168.100.5
         │
    ┌────┴────────────────┐
    │  Instance A          │  mounts: 192.168.100.5:/shares/share1
    │  Instance B          │  mounts: 192.168.100.5:/shares/share1
    └─────────────────────┘
```

**No share network required**: Tenants do not need to specify a share network. The backend is already connected to the appropriate network.

**Drivers that use DHSS=false**: CephFS driver, NetApp ONTAP (NFS/CIFS without SVM management), LVM driver, GlusterFS driver, most hardware appliance drivers.

### Choosing a Mode

| Consideration | DHSS=true | DHSS=false |
|---|---|---|
| Tenant network isolation | Full isolation per share network | Shared backend network |
| Setup complexity | Higher (Neutron integration required) | Lower |
| Share server management | Manila creates/destroys servers | Pre-provisioned by admin |
| Multi-tenancy | Strong; each tenant's shares on their network | Weaker; all exports on same backend network |
| Suitable for | Public clouds, strict isolation | Private clouds, existing NAS infrastructure |

## Share Networks

A **share network** is the tenant-side object that links a Neutron network+subnet to Manila so that share servers can be attached to it (DHSS=true mode only).

```
share_network
  ├── neutron_net_id     → Neutron network UUID
  ├── neutron_subnet_id  → Neutron subnet UUID
  └── share_network_subnets[]
        └── share_servers[]  (one per backend that has allocated here)
```

- One share network can span multiple backends (a share network subnet is created per backend)
- Share servers are created lazily on first share creation within a share network
- Share servers can be shared among multiple shares on the same share network (configurable)

## Share Groups

A **share group** is a collection of shares that can be snapshotted atomically. It is analogous to a Cinder consistency group.

- All shares in a group must use the same share type (or types within the group's share group type)
- A share group snapshot captures all member shares simultaneously
- New shares can be created from a share group snapshot
- The backend must support `share_group_snapshot_support` to use this feature
- Share group types define which share types are allowed within the group

```bash
# Typical workflow
openstack share group type create my-group-type default  # create group type referencing share type
openstack share group create --share-group-type my-group-type --share-type default my-group
openstack share create --share-group my-group --share-type default NFS 10
openstack share group snapshot create my-group --name my-group-snap
```

## Share Replicas

Share replication provides high availability and disaster recovery across availability zones or backends. Three replication models are supported:

### Replication Types

| Type | Description | Use Case |
|---|---|---|
| `dr` (Disaster Recovery) | One active replica; replicas are not mountable until promoted | DR failover; secondary is kept warm but not accessible |
| `readable` | One active (read-write) replica; secondaries are read-only mountable | Read scaling; CDN-style workloads |
| `writable` | All replicas are simultaneously writable | Distributed write workloads; requires backend coordination |

### Replica Lifecycle

```
create_share  →  [active replica created]
                       │
replica_create  →  [secondary replica created, state=out_of_sync]
                       │
              periodic sync  →  [state=in_sync]
                       │
         replica_promote  →  [secondary becomes active, old active becomes secondary]
```

- The `manila-share` service on each backend node manages its local replicas
- Replica sync is backend-driven (Manila does not copy data; the backend replicates)
- Replica states: `in_sync`, `out_of_sync`, `active`, `error`

## Share Migration

Manila supports two migration paths:

### Driver-Assisted Migration

The storage driver handles the data copy natively (e.g., volume replication at the array level). This is:
- Faster (no data leaves the storage array)
- Transparent to clients if the driver supports non-disruptive migration
- Available only when source and destination backends are both capable and are from the same vendor/driver

### Host-Assisted Migration (via manila-data)

Manila mounts both the source and destination shares and copies data using the `manila-data` service:
- Works across any two backends
- Requires temporary network access from the `manila-data` host to both shares
- Clients must quiesce I/O during the cutover phase
- Migration states: `migrating`, `migrating_to`, `data_copying_starting`, `data_copying_in_progress`, `data_copying_completing`, `data_copying_completed`, `data_copying_error`

### Migration Process

```
migration_start (source share)
    │
    ├─ [driver-assisted] → driver copies data → migration_complete
    │
    └─ [host-assisted]
           │
           manila-data mounts source + destination
           │
           data copy loop
           │
           migration_complete (cutover: unmap source, swap export)
```

## Inter-Service Dependencies

| Service | How Manila Uses It |
|---|---|
| Keystone | Token validation via keystonemiddleware; service catalog registration |
| Neutron | Port allocation and network queries for DHSS=true share server setup |
| Nova | Spawning share server VMs (generic driver only) |
| Cinder | Creating share server root volumes (generic driver only) |
| oslo.messaging | All internal RPC (api→scheduler, scheduler→share, api→data) |
| oslo.db | Database access (SQLAlchemy ORM) |
| oslo.policy | RBAC enforcement on all API calls |
| oslo.cache | Caching share type and capability data |
