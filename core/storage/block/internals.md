# Cinder Internals

## Driver Interface

Every Cinder volume driver inherits from `cinder.volume.driver.BaseVD` (and optionally from mixin classes for snapshots, cloning, backup, etc.). The driver is loaded by the cinder-volume service at startup based on the `volume_driver` config option in the backend stanza.

### Required Driver Methods

| Method | Signature | Description |
|---|---|---|
| `create_volume` | `(volume)` | Allocate storage for a new volume. Sets `provider_location` if needed. |
| `delete_volume` | `(volume)` | Release storage and clean up. Must be idempotent. |
| `create_snapshot` | `(snapshot)` | Create a point-in-time snapshot of a volume. |
| `delete_snapshot` | `(snapshot)` | Remove a snapshot. |
| `initialize_connection` | `(volume, connector)` | Export volume to a host. Returns `connection_info` dict consumed by os-brick on the compute node. |
| `terminate_connection` | `(volume, connector, **kwargs)` | Unexport volume from host. |
| `get_volume_stats` | `(refresh=False)` | Return dict of backend stats (free_capacity_gb, total_capacity_gb, driver_version, capabilities). Called periodically by the volume manager to update the scheduler cache. |
| `do_setup` | `(context)` | One-time initialization on service start. Connect to storage backend, validate credentials. |
| `check_for_setup_error` | `()` | Validate configuration and connectivity after do_setup. Raises exception if backend is not usable. |

### Optional but Common Driver Methods

| Method | Description |
|---|---|
| `create_volume_from_snapshot` | Create a new volume from an existing snapshot (faster than full copy) |
| `create_cloned_volume` | Clone an existing volume to a new volume |
| `extend_volume` | Grow a volume to a larger size |
| `copy_image_to_volume` | Write a Glance image to a volume (used during `volume create --image`) |
| `copy_volume_to_image` | Read a volume and upload to Glance |
| `migrate_volume` | Driver-assisted migration to another backend |
| `retype` | Change volume type in-place without data movement (if driver supports it) |
| `update_migrated_volume` | Finalize migration — update provider_id, rename backend objects |
| `manage_existing` | Adopt a pre-existing backend volume |
| `manage_existing_get_size` | Return size of a pre-existing volume |
| `unmanage` | Release Cinder management without deleting data |
| `failover_host` | Switch to replication target (replication v2.1) |
| `enable_replication` | Enable replication for a volume |
| `disable_replication` | Disable replication for a volume |
| `create_consistencygroup` | Create a consistency group (deprecated; use generic groups) |
| `create_group` | Create a generic group |
| `create_group_snapshot` | Create a group snapshot |

### `connection_info` Structure

The dict returned by `initialize_connection` is passed to os-brick on the compute node. Structure varies by protocol:

**iSCSI:**
```python
{
    "driver_volume_type": "iscsi",
    "data": {
        "target_iqn": "iqn.2010-10.org.openstack:volume-abc123",
        "target_portal": "10.0.0.30:3260",
        "target_lun": 1,
        "volume_id": "abc123...",
        "auth_method": "CHAP",
        "auth_username": "cinder-volume-abc123",
        "auth_password": "chappassword",
        "encrypted": False,
        "qos_specs": None,
        "access_mode": "rw"
    }
}
```

**Ceph RBD:**
```python
{
    "driver_volume_type": "rbd",
    "data": {
        "name": "volumes/volume-abc123",
        "hosts": ["ceph-mon01", "ceph-mon02", "ceph-mon03"],
        "ports": ["6789", "6789", "6789"],
        "cluster_name": "ceph",
        "auth_enabled": True,
        "auth_username": "cinder",
        "secret_type": "ceph",
        "secret_uuid": "457eb676-33da-42ec-9a8c-9293d545c337",
        "volume_id": "abc123...",
        "access_mode": "rw",
        "encrypted": False
    }
}
```

