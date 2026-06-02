# Nova Internals

## Scheduler Filters

Filters are applied in sequence to the list of candidate hosts returned by Placement. A host is eliminated the moment any filter returns `False`. Enabled filters are configured in `[filter_scheduler] enabled_filters`.

### Core Filters

**ComputeFilter**
The most fundamental filter. Rejects hosts where `nova-compute` is disabled or where the service is reported as down. Always enable this filter.

**RamFilter**
Rejects hosts where available RAM (after applying `ram_allocation_ratio`) is less than the flavor's requested RAM. Allocation ratio allows overcommit: with `ram_allocation_ratio=1.5`, a host with 32 GB physical RAM appears to have 48 GB available to the scheduler.

**DiskFilter**
Same as RamFilter but for root disk + ephemeral disk. Uses `disk_allocation_ratio`. Checks total available disk on the host against the flavor's `disk + ephemeral` requirement.

**CoreFilter** (deprecated in favour of Placement)
Checks vCPU availability using `cpu_allocation_ratio`. Still available but superseded by Placement VCPU inventory tracking.

**ComputeCapabilitiesFilter**
Evaluates flavor extra specs that start with `capabilities:` against the compute node's reported capabilities. For example, `capabilities:hypervisor_type=QEMU` restricts scheduling to QEMU hypervisors.

**ImagePropertiesFilter**
Evaluates image metadata properties against the compute node's capabilities. An image with `hw_vif_model=virtio` requires the host to support virtio NICs. Properties checked include `architecture`, `hypervisor_type`, `vm_mode`.

**ServerGroupAffinityFilter**
When an instance belongs to an affinity server group, rejects any host that does not already have at least one member of that group. Ensures all group members land on the same host.

**ServerGroupAntiAffinityFilter**
Rejects hosts that already have any member of the same anti-affinity server group. Spreads group members across distinct hosts.

**AggregateInstanceExtraSpecsFilter**
Matches flavor extra specs with prefix `aggregate_instance_extra_specs:` against host aggregate metadata. Only hosts in aggregates whose metadata contains all matching key=value pairs pass. Used to direct workloads to specific hardware types (e.g. `aggregate_instance_extra_specs:ssd=true`).

**AvailabilityZoneFilter**
Restricts scheduling to hosts in the availability zone requested by the user (`--availability-zone` flag). Availability zones are modelled as host aggregates with a special `availability_zone` metadata key.

**NumInstancesFilter**
Rejects hosts whose current instance count exceeds `[filter_scheduler] max_instances_per_host`. Prevents any single host from becoming overloaded while others are empty. Requires `track_instance_changes = True`.

**PciPassthroughFilter**
Rejects hosts that cannot satisfy PCI device passthrough requests specified in the flavor extra specs (`pci_passthrough:alias`).

**NUMATopologyFilter**
Rejects hosts that cannot satisfy the NUMA topology request from the flavor (`hw:numa_nodes`, `hw:numa_mempolicy`). Evaluates NUMA cell availability including CPU, memory, and SR-IOV port placement.

**AggregateMultiTenancyIsolation**
Restricts scheduling to host aggregates that have the tenant's project listed in their `filter_tenant_id` metadata. Used to implement hard project-to-host affinity.

---

## Scheduler Weighers

After filtering, weighers assign a score to each surviving host. Hosts are sorted by total weight (highest wins). Multiple weighers can be active simultaneously; their results are multiplied by their respective `multiplier` config option and summed.

**RAMWeigher**
Scores hosts by available RAM. With a positive multiplier (default `1.0`), favours hosts with more free RAM, spreading instances across hosts. A negative multiplier (`-1.0`) packs instances onto hosts with less free RAM (useful for licence or power management).

Config: `[filter_scheduler] ram_weight_multiplier = 1.0`

**DiskWeigher**
Scores by available disk. Positive multiplier spreads; negative packs.

Config: `[filter_scheduler] disk_weight_multiplier = 1.0`

**CPUWeigher**
Scores by available vCPUs. Not enabled by default since Placement now tracks VCPU inventory.

**MetricsWeigher**
Scores hosts based on arbitrary Ceilometer/Prometheus metrics reported by the compute node. Requires `[metrics]` section configuration and a custom metrics plugin. Rarely used in modern deployments.

**ServerGroupSoftAffinityWeigher**
For `soft-affinity` server groups: boosts the weight of hosts that already have group members. Does not enforce; only influences ordering.

