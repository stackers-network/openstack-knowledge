# Manila Internals

## Driver Interface

All Manila storage drivers inherit from `manila.share.driver.ShareDriver`. The driver interface defines the contract between `manila-share` and the storage backend. Each method maps to a specific share lifecycle operation.

### Core Share Methods

```python
class ShareDriver:

    def create_share(self, context, share, share_server=None):
        """
        Provision a new share on the backend.

        Args:
            context: RequestContext with auth info
            share: Share object (dict-like) with keys:
                   id, size, share_proto, share_network_id, etc.
            share_server: ShareServer object (DHSS=true only); contains
                          the backend_details dict with IPs and credentials
                          populated during share server setup.

        Returns:
            list of export location dicts, each containing:
                {'path': 'host:/export/path',
                 'is_admin_only': False,
                 'metadata': {...}}
        """

    def delete_share(self, context, share, share_server=None):
        """
        Delete the share from the backend and release storage.
        Must be idempotent — safe to call on an already-deleted share.
        """

    def allow_access(self, context, share, access, share_server=None):
        """
        Grant a client access to the share.

        access dict keys:
            access_type: 'ip', 'user', 'cert', 'cephx'
            access_to:   CIDR, username, cert CN, or CephX client name
            access_level: 'rw' or 'ro'
        """

    def deny_access(self, context, share, access, share_server=None):
        """
        Revoke a previously granted access rule.
        Must be idempotent — safe if the rule no longer exists.
        """
```

### Snapshot Methods

```python
    def create_snapshot(self, context, snapshot, share_server=None):
        """
        Create a point-in-time snapshot of a share.

        snapshot dict contains:
            id, share_id, share_size, share_proto

        Returns: dict of provider_location or None
        """

    def delete_snapshot(self, context, snapshot, share_server=None):
        """
        Delete a snapshot. Must be idempotent.
        """

    def create_share_from_snapshot(self, context, share, snapshot,
                                   share_server=None, parent_share=None):
        """
        Create a new share cloned/hydrated from a snapshot.
        Returns: list of export location dicts (same as create_share).
        """

    def revert_to_snapshot(self, context, snapshot, share_access_rules,
                           snapshot_access_rules, share_server=None):
        """
        Revert the share in-place to the given snapshot state.
        All data written after the snapshot is destroyed.
        """
```

### Manage/Unmanage

```python
    def manage_existing(self, share, driver_options):
        """
        Take control of an existing export that was created outside Manila.

        driver_options: dict of backend-specific hints from the operator
        Returns: dict with 'size' (GiB) and 'export_locations'
        """

    def unmanage(self, share):
        """
        Release Manila's management of a share without deleting it.
        """
```

### Capacity and Capability Reporting

```python
    def _update_share_stats(self):
        """
        Called periodically (default every 60 seconds) by the share manager.
        Must populate self._stats dict with:
        {
            'share_backend_name': 'CEPHFS',
            'vendor_name': 'Red Hat',
            'driver_version': '1.0',
            'storage_protocol': 'NFS',
            'total_capacity_gb': 10000.0,
            'free_capacity_gb': 7500.0,
            'reserved_percentage': 0,
            'snapshot_support': True,
            'create_share_from_snapshot_support': True,
            'revert_to_snapshot_support': False,
            'replication_type': None,
            'driver_handles_share_servers': False,
            'pools': [
                {
                    'pool_name': 'ceph-pool-0',
                    'total_capacity_gb': 10000.0,
                    'free_capacity_gb': 7500.0,
                    'allocated_capacity_gb': 2500.0,
                    'reserved_percentage': 0,
                    'snapshot_support': True,
                }
            ]
        }
        """
```

Capability stats flow:

```
manila-share (driver._update_share_stats)
    │  every 60s via share_manager.py periodic task
    ▼
DB: share_stats table (or in-memory cache)
    │
    ▼
manila-scheduler reads stats when evaluating filters
```

### Extend and Shrink

