# Live Migration

Live migration moves a running instance from one compute host to another with minimal (ideally zero) downtime for the guest workload. The instance continues to run throughout the migration; memory pages are transferred in the background while the guest is still executing.

## Migration Types

### Shared-Storage Live Migration

The source and destination compute hosts both mount the same NFS share or Ceph RBD pool as the instance root disk. Only the instance's **memory** needs to be transferred. Disk content stays in place.

- Fastest migration type.
- Zero disk I/O on the source; zero disk writes on the destination.
- Requires a shared storage backend (NFS, Ceph RBD, iSCSI multi-attach).
- Used automatically when nova-compute detects a shared backing filesystem.

```bash
openstack server live-migrate my-instance
```

### Block Live Migration

No shared storage. Nova copies the instance's root disk (and any ephemeral disks) from source to destination over the network while the instance runs. After the disk copy completes, memory is also migrated.

- Slower than shared-storage migration (disk copy adds significant time and network I/O).
- Works with local disk (QCOW2 or raw on LVM).
- Requires `block_migration = True` (`--block-migrate` flag).

```bash
openstack server live-migrate --block-migrate my-instance
```

### Volume-Backed Live Migration

The instance's root disk is a Cinder volume (boot-from-volume). The volume is already accessible from any compute node (via iSCSI, NVMe-oF, or Ceph RBD). Only **memory** is transferred during migration.

- Functionally equivalent to shared-storage migration from a performance perspective.
- No `--block-migrate` flag needed; nova-compute detects it automatically.
- The most operationally clean setup: volumes are persistent and host-independent.

```bash
openstack server live-migrate my-instance
```

---

## Prerequisites

All live migration types require:

1. **Same CPU architecture** on source and destination. x86_64 to x86_64. ARM64 to ARM64. Cross-architecture migration is not supported.

2. **Compatible libvirt and QEMU versions**. The QEMU machine type of the instance must be supported on the destination. QEMU downgrades are not permitted. A libvirt version mismatch (e.g. source has libvirt 9.x, destination has 7.x) will fail pre-migration checks.

3. **Passwordless SSH between nova-compute nodes** (for `qemu+ssh://` migration URI) OR TLS-secured live migration (`qemu+tls://`). The `nova` user on the source must be able to SSH to the `nova` user on the destination without a password.

4. **Networking access between hypervisors** on the migration port range. libvirt uses TCP ports 49152–49261 for live migration data transfer by default.

5. **Neutron port binding on the destination**. Before the migration completes, Nova calls Neutron to bind the instance's port to the destination host. If the destination ML2 agent (OVS or OVN) fails to bind the port, the migration is aborted.

6. **Enough resources on the destination**. Placement must have a valid allocation candidate on the destination host before the migration proceeds.

For block migration specifically:

- Sufficient free disk on the destination (`DISK_GB` inventory).
- Destination must have `[libvirt] images_type` compatible with the source (both qcow2, or both raw/LVM).

---

## Tunnelled vs Direct Migration

### Direct Migration (default)

libvirt opens a direct TCP connection from source to destination hypervisor for memory transfer. The data path bypasses the nova-compute process.

```ini
[libvirt]
live_migration_tunnelled = False
live_migration_uri = qemu+ssh://nova@%s/system
```

The `%s` is replaced by the destination hostname at runtime. This URI is used for the libvirt-to-libvirt management channel. The actual migration data uses a separate TCP connection on the QEMU migration port.

### Tunnelled Migration (`VIR_MIGRATE_TUNNELLED`)

Memory transfer is tunnelled through the libvirt daemon using the libvirt wire protocol instead of a raw QEMU TCP socket. Slower but useful when:

- Firewall rules prevent direct QEMU port access between hypervisors.
- You want encryption on the migration data channel (use with TLS).

```ini
[libvirt]
live_migration_tunnelled = True
```

Tunnelled migration has lower throughput than direct migration for large instances (the libvirt daemon becomes the bottleneck). Use direct migration with TLS (`qemu+tls://`) for encrypted high-throughput migrations.

---

## Pre-Copy Migration

Pre-copy is the default migration algorithm:

1. **Dirty bitmap phase**: QEMU begins copying all memory pages to the destination while the guest continues to run. Modified pages are tracked in a "dirty bitmap".

2. **Iterative rounds**: Each round copies the dirty pages from the previous round. The dirty set typically shrinks as the guest stabilises.

