# Capacity Planning

This skill covers quota management, resource usage monitoring, Placement API queries, overcommit ratio tuning, growth planning, and flavor design for workload types.

## When to Read This Skill

- Setting or adjusting per-project quotas
- Monitoring resource utilization across the cloud
- Determining when to add compute or storage capacity
- Designing flavors for specific workload types
- Tuning overcommit ratios for CPU and memory
- Integrating with Prometheus and Grafana for trend analysis

---

## Quota Management

Quotas limit the resources any project may consume. Nova manages compute quotas, Neutron manages network quotas, Cinder manages volume quotas.

### View Current Quotas

```bash
# Show compute, network, and volume quotas for a project
openstack quota show --project engineering

# Show default quotas (applied to new projects)
openstack quota show --default

# Show only compute quotas
openstack quota show --project engineering --compute

# Show only network quotas
openstack quota show --project engineering --network

# Show only volume quotas
openstack quota show --project engineering --volume
```

### Set Quotas for a Project

```bash
# Set compute quotas
openstack quota set --project engineering \
  --cores 100 \
  --ram 204800 \
  --instances 50 \
  --key-pairs 10 \
  --server-groups 10 \
  --server-group-members 10

# Set network quotas
openstack quota set --project engineering \
  --networks 10 \
  --subnets 20 \
  --routers 10 \
  --ports 100 \
  --floating-ips 20 \
  --security-groups 20 \
  --security-group-rules 100

# Set volume quotas
openstack quota set --project engineering \
  --volumes 50 \
  --snapshots 50 \
  --gigabytes 5000 \
  --backups 20 \
  --backup-gigabytes 5000
```

### Default Quota Values

| Resource | Default Quota |
|---|---|
| Instances | 10 |
| VCPUs | 20 |
| RAM (MB) | 51200 (50 GB) |
| Floating IPs | 10 |
| Fixed IPs | -1 (unlimited) |
| Security groups | 10 |
| Security group rules | 100 |
| Networks | 10 |
| Subnets | 10 |
| Routers | 10 |
| Ports | 50 |
| Volumes | 10 |
| Snapshots | 10 |
| Volume GB | 1000 |
| Key pairs | 100 |

Modify defaults in `nova.conf` (`[quota]` section) and `neutron.conf` (`[quotas]` section) rather than per-project where global policy changes are intended.

### Reset Quotas to Defaults

```bash
# Delete project-specific quota overrides (returns to defaults)
openstack quota delete --project engineering
```

---

## Resource Usage

### Per-Project Usage

```bash
# Show compute usage for a project (VCPUs, RAM, hours)
openstack usage show --project engineering

# Show usage for a date range
openstack usage show --project engineering \
  --start 2026-01-01 \
  --end 2026-04-01

# Show usage for all projects (admin)
openstack usage list
```

### Hypervisor Aggregate Stats

```bash
# Show aggregated stats across all hypervisors
openstack hypervisor stats show

# Show per-hypervisor utilization
openstack hypervisor list --long

# Show detailed stats for a specific hypervisor
openstack hypervisor show <hypervisor-hostname>
```

The key fields in `hypervisor show`:
- `vcpus` — total physical VCPUs
- `vcpus_used` — allocated VCPUs (may exceed physical due to overcommit)
- `memory_mb` — total RAM
- `memory_mb_used` — allocated RAM
- `free_disk_gb` — unallocated disk capacity
- `running_vms` — number of instances on this hypervisor

### Project Absolute Limits

```bash
# Show limits and current usage for the authenticated project
openstack limits show --absolute

# Show limits for a specific project (admin)
openstack limits show --absolute --project engineering
```

---

## Placement API Queries

Placement provides the authoritative view of resource allocation. Use it to understand the real state of resource consumption, independent of Nova's cached views.

### Resource Provider Inventory

```bash
# List all resource providers (one per compute node, plus shared providers)
openstack resource provider list

# Show the full inventory for a resource provider
openstack resource provider inventory list <rp-uuid>
# Output includes: VCPU, MEMORY_MB, DISK_GB totals and reserved amounts

# Example: check a specific compute node's inventory
RP_UUID=$(openstack resource provider list -f value -c uuid -c name | grep compute-01 | cut -f1 -d' ')
openstack resource provider inventory list $RP_UUID
```

### Resource Provider Usage