```python
    def extend_share(self, share, new_size, share_server=None):
        """
        Increase share size to new_size GiB.
        Backend must resize the underlying filesystem/quota.
        """

    def shrink_share(self, share, new_size, share_server=None):
        """
        Decrease share size to new_size GiB.
        Backend must verify current usage <= new_size before shrinking.
        Raise ShareShrinkingPossibleDataLoss if current usage > new_size.
        """
```

## Share Server Lifecycle (DHSS=true)

In DHSS=true mode, `manila-share` manages the full lifecycle of share server infrastructure. The generic driver is the reference implementation; hardware drivers implement analogous steps.

### Share Server Creation

Triggered when a share is first created on a share network that has no active share server for that backend.

```
User: openstack share create --share-network share-net NFS 10
    │
manila-api: write share record (status=creating) → RPC to scheduler
    │
manila-scheduler: select manila-share host → RPC to manila-share
    │
manila-share.share_manager.create_share():
    │
    ├─ 1. Check for existing share server on this (share_network, host) pair
    │      └─ If none: call _setup_server()
    │
    ├─ _setup_server():
    │      │
    │      ├─ 2. Allocate Neutron ports
    │      │       manila calls Neutron API:
    │      │       POST /v2.0/ports  (on tenant's neutron_net_id)
    │      │       → gets port IP, MAC address
    │      │
    │      ├─ 3. Spawn share server VM (generic driver)
    │      │       manila calls Nova API:
    │      │       POST /v2.1/servers
    │      │           image: manila-service-image
    │      │           flavor: service_instance_flavor_id
    │      │           networks: [service_net, tenant_net_port]
    │      │       → polls until VM is ACTIVE
    │      │
    │      ├─ 4. Wait for SSH availability
    │      │       manila SSHes into the new VM to verify it is ready
    │      │
    │      ├─ 5. driver.setup_server()
    │      │       installs NFS-Ganesha or Samba on the VM via SSH
    │      │       configures exports directory
    │      │
    │      └─ 6. Write share_server record to DB
    │              status=active
    │              backend_details: {
    │                  'instance_id': '<nova-vm-uuid>',
    │                  'service_ip': '10.0.0.100',
    │                  'tenant_ip': '10.0.1.10',
    │                  'username': 'manila',
    │                  'pk_path': '/tmp/...'
    │              }
    │
    └─ driver.create_share(share, share_server)
           uses backend_details to SSH/call backend API
           creates the actual filesystem/export
           returns export_locations
```

### Share Server Reuse

By default, Manila reuses an existing share server for subsequent shares on the same `(share_network, backend)` pair:

```
share_network "share-net" + backend "generic@host1"
    └── share_server SS-001 (active, tenant_ip=10.0.1.10)
            ├── share-A  (export: 10.0.1.10:/shares/share-A)
            ├── share-B  (export: 10.0.1.10:/shares/share-B)
            └── share-C  (export: 10.0.1.10:/shares/share-C)
```

Configure share server sharing:
```ini
[DEFAULT]
# Reuse share servers across shares on the same network (default: true)
# Set to false to create one share server per share
use_scheduler_creating_share_from_snapshot = true
```

### Share Server Deletion

```
All shares on share server are deleted
    │
manila-share: periodic cleanup task
    │
    ├─ Check: any shares still using this server? No →
    │
    ├─ driver.teardown_server()
    │       SSH to VM: stop Ganesha/Samba, unmount volumes
    │
    ├─ Delete Nova VM (Nova API DELETE /v2.1/servers/<id>)
    │
    ├─ Delete Neutron ports (Neutron API DELETE /v2.0/ports/<id>)
    │
    └─ Delete Cinder volumes if any
           (Cinder API DELETE /v3/<proj>/volumes/<id>)
```

Cleanup interval: `unused_share_server_cleanup_interval` (default: 10 minutes).

## manila-data Service

The `manila-data` service performs the data-plane work for host-assisted share migration. It is a standalone process with its own oslo.messaging consumer.

### Migration Data Copy Flow