**NFS:**
```python
{
    "driver_volume_type": "nfs",
    "data": {
        "export": "10.0.0.40:/exports/cinder",
        "name": "volume-abc123",
        "options": "vers=4,sec=sys",
        "access_mode": "rw"
    }
}
```

---

## Multi-Backend Configuration

Multi-backend allows a single cinder-volume process to manage multiple storage backends simultaneously.

### Enable Multi-Backend

```ini
[DEFAULT]
enabled_backends = lvm-ssd,ceph-hdd,pure-nvme

[lvm-ssd]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_backend_name = lvm-ssd
volume_group = cinder-ssd
target_protocol = iscsi
target_helper = lioadm

[ceph-hdd]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph-hdd
rbd_pool = volumes-hdd
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337

[pure-nvme]
volume_driver = cinder.volume.drivers.pure.PureISCSIDriver
volume_backend_name = pure-nvme
san_ip = 192.168.10.20
pure_api_token = T-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
use_chap_auth = true
```

When `enabled_backends` is set, the `volume_driver` in `[DEFAULT]` is ignored. Each stanza becomes an independent backend. The cinder-volume process registers one service entry per backend, e.g.:
- `storage01@lvm-ssd`
- `storage01@ceph-hdd`
- `storage01@pure-nvme`

The scheduler's `volume_backend_name` filter matches the type's extra spec `volume_backend_name` against the value in each backend stanza.

### Pool-Aware Scheduling

Many drivers report multiple pools within a single backend (e.g., Ceph reports pools, NetApp reports aggregates). Pool information is included in `get_volume_stats`:

```python
{
    "volume_backend_name": "ceph",
    "driver_version": "2.0.0",
    "storage_protocol": "ceph",
    "pools": [
        {
            "pool_name": "volumes-ssd",
            "total_capacity_gb": 10240,
            "free_capacity_gb": 7168,
            "reserved_percentage": 0,
            "thin_provisioning_support": True,
            "thick_provisioning_support": False,
        },
        {
            "pool_name": "volumes-hdd",
            "total_capacity_gb": 51200,
            "free_capacity_gb": 40960,
            "reserved_percentage": 5,
        }
    ]
}
```

The full host string including pool: `storage01@ceph#volumes-ssd`

---

## Scheduler Internals

The scheduler uses a two-phase pipeline: **filters** (eliminate ineligible backends) then **weighers** (rank remaining backends).

### Built-In Scheduler Filters

| Filter | Description |
|---|---|
| `AvailabilityZoneFilter` | Backend must be in the requested AZ (or AZ must be `nova` which matches any) |
| `CapacityFilter` | Backend must have enough free capacity. Respects `reserved_percentage` and `max_over_subscription_ratio` |
| `CapabilitiesFilter` | Backend capabilities must match volume type extra specs. Parses `<is>`, `<in>`, `<all-in>`, `<or>`, `==`, `!=`, `>=`, `<=` operators |
| `DriverFilter` | Evaluates a Python expression from the backend's `filter_function` capability |
| `JsonFilter` | Parses complex JSON-encoded filter expressions |
| `RetryFilter` | Excludes backends that were already tried and failed for this request |
| `InstanceLocalityFilter` | Prefer backends local to the requesting Nova compute host (for performance) |
| `DifferentBackendFilter` | Places volumes of a consistency group on different backends |
| `SameBackendFilter` | Places volumes of a consistency group on the same backend |
| `AffinityFilter` | Soft affinity: schedule near another volume |

Default filter list in `cinder.conf`:

```ini
[DEFAULT]
scheduler_default_filters = AvailabilityZoneFilter,CapacityFilter,CapabilitiesFilter,DriverFilter
```

### Built-In Scheduler Weighers

| Weigher | Description |
|---|---|
| `CapacityWeigher` | Prefer backends with more free capacity (default weight multiplier: 1.0) |
| `AllocatedCapacityWeigher` | Prefer backends with less allocated capacity (inverse of CapacityWeigher) |
| `ChanceWeigher` | Random weight — distributes load randomly |
| `GoodnessWeigher` | Uses the backend's `goodness_function` expression (allows custom weighting logic) |
| `VolumeNumberWeigher` | Prefer backends with fewer volumes |

