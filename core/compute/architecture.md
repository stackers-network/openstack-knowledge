# Nova Architecture

## Nova Services

Nova is composed of several discrete processes that communicate over an oslo.messaging bus (typically RabbitMQ). Each service can be scaled independently and deployed on separate hosts.

### nova-api

The HTTP API server. Accepts REST requests from clients, validates them against Keystone, enforces policy (oslo.policy), and dispatches work to other services via RPC. Exposes:

- The Nova Compute API (v2.1) on port 8774
- The EC2-compatible API (legacy; disabled by default in recent releases)

`nova-api` itself does not touch the hypervisor. It writes initial instance records to the API database and then sends RPC messages to `nova-conductor` or `nova-scheduler`.

### nova-metadata-api

Serves the instance metadata endpoint at `http://169.254.169.254`. Instances call this to retrieve user-data, cloud-init configuration, SSH keys, and network information. In most deployments it runs as a separate WSGI process, though it can share the same process as `nova-api`. Neutron metadata proxy relays requests from instances to this service.

### nova-scheduler

Selects the target compute host for a new instance (or a resize/migration). The scheduler:

1. Queries Placement for a list of allocation candidates (hosts that satisfy the resource request).
2. Runs the remaining hosts through a configurable filter pipeline.
3. Weights surviving hosts and selects the best one.
4. Sends an RPC message to `nova-conductor` with the selected host.

The scheduler does not hold persistent state. It reads from the API database (aggregate metadata, host aggregates) but does not write instance records.

### nova-conductor

Acts as a database proxy for compute nodes. Compute nodes never connect to the database directly — they call `nova-conductor` over RPC, which performs the DB operations on their behalf. This isolates the database from the compute tier and enables rolling upgrades (the conductor enforces backward-compatible RPC handling).

`nova-conductor` also orchestrates complex multi-step operations: build (fan-out to compute), resize, live migration, and evacuation workflows. There is a superconductor process (at the API cell level) and per-cell conductor processes.

### nova-compute

Runs on every hypervisor host. Manages the instance lifecycle on that host by calling the virt driver (libvirt, VMware, Hyper-V, Ironic). Responsibilities:

- Spawning, rebooting, shutting down, and deleting instances
- Attaching and detaching Cinder volumes
- Hot-plugging network interfaces via Neutron
- Reporting resource inventory to Placement
- Executing live migration at the source and destination
- Running periodic tasks (sync power state, resource audit, cleanup)

### nova-novncproxy (and nova-spiceproxy, nova-serialproxy)

Web-based console proxy services. `nova-novncproxy` proxies WebSocket connections from a browser to the VNC console socket on the compute node. Listens on port 6080 by default.

### Summary Table

| Service | Default Port | Scales Per |
|---|---|---|
| nova-api | 8774 | API cell (horizontal) |
| nova-metadata-api | 8775 | API cell (horizontal) |
| nova-scheduler | — (RPC only) | Global (horizontal) |
| nova-conductor | — (RPC only) | Per cell (horizontal) |
| nova-compute | — (RPC only) | Per compute host (1:1) |
| nova-novncproxy | 6080 | API cell (horizontal) |

---

## Cells v2 Architecture

Cells v2 was introduced in Pike and became mandatory in Rocky. It allows a single Nova deployment to span multiple independent "cells", each with its own message queue and database, while presenting a single unified API to operators and users.

### The API Cell

There is always exactly one API cell. It contains:

- **Nova API database** (`nova_api`): stores instance mappings, cell mappings, request specs, build requests, host mappings, key pairs, and quota usage.
- **`nova-api`** and **`nova-scheduler`** processes.
- **Superconductor** (`nova-conductor` at the API level): routes build requests to the correct cell.

### Cell0

`cell0` is a special virtual cell that holds **failed build records** — instances that could not be scheduled at all (e.g. "No valid host was found"). It has its own database (`nova_cell0`) but no message queue and no compute nodes. When scheduling fails, the instance record is written to `cell0` so users can see it (in ERROR state) and receive an error message.

### Cell1 (and Cell2, Cell3, …)

Each real cell (`cell1`, `cell2`, …) has:

- Its own MariaDB/PostgreSQL database (`nova_cell1`, `nova_cell2`, …) holding instance records, migrations, block device mappings, and instance actions.
- Its own RabbitMQ virtual host (or separate broker).
- Its own `nova-conductor` process(es).
- A set of compute nodes registered to that cell.

Cells are registered in the API database using `nova-manage cell_v2 create_cell`. Operators can map compute hosts to cells and balance workloads across cells.

### Database Layout