```bash
# Show how much of each resource class is currently allocated on a provider
openstack resource provider usage show <rp-uuid>

# Compare inventory vs usage to calculate free capacity
openstack resource provider inventory list <rp-uuid>
openstack resource provider usage show <rp-uuid>
```

### Allocation Candidates

```bash
# Find resource providers that can satisfy a given resource request
openstack allocation candidate list --resource VCPU=4,MEMORY_MB=8192
openstack allocation candidate list --resource VCPU=4,MEMORY_MB=8192,DISK_GB=100

# Request with traits (e.g., SRIOV capable, or a specific aggregate)
openstack allocation candidate list \
  --resource VCPU=4,MEMORY_MB=8192 \
  --required HW_CPU_X86_AVX2

# Show candidates with verbose output including allocation requests
openstack allocation candidate list \
  --resource VCPU=4,MEMORY_MB=8192 \
  --os-placement-api-version 1.36
```

When `allocation candidate list` returns an empty set, no compute host can satisfy the request. This is the root cause of `No valid host was found` scheduler errors.

### Resource Class Usage Across All Providers

```bash
# Summarize VCPU and MEMORY_MB across all resource providers
openstack resource provider list -f value -c uuid | while read uuid; do
  name=$(openstack resource provider show "$uuid" -f value -c name)
  vcpu_total=$(openstack resource provider inventory list "$uuid" -f value | grep VCPU | awk '{print $2}')
  vcpu_used=$(openstack resource provider usage show "$uuid" -f value | grep VCPU | awk '{print $2}')
  echo "$name: VCPU $vcpu_used/$vcpu_total"
done
```

---

## Overcommit Ratios

Overcommit ratios allow Nova to advertise more capacity to Placement than physically exists. This is safe for CPU (which time-shares well) but risky for RAM (which does not).

### Default Ratios

| Resource | Default Ratio | Meaning |
|---|---|---|
| `cpu_allocation_ratio` | 16.0 | Advertise 16x physical VCPUs to Placement |
| `ram_allocation_ratio` | 1.5 | Advertise 1.5x physical RAM to Placement |
| `disk_allocation_ratio` | 1.0 | Advertise exactly the physical disk capacity |

These are configured in `nova.conf` on each compute node (or globally on nova-scheduler):

```ini
# /etc/nova/nova.conf
[DEFAULT]
cpu_allocation_ratio = 16.0
ram_allocation_ratio = 1.5
disk_allocation_ratio = 1.0
```

### Update Ratios at Runtime via Placement

```bash
# Update CPU inventory for a resource provider (changes take effect immediately)
openstack resource provider inventory set <rp-uuid> \
  --resource-class VCPU \
  --allocation-ratio 8.0

openstack resource provider inventory set <rp-uuid> \
  --resource-class MEMORY_MB \
  --allocation-ratio 1.0

openstack resource provider inventory set <rp-uuid> \
  --resource-class DISK_GB \
  --allocation-ratio 1.0
```

### Production Recommendations

| Workload Type | CPU Ratio | RAM Ratio | Notes |
|---|---|---|---|
| General purpose (dev/test) | 4.0–8.0 | 1.0–1.5 | Moderate overcommit is safe for bursty workloads |
| Production web/app servers | 2.0–4.0 | 1.0 | Lower overcommit reduces noisy neighbor risk |
| Memory-intensive workloads | 2.0–4.0 | 1.0 | Never overcommit RAM for workloads that use all allocated memory |
| HPC / CPU-pinned instances | 1.0 | 1.0 | Dedicated CPUs require 1:1 ratio |
| Kubernetes worker nodes | 2.0–4.0 | 1.0–1.2 | k8s requests/limits provide secondary protection |

**CPU overcommit at 16:1** (the default) is aggressive for production. A more conservative starting point is **4:1 to 8:1**. Monitor CPU steal time (`%st` in `top`) on compute nodes — sustained steal above 5% indicates the ratio is too high.

**RAM overcommit at 1.5** means 50% more RAM is promised than exists. If instances actually use their allocated RAM, the hypervisor will swap, causing severe performance degradation. For memory-sensitive workloads, set `ram_allocation_ratio = 1.0`.

### Applying Ratio Changes Cluster-Wide

```bash
# Update nova.conf on all compute nodes
# (Use your configuration management tool — Ansible, Puppet, etc.)
# Then restart nova-compute to push the updated inventory to Placement:
systemctl restart nova-compute

# Verify the new inventory values in Placement
openstack resource provider inventory list <rp-uuid>
```