Configure weighers:

```ini
[DEFAULT]
scheduler_default_weighers = CapacityWeigher,GoodnessWeigher
capacity_weight_multiplier = 1.0    # positive = prefer more free space
goodness_weight_multiplier = 1.0
```

### DriverFilter and GoodnessWeigher Expressions

These allow backends to provide custom scheduling logic using Python expressions evaluated against backend stats:

```ini
[lvm-ssd]
filter_function = "stats.total_capacity_gb < 500"
goodness_function = "100 - stats.allocated_capacity_gb / stats.total_capacity_gb * 100"
```

Available variables: `stats.*` (any key from `get_volume_stats`), `volume.*` (properties of the volume being scheduled), `context.*`.

### Scheduler Capabilities Cache

The scheduler receives `update_service_capabilities` RPC messages from each cinder-volume service at `scheduler_driver_init_wait_time` intervals (default: 60 seconds). These are stored in `HostManager._pool_state_cache`. This cache is in-memory and is lost on scheduler restart — the scheduler waits for at least one capability update from each backend before processing requests.

---

## Volume Replication v2.1

Cinder replication v2.1 (introduced in Mitaka, current in 2025.x) provides disaster-recovery failover between a primary and one or more replica backends. It is a per-backend operation, not per-volume (all volumes on the backend are replicated).

### Concepts

| Term | Description |
|---|---|
| Primary backend | Normal active backend serving I/O |
| Secondary/replica backend | Backup backend; data is replicated here asynchronously or synchronously |
| Replication status | Per-volume: `enabled`, `disabled`, `not-capable`, `error`, `failing-over`, `failed-over` |
| `failover_host` | Admin RPC command that promotes the replica to primary |
| `freeze_host` | Marks a backend as not accepting new operations (pre-failover step) |
| `thaw_host` | Re-enables a frozen backend |

### Driver Requirements for Replication v2.1

Drivers must implement:
- `enable_replication(context, group, volumes)` — enable replication on a group of volumes
- `disable_replication(context, group, volumes)` — disable replication
- `failover_replication(context, group, volumes, secondary_backend_id)` — perform failover
- `list_replication_targets(context, group)` — return available replication targets
- Report `replication_enabled: True` and `replication_targets` in `get_volume_stats`

### Configuring Replication

Example for a driver that supports replication:

```ini
[ceph-primary]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph-primary
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337
replication_device = backend_id:ceph-dr,conf:/etc/ceph/ceph-dr.conf,user:cinder,pool:volumes-dr,secret_uuid:88b4c97c-aaaa-bbbb-cccc-1234567890ab
```

### Failover Procedure

```bash
# 1. Freeze the primary backend (stops new operations)
cinder --os-volume-api-version 3.0 failover-host storage01@ceph-primary --backend-id ceph-dr

# Alternatively using the admin API:
openstack volume backend failover storage01@ceph-primary --backend-id ceph-dr

# 2. After failover, cinder-volume reports host as 'storage01@ceph-primary' but
#    operating on ceph-dr. Update Nova to use new attachment paths.

# 3. To fail back (when primary is restored):
openstack volume backend failover storage01@ceph-primary --backend-id default
```

---

## Database Models

Cinder uses SQLAlchemy ORM with Alembic-managed migrations. Key tables:

### `volumes`

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `display_name` | VARCHAR(255) | User-visible name |
| `display_description` | VARCHAR(255) | User-visible description |
| `user_id` | VARCHAR(255) | Owner user UUID |
| `project_id` | VARCHAR(255) | Owner project UUID |
| `status` | VARCHAR(255) | `creating`, `available`, `in-use`, `deleting`, `error`, `migrating`, `maintenance`, etc. |
| `size` | INTEGER | Size in GiB |
| `volume_type_id` | UUID | FK → `volume_types.id` |
| `availability_zone` | VARCHAR(255) | AZ name |
| `host` | VARCHAR(255) | Backend host: `hostname@backend#pool` |
| `provider_location` | VARCHAR(255) | Driver-specific location (e.g., LV path, RBD image name) |
| `provider_id` | VARCHAR(255) | Driver-specific identifier |
| `encryption_key_id` | UUID | Barbican secret UUID for encrypted volumes |
| `replication_status` | VARCHAR(255) | Replication state |
| `migration_status` | VARCHAR(255) | Migration state |
| `multiattach` | BOOLEAN | Multi-attach enabled |
| `bootable` | BOOLEAN | Marked as bootable |
| `deleted` | BOOLEAN | Soft-delete flag |
| `deleted_at` | DATETIME | Soft-delete timestamp |
| `created_at` | DATETIME | Creation timestamp |
| `updated_at` | DATETIME | Last update timestamp |

### `snapshots`

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `volume_id` | UUID | FK → `volumes.id` |
| `status` | VARCHAR(255) | `creating`, `available`, `deleting`, `error`, etc. |
| `volume_size` | INTEGER | Size of parent volume at snapshot time |
| `provider_location` | VARCHAR(255) | Driver-specific snapshot location |
| `encryption_key_id` | UUID | Inherited from parent volume |
| `cgsnapshot_id` | UUID | FK → `cgsnapshots.id` if part of a CG snapshot |
| `group_snapshot_id` | UUID | FK → `group_snapshots.id` |

### `backups`

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `volume_id` | UUID | UUID of the backed-up volume |
| `status` | VARCHAR(255) | `creating`, `available`, `restoring`, `deleting`, `error` |
| `size` | INTEGER | Volume size at backup time (GiB) |
| `object_count` | INTEGER | Number of backup objects written |
| `container` | VARCHAR(255) | Swift container or S3 bucket name |
| `parent_id` | UUID | Parent backup UUID for incremental backups |
| `is_incremental` | BOOLEAN | True if this is an incremental backup |
| `has_dependent_backups` | BOOLEAN | True if there are incrementals dependent on this backup |
| `driver_info` | TEXT | JSON blob of driver-specific backup metadata |
| `encryption_key_id` | UUID | Key UUID for encrypted volume backups |

### `volume_types`

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `name` | VARCHAR(255) | User-visible type name |
| `description` | VARCHAR(255) | Optional description |
| `qos_specs_id` | UUID | FK → `quality_of_service_specs.id` |
| `is_public` | BOOLEAN | Visible to all projects |
| `deleted` | BOOLEAN | Soft-delete flag |

### `volume_type_extra_specs`

| Column | Type | Description |
|---|---|---|
| `id` | INTEGER | Primary key |
| `volume_type_id` | UUID | FK → `volume_types.id` |
| `key` | VARCHAR(255) | Extra spec key |
| `value` | VARCHAR(255) | Extra spec value |

### `quality_of_service_specs`

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `name` | VARCHAR(255) | QoS spec name |
| `consumer` | VARCHAR(255) | `front-end`, `back-end`, or `both` |
| `specs` | JSON | Key-value pairs of QoS parameters |

### `volume_attachment`

| Column | Type | Description |
|---|---|---|
| `id` | UUID | Primary key |
| `volume_id` | UUID | FK → `volumes.id` |
| `instance_uuid` | UUID | Nova instance UUID |
| `attached_host` | VARCHAR(255) | Compute host where attached |
| `mountpoint` | VARCHAR(255) | Device path on compute node |
| `attach_mode` | VARCHAR(36) | `rw` or `ro` |
| `attach_status` | VARCHAR(255) | `attached`, `detached` |
| `connection_info` | TEXT | JSON blob from `initialize_connection` |

---

## RPC Flow

All inter-service communication uses oslo.messaging. Cinder uses both `call` (synchronous, waits for reply) and `cast` (fire-and-forget).