```
manila-api: migration_start RPC → manila-data
    │
manila-data.data_manager.migration_start():
    │
    ├─ 1. Create destination share (via manila-share RPC, status=migrating_to)
    │
    ├─ 2. Add temporary access rules to source (rw for data service IP)
    │      Add temporary access rules to destination (rw for data service IP)
    │
    ├─ 3. Mount source share locally
    │       /tmp/manila-migration/<src-share-id>/
    │
    ├─ 4. Mount destination share locally
    │       /tmp/manila-migration/<dst-share-id>/
    │
    ├─ 5. Iterative copy loop (using shutil or rsync-style logic)
    │       task_state = data_copying_in_progress
    │       progress percentage updated in DB
    │
    ├─ 6. On migration_complete RPC from API:
    │       Final rsync pass (catch changes during copy)
    │       Unmount both shares
    │       Remove temporary access rules
    │
    ├─ 7. Driver swap (source export → destination export)
    │       Update share.export_locations in DB
    │       Update share.host, share.share_type, share.share_network
    │       Delete source share from old backend
    │
    └─ 8. task_state = None, status = available
           Share record now points to destination backend
```

The `manila-data` service requires:
- Network access to both source and destination share export paths
- Sufficient local disk for temporary mount points (not data — it streams)
- The NFS/CIFS client utilities installed on the data service host

## Scheduler Filters

The scheduler runs a pipeline of filter classes to eliminate incompatible share backends, then weighers to rank them. The winning backend receives the share creation RPC.

### Filter Pipeline

```
share_create request
    │
    ▼
[all active manila-share nodes with their reported capabilities]
    │
    ├─ AvailabilityZoneFilter
    │    Keep only backends in the requested AZ (or all AZs if not specified)
    │
    ├─ CapacityFilter
    │    Keep backends where:
    │      free_capacity_gb >= requested_size
    │      free_capacity_gb > total_capacity_gb * reserved_percentage / 100
    │
    ├─ CapabilitiesFilter
    │    Match share type extra_specs against backend capability dict
    │    Example: snapshot_support=True must match backend's snapshot_support=True
    │    Supports operators: =, <, >, <=, >=, <is>, <in>, <or>
    │
    ├─ ShareReplicationFilter
    │    When creating a replica: require that backend has same replication_type
    │    as the existing active replica's backend
    │
    └─ [custom filters via scheduler_default_filters config]
```

### CapabilitiesFilter Matching Rules

The filter compares share type extra_specs to backend capability keys:

| Extra Spec | Backend Value | Match? |
|---|---|---|
| `snapshot_support=True` | `snapshot_support: True` | Yes |
| `snapshot_support=True` | `snapshot_support: False` | No |
| `replication_type=readable` | `replication_type: readable` | Yes |
| `replication_type=readable` | `replication_type: dr` | No |
| `provisioning:max_share_size=<= 500` | `provisioning:max_share_size: 1000` | Yes (500 <= 1000) |

### Weighers

After filtering, weighers score remaining backends. The default weigher is `CapacityWeigher`, which preferentially places shares on the backend with the most free capacity (or least, depending on `capacity_weight_multiplier`):

```ini
[DEFAULT]
# Positive = place on most free backend (default)
# Negative = place on least free backend (spread usage evenly)
capacity_weight_multiplier = 1.0
```

### Scheduler Configuration

```ini
[DEFAULT]
# Filter classes to use (comma-separated)
scheduler_default_filters = AvailabilityZoneFilter,CapacityFilter,CapabilitiesFilter,ShareReplicationFilter

# Weigher classes to use
scheduler_default_weighers = CapacityWeigher

# How long (seconds) before a backend's stats are considered stale
scheduler_driver = manila.scheduler.drivers.filter.FilterScheduler
```

## Database Models

Manila uses SQLAlchemy with Alembic migrations. The schema lives in `manila/db/migrations/alembic/versions/`.

### Core Tables