**ServerGroupSoftAntiAffinityWeigher**
For `soft-anti-affinity` server groups: penalises hosts that already have group members, nudging the scheduler toward less-occupied hosts.

**IoOpsWeigher**
Penalises hosts with high current I/O operations (disk-intensive workloads). Useful for mixed workloads.

Config: `[filter_scheduler] io_ops_weight_multiplier = -1.0`

---

## Cells v2 Database Layout

### API Database (`nova_api`)

| Table | Purpose |
|---|---|
| `cell_mappings` | One row per cell: UUID, name, `transport_url`, `database_connection` |
| `host_mappings` | Maps `host` name to a cell UUID |
| `instance_mappings` | Maps instance UUID to `cell_id`; includes project ID for fast tenant lookups |
| `build_requests` | In-flight server build state (before cell assignment; short-lived) |
| `request_specs` | Serialised `RequestSpec` (flavor, image, AZ, hints, group policy) for each instance |
| `quota_usages` | Aggregated usage counters per project/user for each resource class |
| `quotas` | Per-project quota limits |
| `quota_classes` | Default quota class limits |
| `key_pairs` | SSH key pairs (user-scoped) |
| `flavors` | Flavor definitions (shared globally) |
| `flavor_extra_specs` | Extra spec key-value pairs per flavor |
| `aggregate_metadata` | Host aggregate metadata (used by scheduler filters) |
| `aggregate_hosts` | Host aggregate membership |

### Cell Database (`nova_cell1`, etc.)

| Table | Purpose |
|---|---|
| `instances` | One row per instance: power state, task state, vm state, host, node, flavor snapshot |
| `migrations` | Active and historical migration records (resize, live-migrate, evacuate) |
| `block_device_mappings` | Volume and ephemeral disk attachments |
| `instance_actions` | Audit log: who requested what operation and when |
| `instance_actions_events` | Sub-steps of each instance action (create, power_on, etc.) |
| `instance_faults` | Error details when an instance enters ERROR state |
| `instance_extra` | JSON blob of extra instance data: NUMA topology, VCPU pinning, PCI requests |
| `instance_info_caches` | Cached Neutron port information (avoids repeated Neutron API calls) |
| `virtual_interfaces` | Legacy VIF records (retained for backward compatibility) |
| `security_groups` | Legacy security groups (nova-network; deprecated) |
| `console_auth_tokens` | Short-lived tokens for VNC/SPICE/serial console access |

---

## Conductor as Database Proxy

Before cells v2 and the conductor pattern, compute nodes connected directly to the MySQL database. This created several problems: compromised compute nodes could read or tamper with all cloud data, and rolling upgrades were difficult because different service versions had to share the same schema simultaneously.

`nova-conductor` solves both problems:

1. **Security isolation**: Compute nodes have no database credentials. They call `nova-conductor` over encrypted RPC for all DB reads and writes. A compromised compute node cannot query or mutate the database.

2. **Rolling upgrade proxy**: The conductor enforces that RPC messages from older compute nodes (using a lower `rpcapi` version) are translated to the current DB schema. This allows mixed-version deployments during a rolling upgrade window.

3. **Complex workflow orchestration**: Long-running operations (build, resize, live migration) involve many coordinated steps across multiple services. The conductor drives these state machines, handling retries, rollbacks, and error reporting.

The superconductor (API-level conductor) handles cell routing: it looks up the `host_mapping` in the API DB to find which cell owns a given compute host, then forwards the RPC to the correct cell's conductor.

---

## RPC Versioning

Nova uses oslo.messaging for all inter-service communication. RPC method signatures are versioned using a `<major>.<minor>` scheme:

- **Major version bump**: backward-incompatible change. Both sides must be updated before the change can be used.
- **Minor version bump**: backward-compatible addition (e.g. a new optional parameter). The caller can use it; older callees ignore the new field.

Each service (nova-compute, nova-conductor, nova-scheduler) has an `rpcapi.py` that defines the client-side versioned stub. The manager (`manager.py`) defines the server-side methods.

During a rolling upgrade:
1. Update nova-conductor first (it has the highest compatibility burden).
2. Update nova-scheduler.
3. Update nova-compute nodes one at a time.
4. Once all nodes are on the new version, remove the old compatibility code by setting `[upgrade_levels] compute = auto`.

---

## Instance State Machine

### VM States