---

## Growth Planning

### Utilization Thresholds

Trigger capacity planning reviews when sustained (7-day average) utilization exceeds:

| Resource | Warning | Critical |
|---|---|---|
| Effective CPU (adjusted for ratio) | 70% | 85% |
| Physical RAM | 70% | 85% |
| Root disk | 60% | 75% |
| Network bandwidth | 60% | 80% |

"Effective CPU" = `vcpus_used / (vcpus * cpu_allocation_ratio)`. At 16:1 overcommit, effective 85% means 13.6x physical CPUs are allocated.

### Monitoring with Prometheus and Grafana

Nova exposes metrics via the Placement API. Use the OpenStack Exporter or direct Placement API scraping:

```bash
# Install openstack-exporter (golang-based, scrapes OpenStack APIs)
# https://github.com/openstack-exporter/openstack-exporter

# Key metrics to track:
# openstack_nova_vcpus_available — total VCPU capacity (with overcommit)
# openstack_nova_vcpus_used — allocated VCPUs
# openstack_nova_memory_available_bytes — total RAM (with overcommit)
# openstack_nova_memory_used_bytes — allocated RAM
# openstack_nova_running_vms — instance count
# openstack_nova_local_storage_available_bytes — disk capacity
# openstack_nova_local_storage_used_bytes — disk allocated
```

Sample Prometheus scrape config:

```yaml
scrape_configs:
  - job_name: openstack
    static_configs:
      - targets: ['openstack-exporter:9180']
```

Sample Grafana queries for capacity dashboards:

```
# CPU utilization ratio
openstack_nova_vcpus_used / openstack_nova_vcpus_available

# Memory utilization ratio
openstack_nova_memory_used_bytes / openstack_nova_memory_available_bytes

# Per-hypervisor CPU
openstack_nova_vcpus_used{hypervisor_hostname=~".*"} / openstack_nova_vcpus_available{hypervisor_hostname=~".*"}
```

### When to Add Compute Nodes

Add compute capacity when:

1. The 7-day average of any resource (CPU, RAM, disk) exceeds 75% at the cluster level.
2. Scheduler failures (`No valid host was found`) occur more than 1% of the time.
3. Instance launch times exceed SLA thresholds due to scheduling delays.
4. CPU steal time (`%st`) on existing nodes exceeds 5% sustained — indicates overcommit is too aggressive, and you need more physical capacity.

### Adding a Compute Node

```bash
# After provisioning and configuring the new compute node:

# 1. Verify nova-compute started and registered
openstack compute service list | grep <new-node-hostname>

# 2. Verify it appeared as a resource provider in Placement
openstack resource provider list | grep <new-node-hostname>

# 3. Verify inventory is correct
RP_UUID=$(openstack resource provider list -f value -c uuid -c name | grep <new-node-hostname> | cut -f1 -d' ')
openstack resource provider inventory list $RP_UUID

# 4. Add to host aggregates if applicable
openstack aggregate add host <aggregate-name> <new-node-hostname>
```

---

## Flavor Design

Flavors define the resource profile of an instance. Well-designed flavors match workload requirements without wasting capacity.

### Standard Flavor Sizing

```bash
# Create flavors for a typical set of workload tiers
# Naming convention: <ram-gb>gb.<vcpu>cpu or descriptive names

# Micro (development, testing)
openstack flavor create --ram 512 --vcpus 1 --disk 10 m1.micro

# Small
openstack flavor create --ram 2048 --vcpus 1 --disk 20 m1.small

# Medium
openstack flavor create --ram 4096 --vcpus 2 --disk 40 m1.medium

# Large
openstack flavor create --ram 8192 --vcpus 4 --disk 80 m1.large

# XLarge
openstack flavor create --ram 16384 --vcpus 8 --disk 160 m1.xlarge

# Memory-optimized
openstack flavor create --ram 32768 --vcpus 4 --disk 80 m1.memory.large

# Compute-optimized
openstack flavor create --ram 8192 --vcpus 16 --disk 80 m1.compute.large
```

### Ephemeral Disk vs. Boot-from-Volume