```
nova_api (API database)
├── instance_mappings       — maps instance UUID → cell UUID
├── cell_mappings           — registered cells (name, transport_url, db_connection)
├── host_mappings           — registered compute hosts → cell
├── build_requests          — in-flight build state before cell assignment
├── request_specs           — scheduler request (flavor, image, AZ, group policy)
├── quota_usages            — project/user resource counters
└── key_pairs               — SSH key pairs (moved to API DB in Rocky)

nova_cell0 (cell0 database)
└── instances               — failed-to-schedule instances only

nova_cell1 (cell1 database)
├── instances               — instance records for this cell
├── migrations              — resize/live-migrate/evacuate records
├── block_device_mappings   — volume and ephemeral disk attachments
├── instance_actions        — audit log of instance operations
├── instance_faults         — fault records for ERROR states
└── virtual_interfaces      — legacy network port records
```

---

## Scheduling Pipeline

A new instance build follows this path:

1. **nova-api** receives `POST /servers`. It validates the request, writes a `build_request` to the API DB, and sends a `select_destinations` RPC call to `nova-scheduler`.

2. **nova-scheduler** calls Placement's `GET /allocation_candidates` endpoint, providing the resource request (VCPU, MEMORY_MB, DISK_GB, traits, aggregates). Placement returns a ranked list of allocation candidate sets (resource providers + allocation plans).

3. **nova-scheduler** runs the Placement-filtered candidates through the **filter pipeline** (ComputeFilter, RamFilter, DiskFilter, etc.) to remove hosts that fail soft constraints that Placement cannot evaluate.

4. **nova-scheduler** runs surviving hosts through the **weigher pipeline** (RAMWeigher, DiskWeigher, etc.) and sorts them.

5. **nova-scheduler** calls Placement to **claim resources** (POST /allocations) for the winning host. If the claim fails (race condition), it retries with the next candidate.

6. **nova-scheduler** returns the selected host to the superconductor.

7. **Superconductor** resolves the cell for that host (via `host_mappings` in the API DB), promotes the `build_request` to a real instance record in the cell DB, and sends a `build_and_run_instance` RPC call to the cell's `nova-conductor`.

8. **Cell nova-conductor** sends the `build_and_run_instance` RPC to the target `nova-compute`.

9. **nova-compute** (the compute manager) executes the build:
   - Creates a Neutron port (or uses the one provided).
   - Downloads the Glance image to a local cache.
   - Creates block devices (Cinder volumes, ephemeral disks).
   - Calls the virt driver (libvirt) to define and boot the domain.
   - Updates the instance record to ACTIVE.

---

## Inter-Service Integration

### Neutron (Networking)

Nova calls Neutron to create or bind network ports during server build, live migration, and resize. The interaction uses Neutron's port-binding API (port binding extended attribute). The compute driver signals the binding host to Neutron when the instance is placed on a specific hypervisor.

Key operations:
- `POST /v2.0/ports` — create a port (or caller provides port ID)
- `PUT /v2.0/ports/{id}` with `binding:host_id` — bind to hypervisor
- Port binding failure is a common cause of ERROR instances.

### Cinder (Block Storage)

Nova calls Cinder to attach volumes during server create (boot-from-volume), `openstack server add volume`, and resize. Cinder manages the volume connection info (iSCSI/NVMe target, Ceph RBD secret) and nova-compute calls the virt driver to connect the block device.

### Glance (Image Service)

Nova calls Glance to download image data when a compute node needs to boot from an image it does not yet have cached locally. The image is written to `instances/_base/` on the compute node (the image cache). Subsequent instances using the same image reuse the cached base image (copy-on-write via qcow2 backing files with libvirt).

### Placement

Nova is Placement's primary consumer. `nova-compute` reports resource inventories to Placement periodically (and on startup). The scheduler queries Placement for allocation candidates on every build. Placement enforces that claimed resources do not exceed inventory limits.

---

## Hypervisor Support

| Hypervisor | Virt Driver | Notes |
|---|---|---|
| KVM via libvirt | `libvirt.LibvirtDriver` | Default; full feature support including live migration, NUMA, SR-IOV, hugepages |
| QEMU (no KVM) | `libvirt.LibvirtDriver` (with `virt_type=qemu`) | Used for testing or nested virt; significant performance penalty |
| VMware vSphere | `vmwareapi.VMwareVCDriver` | Uses vCenter API; separate feature matrix |
| Hyper-V | `hyperv.HyperVDriver` | Windows compute nodes; maintained by the community |
| Ironic (bare metal) | `ironic.IronicDriver` | Nova provisions bare-metal nodes via the Ironic API; Placement manages the `CUSTOM_BAREMETAL_*` resources |
| Fake driver | `fake.FakeDriver` | CI/testing only |

libvirt/KVM is the dominant production hypervisor. All advanced features (CPU pinning, huge pages, SR-IOV, nested resource providers for NUMA) are best supported on KVM.