```
shares
├── id              UUID primary key
├── deleted         soft-delete flag
├── user_id         owner user
├── project_id      owner project
├── host            manila-share host (e.g., "manila@cephfs#pool")
├── size            GiB
├── status          creating | available | error | deleting | migrating | ...
├── task_state      migration/replication in-progress state
├── share_proto     NFS | CIFS | CephFS | ...
├── share_type_id   FK → share_types
├── share_network_id FK → share_networks (nullable, DHSS=true only)
├── share_server_id FK → share_servers (nullable, DHSS=true only)
└── is_public       boolean

share_instances
├── id              UUID; a share can have multiple instances (replicas)
├── share_id        FK → shares
├── host
├── status
├── replica_state   active | in_sync | out_of_sync (replication only)
└── availability_zone

export_locations
├── id
├── share_instance_id  FK → share_instances
├── path            mount path string
└── is_admin_only   admin-visible paths (e.g., management IPs)

share_access_map
├── id
├── share_id        FK → shares
├── access_type     ip | user | cert | cephx
├── access_to       CIDR, username, cert CN, CephX id
├── access_level    rw | ro
└── state           active | error | queued_to_apply | queued_to_deny

share_types
├── id
├── name
├── is_public
└── extra_specs[]   FK → share_type_extra_specs

share_type_extra_specs
├── share_type_id   FK → share_types
├── key
└── value

share_networks
├── id
├── project_id
└── name

share_network_subnets
├── id
├── share_network_id  FK → share_networks
├── neutron_net_id
├── neutron_subnet_id
└── availability_zone_id

share_servers
├── id
├── share_network_subnet_id  FK → share_network_subnets
├── host
├── status          creating | active | deleting | error
└── backend_details  JSON blob (driver-specific: IPs, credentials, VM IDs)

share_snapshots
├── id
├── share_id
├── size
├── status
└── provider_location  (driver-specific opaque reference)

share_groups
├── id
├── project_id
├── share_type_ids[]
└── status

share_group_snapshots
├── id
├── share_group_id
└── status
```

### State Machine for Shares

```
                    ┌─────────┐
            ─────▶  │ creating│
                    └────┬────┘
                         │ create_share() success
                         ▼
                    ┌─────────┐
         ┌──────────│available│◀──────────────────┐
         │          └────┬────┘                   │
         │               │                        │
    extending        deleting                  extending/
    shrinking            │                    shrinking done
         │          ┌────▼────┐
         └────▶     │ deleting│
                    └────┬────┘
                         │
                    ┌────▼────┐
                    │ deleted │
                    └─────────┘

Additional states:
  error           — driver raised an exception
  migrating       — migration_start called on source
  migrating_to    — this instance is the migration destination
  manage_starting — manage_existing() in progress
  unmanage_starting — unmanage() in progress
  replication_change — replica promotion in progress
```

### Access Rule State Machine

```
queued_to_apply → applying → active
                           → error

queued_to_deny  → denying  → deleted
                           → error
```

Access rules are applied asynchronously. After `allow_access` is called, the rule enters `queued_to_apply`. The `manila-share` periodic task picks up queued rules and calls the driver's `allow_access()` method. On success the state transitions to `active`.

## Periodic Tasks

`manila-share` runs several periodic tasks via the `oslo.service` periodic task framework:

| Task | Default Interval | Purpose |
|---|---|---|
| `_update_share_stats` | 60 seconds | Report capacity/capabilities to scheduler |
| `_update_access_rules_if_needed` | 60 seconds | Apply/deny queued access rules |
| `delete_free_share_servers` | 600 seconds | GC unused share servers (DHSS=true) |
| `_do_share_replication_update` | 300 seconds | Trigger backend replica sync check |
| `update_share_instances_access` | 300 seconds | Reconcile access on recovered instances |

## RPC Versioning

Manila uses oslo.messaging versioned RPC with `oslo.versionedobjects`. API service calls follow:

```
manila-api
    │  cast/call to manila.scheduler.rpcapi
    │  topic: manila-scheduler
    ▼
manila-scheduler
    │  cast to manila.share.rpcapi
    │  topic: manila-share.<host>
    ▼
manila-share
    │  cast/call to manila.data.rpcapi (migration only)
    │  topic: manila-data
    ▼
manila-data
```

Each rpcapi module specifies `RPC_API_VERSION` and uses `can_send_version()` for rolling upgrade compatibility, allowing mixed-version deployments during upgrades.
