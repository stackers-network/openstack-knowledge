# Upgrades — Strategy and Version Guides

This skill covers the OpenStack upgrade strategy, the SLURP release program, rolling upgrade procedures, and database migration commands. Version-specific guides live in sub-files.

## When to Read This Skill

- Planning or executing an upgrade between OpenStack releases
- Understanding which upgrade paths are valid
- Configuring rolling upgrades to minimize downtime
- Running database migrations after upgrading service packages

## Sub-Files

| File | What It Covers |
|---|---|
| [2025.1-to-2025.2.md](2025.1-to-2025.2.md) | Upgrading from Epoxy (2025.1) to the 2025.2 release — sequential path |
| [2025.1-to-2026.1.md](2025.1-to-2026.1.md) | Upgrading from Epoxy (2025.1) to 2026.1 via the SLURP path |
| [2025.2-to-2026.1.md](2025.2-to-2026.1.md) | Upgrading from 2025.2 to 2026.1 — sequential path |

---

## SLURP — Skip Level Upgrade Release Process

**SLURP** (Skip Level Upgrade Release Process) is an OpenStack upgrade program introduced to allow operators to skip intermediate releases when upgrading between major versions.

### How It Works

OpenStack releases every six months (roughly March and September). Before SLURP, operators had to upgrade through every intermediate release — a costly and time-consuming process for operators running production clouds.

Under the SLURP program:
- Alternate releases are designated as **SLURP releases**.
- Operators on a SLURP release may upgrade directly to the **next SLURP release**, skipping the intermediate release entirely.
- Non-SLURP releases must upgrade sequentially to the next release before moving further.

### Current SLURP Releases

| Release | Codename | SLURP? | Year |
|---|---|---|---|
| 2024.1 | Caracal | Yes | 2024 |
| 2024.2 | Dalmatian | No | 2024 |
| **2025.1** | **Epoxy** | **Yes** | **2025** |
| 2025.2 | Flamingo | No | 2025 |
| **2026.1** | **Gazpacho** | **Yes** | **2026** |

### Upgrade Paths for 2025.x and 2026.1

```
2025.1 (Epoxy) ──────────────────────────────► 2026.1   [SLURP path — skips 2025.2]
2025.1 (Epoxy) ──────► 2025.2 ──────────────► 2026.1   [Sequential path — two hops]
                        2025.2 ──────────────► 2026.1   [Sequential path — one hop]
```

### SLURP Requirements

For a SLURP upgrade to succeed, the following conditions must be met:

1. **Source release must be a SLURP release** — 2025.1 is SLURP; 2025.2 is not and cannot skip to 2026.1.
2. **All online data migrations must be complete** — run `<service>-manage db online_data_migrations` on the source release until it reports zero changes before starting the upgrade.
3. **No skipped deprecations** — config options deprecated in the source release must be removed or migrated before the SLURP upgrade, because the target SLURP release removes them.
4. **All services must be at the source SLURP version** — mixed-version intermediate states must be resolved before starting the SLURP upgrade.

### Why SLURP Matters for Operators

- Reduces upgrade frequency from twice yearly to once yearly (if desired).
- Allows planning upgrades around organizational maintenance windows.
- Means operators have more time to test and validate before the next required upgrade.
- Trade-off: a larger version delta means more breaking changes to review at once.

---

## General Upgrade Strategy

### Pre-Upgrade Checklist

Complete every item before touching packages:

- [ ] Read the release notes for the target version — especially the "Upgrade Notes" and "Deprecation Notes" sections
- [ ] Check `reference/known-issues/` for known upgrade bugs affecting your configuration
- [ ] Verify all services are healthy: `openstack compute service list`, `openstack network agent list`, `openstack volume service list`
- [ ] Resolve any services in `down` state before upgrading
- [ ] Verify database cluster health (Galera: `wsrep_local_state_comment = Synced`)
- [ ] Verify RabbitMQ cluster health: `rabbitmqctl cluster_status`, zero alarms
- [ ] Complete all pending online data migrations on the current version
- [ ] Remove or migrate all deprecated configuration options for the current version
- [ ] Take full database backups (see `operations/backup-restore/`)
- [ ] Take configuration backups: `tar czf openstack-config-backup.tar.gz /etc/keystone /etc/nova /etc/neutron /etc/cinder /etc/glance`
- [ ] Plan a maintenance window (rolling upgrades reduce but do not eliminate downtime risk)
- [ ] Test the upgrade procedure in a non-production environment first

### Service Upgrade Order

Upgrade services in this order. The order matters because later services depend on earlier ones.

```
1. Infrastructure
   ├── RabbitMQ
   ├── MariaDB / Galera
   └── Memcached

2. Identity
   └── Keystone

3. Image
   └── Glance

4. Block Storage
   └── Cinder

5. Networking
   └── Neutron

6. Compute
   ├── Nova API / Conductor / Scheduler (control plane)
   └── Nova Compute (data plane — last to upgrade)

7. Remaining Services (in any order after Nova)
   ├── Placement
   ├── Heat
   ├── Octavia
   ├── Designate
   ├── Barbican
   ├── Ironic
   ├── Magnum
   └── Manila
```