| State | Meaning |
|---|---|
| `ACTIVE` | Running normally |
| `SHUTOFF` | Powered off (stopped); still consumes quota |
| `PAUSED` | CPU/memory frozen; state held in hypervisor |
| `SUSPENDED` | Saved to disk; no RAM or CPU used |
| `SHELVED` | Instance snapshot taken, offloaded from host |
| `SHELVED_OFFLOADED` | Same as SHELVED; fully removed from compute host |
| `BUILD` | In the process of being created |
| `RESIZE` | Being resized or cold-migrated |
| `VERIFY_RESIZE` | Awaiting operator confirmation of resize |
| `REVERT_RESIZE` | Rolling back a resize |
| `MIGRATING` | Live migration in progress |
| `RESCUE` | Booted from rescue image to repair the OS |
| `ERROR` | A terminal error state; requires operator action |
| `DELETED` | Soft-deleted; row retained in DB for audit |

### Task States

Task state provides finer-grained tracking within a vm_state. For example, an `ACTIVE` instance might have task_state `POWERING_OFF` while a stop is in progress, then `None` when idle again.

Common task states: `scheduling`, `block_device_mapping`, `networking`, `spawning`, `image_snapshot`, `image_snapshot_pending`, `image_uploading`, `resize_prep`, `resize_migrating`, `resize_migrated`, `resize_finish`, `deleting`, `soft_deleting`, `powering_off`, `powering_on`, `rebooting`, `rebooting_hard`, `pausing`, `unpausing`, `suspending`, `resuming`, `shelving`, `shelving_image_pending`, `shelving_image_uploading`, `shelving_offloading`, `unshelving`, `migrating`, `live_migrating`.

### State Transitions (key paths)

```
CREATE REQUEST
  → BUILD / scheduling
  → BUILD / block_device_mapping
  → BUILD / networking
  → BUILD / spawning
  → ACTIVE / None        (success)
  → ERROR / None         (failure at any step)

STOP
  ACTIVE / None → ACTIVE / powering_off → SHUTOFF / None

START
  SHUTOFF / None → SHUTOFF / powering_on → ACTIVE / None

RESIZE (cold migration)
  ACTIVE / None → ACTIVE / resize_prep
  → RESIZE / resize_migrating
  → RESIZE / resize_migrated
  → RESIZE / resize_finish
  → VERIFY_RESIZE / None  (awaiting confirm)
  → ACTIVE / None         (after confirm)
  → ACTIVE / None         (after revert, back to original)

LIVE MIGRATION
  ACTIVE / None → ACTIVE / migrating → ACTIVE / None  (success)
  → ACTIVE / None (rolled back on failure)

SHELVE
  ACTIVE / None → ACTIVE / shelving
  → ACTIVE / shelving_image_pending
  → ACTIVE / shelving_image_uploading
  → SHELVED / None
  → SHELVED_OFFLOADED / None  (after offload)
```

---

## Compute Manager Lifecycle Hooks

`nova/compute/manager.py` defines the `ComputeManager` class, which handles all instance lifecycle operations on a compute node. Key methods:

**`build_and_run_instance`**
Entry point for a new instance build. Calls `_build_and_run_instance` which orchestrates: network setup (Neutron port binding), block device creation (Cinder attach or local disk), image download (Glance), and virt driver `spawn()`.

**`_spawn`** (virt driver method)
The hypervisor-specific method (e.g. `LibvirtDriver.spawn`) that creates the domain XML, writes disk images, and starts the VM.

**`terminate_instance`**
Handles DELETE: triggers cleanup of network ports, Cinder detach, disk deletion, and finally libvirt domain destruction. Marks the instance as DELETED.

**`resize_instance`**
Handles the source side of a cold migration/resize: prepares the disk and moves the instance. The destination side is handled by `finish_resize`.

**`live_migration`**
Source-side live migration driver. Calls `_do_live_migration`, which runs pre-checks, drives the migration loop, and handles post-copy if requested.

**`_sync_power_states`** (periodic)
Runs every `[compute] sync_power_state_interval` seconds. Queries libvirt for actual power state of all instances and reconciles against the DB. Catches cases where libvirt restarted or an instance died unexpectedly.

**`update_available_resource`** (periodic)
Runs every `[compute] resource_tracker_poll_interval` seconds. Queries libvirt for current resource usage (vCPUs, RAM, disk), updates the local resource tracker, and reports the new inventory to Placement.

**`_cleanup_incomplete_migrations`** (periodic)
Finds migrations stuck in `migrating` state where the instance did not complete successfully and cleans up orphaned disk files.
