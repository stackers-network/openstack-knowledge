# Cinder Architecture

## Service Components

Cinder is composed of four daemon processes that communicate over an oslo.messaging RPC bus (typically RabbitMQ). Each process can be scaled and placed independently.

### cinder-api

The WSGI API server. Validates and authenticates requests (via keystonemiddleware), enforces policy (policy.yaml / oslo.policy), and forwards work to cinder-scheduler or cinder-volume via RPC. The API is versioned; the current stable version is **v3** (microversions). API v2 was removed in the Xena release.

The API process is stateless and horizontally scalable behind a load balancer. All state lives in the database.

Key responsibilities:
- Accept and validate volume/snapshot/backup/type CRUD requests
- Enforce project quotas before dispatching (quota checks use the DB)
- Route requests to the appropriate scheduler or volume RPC target
- Return job IDs for asynchronous operations (volume creation, backup, etc.)

Configuration file: `/etc/cinder/cinder.conf`
WSGI entry point: `cinder.wsgi.osapi_volume`

### cinder-scheduler

Selects which backend (cinder-volume service instance and pool) should fulfill a `create_volume` or `retype` request. It runs a configurable filter and weigher pipeline, similar to nova-scheduler.

The scheduler maintains an in-memory capability cache populated by periodic `update_service_capabilities` messages sent by each cinder-volume worker. This cache holds free capacity, total capacity, thin-provisioning info, driver capabilities, and backend-specific extra specs support.

Default scheduler driver: `cinder.scheduler.filter_scheduler.FilterScheduler`

The scheduler is stateless between requests and can be scaled horizontally. Multiple scheduler instances process requests concurrently from the same RabbitMQ queue.

### cinder-volume

The data-plane orchestrator. One cinder-volume process manages one or more storage backends. It receives RPC calls from the scheduler (for creates) or from cinder-api (for operations on existing volumes) and calls the appropriate storage driver.

Each cinder-volume process registers itself in the `services` table with a `host` value of the form `<hostname>@<backend_name>`. For multi-backend deployments the host value includes the backend stanza name, e.g., `storage01@lvm` or `storage01@ceph`.

Key responsibilities:
- Call storage driver to create/delete/extend/clone volumes and snapshots
- Export volumes for attachment (iSCSI target creation, RBD map, NFS export)
- Call Nova's os-initialize_connection and os-terminate_connection APIs
- Manage volume migration between backends
- Perform periodic capacity reporting to the scheduler

cinder-volume is the only component that requires direct network connectivity to the storage backend. It does not need to be co-located with compute nodes.

### cinder-backup

Manages volume backup and restore operations. Supports multiple backup backends:

| Backend | Driver | Notes |
|---|---|---|
| Ceph (RBD) | `cinder.backup.drivers.ceph.CephBackupDriver` | Efficient incremental backups via RBD diff |
| Swift | `cinder.backup.drivers.swift.SwiftBackupDriver` | Default; stores backup chunks in Swift containers |
| NFS | `cinder.backup.drivers.nfs.NFSBackupDriver` | Mounts NFS share, stores backup files |
| POSIX | `cinder.backup.drivers.posix.PosixBackupDriver` | Local filesystem; rarely used in production |
| Google Cloud Storage | `cinder.backup.drivers.google.GoogleBackupDriver` | For hybrid clouds |

Backup operations are asynchronous. cinder-backup reads from the volume (via the volume driver or via a snapshot) and writes chunked, optionally compressed and encrypted, data to the backup store.

cinder-backup can be scaled horizontally. Multiple instances share the work queue. Backup metadata is stored in the Cinder database.

---

## Volume Lifecycle

```
User request                   cinder-api
   │                               │
   │  POST /volumes                │  validate + quota check
   │ ────────────────────────────► │
   │                               │  cast to scheduler
   │                               ▼
                           cinder-scheduler
                               │  run filter/weigher pipeline
                               │  select backend host+pool
                               │  cast to cinder-volume@host
                               ▼
                       cinder-volume (on chosen host)
                               │  call driver.create_volume()
                               │  update DB: status=available
                               ▼
                         Storage Backend
                           (LVM / Ceph / etc.)
```

Attach flow (Nova-initiated):