### Create Volume RPC Flow

```
cinder-api (HTTP POST /volumes)
  │
  ├─ DB: create volumes row (status=creating)
  ├─ DB: reserve quota
  │
  ├─ scheduler_rpcapi.create_volume()   [cast to scheduler queue]
  │
cinder-scheduler (consumes from cinder-scheduler queue)
  │
  ├─ run filters → eliminate backends
  ├─ run weighers → rank backends
  ├─ select winner: storage01@ceph#volumes-ssd
  │
  ├─ volume_rpcapi.create_volume()   [cast to storage01@ceph queue]
  │
cinder-volume (storage01@ceph)
  │
  ├─ driver.create_volume(volume)
  ├─ DB: update volumes row (status=available, host=storage01@ceph#volumes-ssd)
  ├─ update quota usage
```

### Attach Volume RPC Flow (Nova-initiated)

```
nova-compute
  │
  ├─ cinder_client.volumes.reserve(volume_id)          [HTTP → cinder-api]
  │   └─ DB: volume status → attaching
  │
  ├─ cinder_client.volumes.initialize_connection(       [HTTP → cinder-api]
  │       volume_id, connector)
  │   └─ volume_rpcapi.initialize_connection() [call → cinder-volume]
  │       └─ driver.initialize_connection() → returns connection_info
  │
  ├─ os-brick.connect_volume(connection_info)          [local on compute node]
  │   └─ iscsiadm / rbd map / NFS mount
  │
  ├─ libvirt: hot-attach device to instance
  │
  ├─ cinder_client.volumes.attach(volume_id,            [HTTP → cinder-api]
  │       instance_uuid, mountpoint)
  │   └─ DB: volume status → in-use, create volume_attachment row
```

### Backup Create RPC Flow

```
cinder-api (POST /backups)
  │
  ├─ DB: create backups row (status=creating)
  │
  ├─ backup_rpcapi.create_backup()   [cast to cinder-backup queue]
  │
cinder-backup
  │
  ├─ Create a temporary snapshot of the volume
  ├─ Read snapshot via volume driver (attach locally or use driver export)
  ├─ Write chunks to backup store (Swift, Ceph, etc.)
  ├─ Delete temporary snapshot
  ├─ DB: update backups row (status=available, object_count, size)
```

---

## Volume Manager Periodic Tasks

The cinder-volume manager runs several periodic tasks:

| Task | Default Interval | Description |
|---|---|---|
| `_report_driver_status` | 60 seconds | Call `get_volume_stats`, publish capabilities to scheduler |
| `_clean_temporary_snapshots` | 300 seconds | Remove leaked temporary snapshots from failed backups |
| `_update_volume_pg_fields` | 60 seconds | Sync volume group membership metadata |
| `_notify_about_service_capabilities` | 60 seconds | oslo.notifications: publish backend stats to Ceilometer/Gnocchi |
| `_do_cleanup` | At startup | Recover volumes stuck in transitional states after crash |

The startup cleanup (`_do_cleanup`) handles volumes left in `creating`, `deleting`, or `attaching` states after a cinder-volume crash. It resets them to `error` so the operator can investigate.

---

## Thin Provisioning

Thin provisioning allows the sum of allocated volume sizes to exceed actual backend capacity. The scheduler enforces limits via:

```ini
[DEFAULT]
max_over_subscription_ratio = 20.0   # Allow 20x overcommit
reserved_percentage = 10             # Reserve 10% of capacity for internal use
```

A backend reports `thin_provisioning_support = True` and `provisioned_capacity_gb` in its stats. The `CapacityFilter` computes:

```
virtual_free = total_capacity * max_over_subscription_ratio
             - provisioned_capacity
             - total_capacity * reserved_percentage / 100
```

If `virtual_free < requested_size`, the backend is filtered out.

Per-backend override:

```ini
[ceph-ssd]
max_over_subscription_ratio = 10.0
reserved_percentage = 5
```