3. **Stop-and-copy**: When the remaining dirty set is small enough (or the iteration limit is reached), the guest is briefly paused, the final dirty pages are copied, and control is switched to the destination. The guest resumes on the destination.

The total downtime (pause at the end) is typically 100–500 ms for a well-behaved guest. A guest that writes memory very rapidly ("dirty page storm") can cause the iteration count to climb indefinitely — this is handled by auto-converge or post-copy.

---

## Post-Copy Migration

Post-copy inverts the algorithm: the guest is **moved to the destination first**, then the remaining memory pages are fetched on-demand from the source as the guest faults on them.

Advantages:
- Guaranteed completion time — the migration will always finish.
- Suitable for workloads with high memory write rates that prevent pre-copy convergence.

Disadvantages:
- If **either** the source or the destination crashes during post-copy, the instance is lost (memory is split across two hosts).
- Network interruption during post-copy = dead instance.
- Increased latency during post-copy phase as page faults are served remotely.

Post-copy is triggered in two ways:

1. **Automatic**: When pre-copy has iterated `max_iteration` times without convergence, Nova can automatically switch to post-copy.
2. **Manual**: An operator force-completes the migration (`nova live-migration-force-complete`).

```ini
[libvirt]
# If post_copy is enabled and pre-copy fails to converge, switch automatically
# (requires libvirt >= 1.3.3, QEMU >= 2.5)
# Nova will enable this if live_migration_permit_post_copy = True
```

In `/etc/nova/nova.conf` on the compute nodes:

```ini
[libvirt]
live_migration_permit_post_copy = True
live_migration_permit_auto_converge = True
```

**Auto-converge**: QEMU throttles the vCPU speed on the source to reduce the memory write rate, allowing pre-copy to converge. This impacts guest performance while migration is in progress. Less risky than post-copy. Enable via `live_migration_permit_auto_converge = True`.

---

## Monitor Migration Progress

```bash
# List all active and recent migrations for a server
openstack server migration list my-instance

# Show detailed progress of a specific migration
openstack server migration show my-instance <migration-id>

# Example output fields:
# status:               running | completed | failed | cancelled
# memory_total_bytes:   total guest memory to transfer
# memory_remaining_bytes: still to transfer
# disk_total_bytes:     total disk to copy (block migration only)
# disk_remaining_bytes: still to copy
```

The Nova CLI also exposes the legacy `nova` commands:

```bash
# Legacy: list all migrations across the cell (admin)
nova migration-list

# Show migrations for a specific host
nova migration-list --host compute02.example.com

# Show migrations by status
nova migration-list --status running
```

For deeper inspection, query the libvirt job info directly on the source compute node:

```bash
# On the source compute node
virsh domjobinfo <instance-name>
# e.g. virsh domjobinfo instance-00000042
```

`domjobinfo` reports `Memory remaining`, `Memory total`, `Dirty rate`, and `Iteration` count in real time.

---

## Auto-Converge and Post-Copy Switchover

If a migration is not converging (dirty rate exceeds transfer rate), you have three options:

### 1. Force-Complete (triggers post-copy)

```bash
# Switch the live migration to post-copy (completes immediately)
openstack server migration force-complete my-instance <migration-id>

# Legacy nova command
nova live-migration-force-complete my-instance <migration-id>
```

This triggers a post-copy switchover: the guest moves to the destination immediately and remaining pages are fetched on demand.

### 2. Abort the Migration

```bash
openstack server migration abort my-instance <migration-id>

# Legacy
nova live-migration-abort my-instance <migration-id>
```

The migration is rolled back; the instance stays on the source host. Not available once post-copy has started (the instance is already on the destination at that point).

### 3. Wait for Completion Timeout

Nova automatically aborts migrations that exceed `[libvirt] live_migration_completion_timeout` seconds. Default: 800 seconds.

```ini
[libvirt]
# Abort after 30 minutes if not complete
live_migration_completion_timeout = 1800

# Action when timeout is reached: abort or force_complete
live_migration_timeout_action = abort
```

Set `live_migration_timeout_action = force_complete` to automatically switch to post-copy when the timeout is reached instead of aborting.

---

## Configuration Reference

All settings on the compute node in `/etc/nova/nova.conf`:

```ini
[libvirt]
# Migration protocol URI; %s is replaced by destination hostname
live_migration_uri = qemu+ssh://nova@%s/system

# Use TLS instead of SSH (requires libvirt TLS certificates)
# live_migration_uri = qemu+tls://%s/system

# Tunnel migration data through libvirt (slower but avoids direct QEMU ports)
live_migration_tunnelled = False

# Allow auto-converge (throttle vCPU to reduce dirty page rate)
live_migration_permit_auto_converge = True

# Allow post-copy (switch guest to destination mid-migration)
live_migration_permit_post_copy = True

# Total time (seconds) before migration is aborted or force-completed
live_migration_completion_timeout = 800

# Action when completion_timeout is reached: abort or force_complete
live_migration_timeout_action = abort

# Maximum number of iterative pre-copy rounds before switching to post-copy
# (only meaningful with permit_post_copy = True)
live_migration_max_move_iterations = 150

# Time (seconds) the guest can be paused for final copy
# (currently only enforced on some libvirt versions)
live_migration_downtime = 500
live_migration_downtime_steps = 10
live_migration_downtime_delay = 75

# Bandwidth limit for live migration in MB/s (0 = unlimited)
live_migration_bandwidth = 0

# Progress timeout: abort if no progress in this many seconds
live_migration_progress_timeout = 150
```

---

## Troubleshooting Failures

### "No valid host was found"

The scheduler could not find a suitable destination. Check:

```bash
# Query allocation candidates to confirm resources exist
openstack allocation candidate list \
  --resource VCPU=<vcpus> \
  --resource MEMORY_MB=<ram> \
  --resource DISK_GB=<disk>

# Check if target host has space
openstack resource provider inventory list <provider-uuid>

# Check if target host's compute service is enabled
openstack compute service list --host compute04.example.com
```

### "Unable to migrate to the same host"

Nova refuses to migrate to the source host. Specify a different destination host explicitly:

```bash
openstack server live-migrate --host compute04.example.com my-instance
```

### "Incompatible CPU features" / "CPU model check failed"

libvirt's pre-migration checks found that the destination CPU cannot support the instance's CPU model. Fix options:

1. Set `cpu_mode = host-model` on all compute nodes (recommended). Nova will translate the CPU model to the closest match on each host.
2. Pin all hosts to the same CPU model via aggregate filtering.
3. Use `cpu_mode = custom` with a common baseline model (e.g. `Nehalem`) that all hosts support.

```ini
[libvirt]
cpu_mode = host-model
# or use a common baseline:
# cpu_mode = custom
# cpu_model = Cascadelake-Server-noTSX
```

### "Permission denied" on NFS Migration Socket

When using NFS shared storage, the libvirt UNIX socket permissions may prevent the `nova` user from running migration. Ensure:

```bash
# nova-compute must run as (or have access to) the libvirt group
usermod -aG libvirt nova
```

Check `/var/log/nova/nova-compute.log` and `/var/log/libvirt/libvirtd.log` for exact permission errors.

### "Port binding failed on destination"

Neutron failed to bind the instance's network port to the destination host. Check:

```bash
# Show port binding state (admin)
openstack port show <port-uuid>
# Look at binding:status and binding:vif_details

# Check Neutron agent on destination
openstack network agent list --host compute04.example.com

# Check OVS/OVN logs on destination
journalctl -u neutron-openvswitch-agent -n 100
# or for OVN:
journalctl -u ovn-controller -n 100
```

The most common causes: OVS agent not running on destination; OVN chassis not registered; VLAN/VXLAN segment not present on destination.

### Migration Stuck in "migrating" State

A migration can get stuck if the nova-compute process on either host died mid-migration.

```bash
# Check for stuck migrations (admin)
nova migration-list --status running

# Forcibly abort the migration via the API
openstack server migration abort my-instance <migration-id>

# If that fails (post-copy started), the instance may be in ERROR state.
# Hard reset it on the destination host:
openstack server reboot --hard my-instance

# If the instance is truly lost, evacuate from the dead host
openstack server evacuate --host compute04.example.com my-instance
```

### Checking Logs

Always check both source and destination compute logs for a migration failure:

```bash
# Source compute node
grep -i "live_migration\|migrate" /var/log/nova/nova-compute.log | tail -100

# Destination compute node
grep -i "live_migration\|migrate\|pre_live_migration" /var/log/nova/nova-compute.log | tail -100

# libvirt log on source
tail -100 /var/log/libvirt/libvirtd.log

# Check the instance's action history in the API
openstack server event list my-instance
openstack server event show my-instance <request-id>
```