```
nova-compute
   │  os-attach (reserve)          cinder-api
   │ ────────────────────────────► │
   │                               │  RPC → cinder-volume
   │                               ▼
                       cinder-volume
                           │  driver.initialize_connection()
                           │  returns connection_info (iqn, portal, rbd pool, etc.)
                           ▼
                       cinder-api → returns connection_info to nova-compute

nova-compute
   │  attach device (libvirt/os-brick connects iSCSI/RBD)
   │  os-attach (finalize)
   │ ────────────────────────────► cinder-api → volume status: in-use
```

---

## Volume Types

A volume type is a named set of extra specifications that maps user intent to backend capabilities. Every volume create request either specifies a type or uses the configured default (`default_volume_type` in `[DEFAULT]`).

Volume types contain:
- **Extra specs**: key-value pairs matched against driver-reported capabilities and scheduler filters. Common keys:
  - `volume_backend_name` — direct-target a specific backend
  - `replication_enabled` — `<is> True` for replicated backends
  - `IOPS_sec` — passed to QoS specs if linked
  - `multiattach` — `<is> True` to allow multi-attach volumes
  - `encrypted` — set by encryption providers (see below)
- **QoS specs**: linked QoS policy (see below)
- **Encryption spec**: linked encryption provider (see below)

```bash
# Create a volume type
openstack volume type create ssd-replicated \
  --property volume_backend_name=ceph-ssd \
  --property replication_enabled='<is> True'

# Show type and its extra specs
openstack volume type show ssd-replicated
```

### Default Volume Type

Set in `cinder.conf`:

```ini
[DEFAULT]
default_volume_type = standard
```

If no default is configured and a request omits `--type`, Cinder uses any type named `__DEFAULT__` (created automatically during DB migration in recent releases).

---

## QoS Specs

QoS specs define I/O limits that are enforced either front-end (by the hypervisor/libvirt) or back-end (by the storage array). They are created independently and then associated with one or more volume types.

Front-end QoS (enforced by libvirt via cgroup blkio):

| Key | Description |
|---|---|
| `total_bytes_sec` | Total throughput limit (bytes/sec) |
| `read_bytes_sec` | Read throughput limit |
| `write_bytes_sec` | Write throughput limit |
| `total_iops_sec` | Total IOPS limit |
| `read_iops_sec` | Read IOPS limit |
| `write_iops_sec` | Write IOPS limit |
| `total_bytes_sec_max` | Burst total throughput |
| `total_iops_sec_max` | Burst total IOPS |

Back-end QoS keys are driver-specific (e.g., Pure Storage uses `maxIOPS`, `maxBWS`).

```bash
openstack volume qos create high-iops \
  --consumer front-end \
  --property total_iops_sec=5000 \
  --property total_bytes_sec=524288000

openstack volume qos associate high-iops ssd-replicated
```

---

## Volume Encryption

Cinder supports per-type volume encryption using the LUKS provider backed by Barbican for key management.

When a volume of an encrypted type is created:
1. cinder-volume calls Barbican to generate a new symmetric key (AES-256 by default).
2. The key reference (Barbican secret UUID) is stored in the `encryption_key_id` column of the volumes table.
3. nova-compute retrieves the key from Barbican at attach time and sets up dm-crypt/LUKS on the compute node.
4. The encrypted device is presented to the instance.

```bash
# Create an encryption provider for a volume type
openstack volume encryption type create \
  --provider nova.volume.encryptors.luks.LuksEncryptor \
  --cipher aes-xts-plain64 \
  --key-size 256 \
  --control-location front-end \
  encrypted-ssd
```

Requirements:
- Barbican must be deployed and registered in the Keystone catalog
- `[keymgr] backend = barbican` in `cinder.conf`
- Nova compute nodes need `cryptsetup` and `dm-crypt` kernel module

---

## Integration with Nova

Nova interacts with Cinder for:

| Operation | Nova action | Cinder action |
|---|---|---|
| Boot from volume | `os-createBackup` or regular create | Cinder creates volume from image (clones or copies) |
| Attach volume | `os-attach` (reserve → initialize_connection → attach) | Driver exports volume, returns connector info |
| Detach volume | `os-detach` (begin_detach → terminate_connection → detach) | Driver unexports volume |
| Resize instance (root volume) | `os-resize` triggers volume extend | `extend_volume` on the root Cinder volume |
| Evacuate | Nova calls detach + attach on new host | Cinder volume stays in-use until re-attached |
| Snapshot instance | Nova calls `os-createBackup` | Cinder creates a snapshot of each attached volume |

The Nova compute node uses **os-brick** (the OpenStack connector library) to handle the actual host-level connection: logging into iSCSI targets, mapping RBD devices, mounting NFS, etc.

Multiattach volumes (requires `multiattach=True` extra spec and a capable driver):
- Nova must use the libvirt `shareable` disk mode
- Only supported for volumes where the driver reports `multiattach` capability

---

## Integration with Glance

Cinder can use Glance images as volume sources and can export volumes back to Glance:

| Flow | Description |
|---|---|
| Volume from image | `openstack volume create --image <img>` — Cinder downloads image from Glance, writes to new volume |
| Upload volume to image | `openstack image create --volume <vol>` — Cinder reads volume data, uploads to Glance |
| Glance image store (Cinder) | Glance can store images directly in a Cinder volume (via `cinder` store in glance-api.conf) |

For the "volume from image" path, certain drivers (Ceph, NetApp) support native cloning from Glance images stored in the same backend, bypassing full data copy.

---

## Active-Active HA Architecture

### Traditional Active-Passive

The simplest HA model: two cinder-volume nodes share a Pacemaker/Corosync cluster. One node is active; the other takes over on failure. Volume host names are stable (virtual IP or fixed hostname). No DLM required.

Limitation: failover time is 30–60 seconds; backend I/O is interrupted during that window.

### Active-Active HA (2025.x)

Active-active allows multiple cinder-volume workers for the same backend to process requests simultaneously. This requires:

1. **A distributed lock manager (DLM)** — Tooz-backed, typically using etcd3 or ZooKeeper as the coordination backend. Prevents concurrent conflicting operations on the same volume.

2. **Stable per-worker host names** — Each worker registers with a UUID-based `cluster` name rather than hostname, so the scheduler can address the cluster rather than individual nodes.

3. **`[coordination]` section in cinder.conf**:

```ini
[coordination]
backend_url = etcd3+http://etcd01:2379,etcd3+http://etcd02:2379,etcd3+http://etcd03:2379
```

4. **`[DEFAULT] cluster = cinder-cluster-1`** — All workers in the same AA group share this cluster name.

Active-active cinder-backup is also supported using the same Tooz coordination mechanism.

The scheduler routes requests to the cluster name. Individual workers compete for lock ownership before executing a driver call. If a worker dies mid-operation, another worker detects the stale lock (via TTL expiry) and can attempt recovery.

**Requirements for active-active:**
- All workers must use a shared-access backend (Ceph, a SAN with multipath, etc.). LVM is not suitable for active-active unless the volume group is on shared storage.
- etcd 3.x or ZooKeeper 3.6+
- `oslo.concurrency` and `tooz` Python packages

### Database HA

Cinder requires a highly-available database. Recommended: MariaDB Galera Cluster (3+ nodes) fronted by ProxySQL or HAProxy. The API and scheduler processes are stateless and tolerate brief DB unavailability; in-flight driver operations may fail if the DB becomes unavailable mid-operation.

---

## Service Registration and Health

All Cinder services register in the `services` table. Check service health:

```bash
openstack volume service list
```

Expected output for a healthy three-backend deployment:

```
+------------------+----------------------------+------+---------+-------+
| Binary           | Host                       | Zone | Status  | State |
+------------------+----------------------------+------+---------+-------+
| cinder-scheduler | controller01               | nova | enabled | up    |
| cinder-volume    | storage01@lvm              | nova | enabled | up    |
| cinder-volume    | storage01@ceph             | nova | enabled | up    |
| cinder-backup    | storage01                  | nova | enabled | up    |
+------------------+----------------------------+------+---------+-------+
```

A service with `State: down` indicates missed heartbeats. Default heartbeat interval: 10 seconds. Service is considered down after `service_down_time` seconds (default: 60).