```bash
# Flavor with no ephemeral disk (instance must boot from volume)
openstack flavor create --ram 8192 --vcpus 4 --disk 0 m1.no-local-disk

# Flavor with large ephemeral disk (for scratch/temp data)
openstack flavor create --ram 16384 --vcpus 8 --disk 200 --ephemeral 500 m1.data
```

### NUMA Topology Flavors

Use NUMA topology hints for workloads that benefit from NUMA locality (databases, memory-intensive apps).

```bash
# Create a flavor with NUMA topology requirements
openstack flavor create --ram 16384 --vcpus 8 --disk 80 m1.numa

# Set NUMA topology extra specs
openstack flavor set m1.numa \
  --property hw:numa_nodes=1

# Split across 2 NUMA nodes (for large instances that span nodes)
openstack flavor set m1.numa-2node \
  --property hw:numa_nodes=2

# Pin specific vCPUs to specific NUMA nodes
openstack flavor set m1.numa \
  --property hw:numa_cpus.0=0,1,2,3 \
  --property hw:numa_mem.0=16384
```

### CPU Pinning (Dedicated CPUs)

CPU pinning allocates dedicated physical CPU cores to the instance, eliminating time-sharing.

```bash
openstack flavor create --ram 16384 --vcpus 8 --disk 80 m1.pinned

# Enable CPU pinning
openstack flavor set m1.pinned \
  --property hw:cpu_policy=dedicated

# Optional: require simultaneous multi-threading siblings together
openstack flavor set m1.pinned \
  --property hw:cpu_thread_policy=require  # requires SMT siblings
# or
openstack flavor set m1.pinned \
  --property hw:cpu_thread_policy=isolate  # isolate from SMT siblings (lower performance, more isolation)
```

Compute nodes hosting pinned instances must have a `dedicated` CPU pool configured. Set the CPU topology in nova.conf:

```ini
# /etc/nova/nova.conf on compute nodes with pinned instances
[compute]
cpu_dedicated_set = 4-23    # physical CPUs reserved for pinned instances
cpu_shared_set = 0-3        # physical CPUs for shared (unpinned) instances
```

### Huge Pages

Huge pages (2 MB or 1 GB) improve memory performance for memory-intensive workloads by reducing TLB pressure.

```bash
openstack flavor create --ram 32768 --vcpus 8 --disk 80 m1.hugepages

# Request 2 MB huge pages (most common)
openstack flavor set m1.hugepages \
  --property hw:mem_page_size=large

# Request any available page size (huge pages preferred, falls back to 4K)
openstack flavor set m1.hugepages \
  --property hw:mem_page_size=any

# Request a specific size: 2048 KB (2 MB) or 1048576 KB (1 GB)
openstack flavor set m1.hugepages \
  --property hw:mem_page_size=2048
```

Compute nodes must have huge pages pre-allocated. Configure in grub:

```bash
# /etc/default/grub — allocate 500x 2MB huge pages at boot
GRUB_CMDLINE_LINUX="... hugepages=500 hugepagesz=2M"
update-grub && reboot
```

### Flavor Access Control

By default, flavors are public (available to all projects). Restrict access with private flavors:

```bash
# Create a private flavor
openstack flavor create --ram 131072 --vcpus 32 --disk 0 m1.hpc --private

# Grant access to specific projects
openstack flavor set m1.hpc --project hpc-team
openstack flavor set m1.hpc --project research-group

# List projects with access
openstack flavor access list m1.hpc
```

### Flavor Summary Table

| Flavor | vCPUs | RAM | Disk | Use Case |
|---|---|---|---|---|
| m1.micro | 1 | 512 MB | 10 GB | CI/CD runners, test bots |
| m1.small | 1 | 2 GB | 20 GB | Dev environments, small APIs |
| m1.medium | 2 | 4 GB | 40 GB | Web/app servers, small DBs |
| m1.large | 4 | 8 GB | 80 GB | Production app servers |
| m1.xlarge | 8 | 16 GB | 160 GB | Large databases, message brokers |
| m1.memory.large | 4 | 32 GB | 80 GB | In-memory caches (Redis, Memcached) |
| m1.compute.large | 16 | 8 GB | 80 GB | CPU-bound compute, encoding |
| m1.pinned | 8 | 16 GB | 80 GB | Latency-sensitive, dedicated CPU |
| m1.hugepages | 8 | 32 GB | 80 GB | Memory-intensive, large working sets |
| m1.hpc | 32 | 128 GB | 0 GB | HPC, boot from volume |