### Per-Service Upgrade Steps

For each service, follow this pattern:

```bash
# 1. Stop the service
systemctl stop <service>

# 2. Upgrade the package
apt-get install --only-upgrade python3-<service> openstack-<service>
# or
dnf upgrade python3-<service> openstack-<service>

# 3. Update configuration (merge new config options, remove deprecated ones)
# Compare /etc/<service>/<service>.conf.dpkg-new or .rpmnew with your running config

# 4. Run database migration (if the service has a DB)
<see database migrations section below>

# 5. Start the service
systemctl start <service>

# 6. Verify the service is healthy
systemctl status <service>
openstack <service> service list  # or equivalent
```

---

## Rolling Upgrades

Rolling upgrades allow upgrading the control plane while keeping the data plane running. This minimizes instance disruption but requires careful sequencing.

### Services That Support Rolling Upgrades

| Service | Rolling Upgrade Support |
|---|---|
| Nova | Yes — since Rocky (N-1 compute ↔ N conductor) |
| Cinder | Yes — since Rocky |
| Neutron | Yes — partial (server before agents) |
| Keystone | Yes — stateless, multiple versions can coexist |
| Glance | Partial — workers are stateless; DB migration requires brief downtime |

### RPC Version Pinning

When performing a rolling upgrade of Nova or Cinder, pin RPC versions so that the new control plane communicates with old-version compute/volume nodes using the previous RPC version.

**Nova — pin before upgrading conductor/scheduler:**

```ini
# /etc/nova/nova.conf on conductor and scheduler nodes
[upgrade_levels]
compute = auto
```

Setting `compute = auto` tells nova-conductor and nova-scheduler to detect the minimum compute version in the cluster and pin accordingly. This is the recommended approach.

For explicit pinning to a specific version:

```ini
[upgrade_levels]
compute = 6.1   # Replace with the RPC version of the old compute nodes
```

**Cinder — pin before upgrading cinder-api and cinder-scheduler:**

```ini
# /etc/cinder/cinder.conf
[upgrade_levels]
volume = auto
```

**Remove the pin after all nodes are upgraded:**

```ini
# Remove or comment out the [upgrade_levels] section once all nodes
# are running the new version
```

### Rolling Upgrade Sequence for Nova

```bash
# Phase 1: Pin RPC versions
# Add [upgrade_levels] compute = auto to nova.conf on controller
# Restart nova-conductor and nova-scheduler

# Phase 2: Upgrade control plane
apt-get install --only-upgrade nova-api nova-conductor nova-scheduler nova-novncproxy

# Phase 3: Run DB migrations (expand phase)
nova-manage api_db sync
nova-manage db sync

# Phase 4: Upgrade compute nodes one at a time
# For each compute node:
apt-get install --only-upgrade nova-compute
systemctl restart nova-compute
# Verify: openstack compute service list | grep <node>

# Phase 5: Run online data migrations
nova-manage db online_data_migrations
# Repeat until it reports 0 rows migrated

# Phase 6: Remove RPC pin
# Remove [upgrade_levels] section from nova.conf
# Restart nova-conductor and nova-scheduler
```

---

## Database Migrations

### Keystone

```bash
keystone-manage db_sync
```

### Nova

Nova has three databases: `nova_api`, `nova` (cell DB), and `nova_cell0`. All must be migrated.

```bash
# Migrate the API database
nova-manage api_db sync

# Migrate the main cell database (and nova_cell0)
nova-manage db sync

# Run online data migrations (repeat until 0 found)
nova-manage db online_data_migrations
```

### Neutron

```bash
# Migrate all Neutron DB heads (supports multiple migration branches)
neutron-db-manage upgrade heads
```

### Cinder

```bash
cinder-manage db sync

# Run online data migrations (repeat until 0 found)
cinder-manage db online_data_migrations
```

### Glance

```bash
glance-manage db_sync
```

### Heat

```bash
heat-manage db_sync
```

### Octavia

```bash
octavia-db-manage upgrade head
```

### Designate

```bash
designate-manage database sync
```

### Barbican

```bash
barbican-manage db upgrade
```

### Ironic

```bash
ironic-dbsync upgrade
# Run online data migrations
ironic-dbsync online_data_migrations
```

### Magnum

```bash
magnum-db-manage upgrade
```

### Manila

```bash
manila-manage db sync
```

---

## Post-Upgrade Verification

After upgrading all services, verify the cloud is operational:

```bash
# Verify all compute services are up
openstack compute service list

# Verify all network agents are up
openstack network agent list

# Verify all volume services are up
openstack volume service list

# Issue a token to confirm Keystone is healthy
openstack token issue

# List images to confirm Glance is healthy
openstack image list

# List hypervisors to confirm Placement is healthy
openstack hypervisor list

# Run the full health check
# See operations/health-check/
```

Check service logs for deprecation warnings that must be addressed before the next upgrade:

```bash
# Scan logs for deprecation warnings
journalctl -u nova-api | grep -i deprecat | tail -20
journalctl -u neutron-server | grep -i deprecat | tail -20
journalctl -u cinder-api | grep -i deprecat | tail -20
journalctl -u keystone | grep -i deprecat | tail -20
```
