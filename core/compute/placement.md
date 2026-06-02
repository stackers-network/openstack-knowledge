# Placement Service

## What Placement Does

Placement is a REST API service that tracks **resource inventories** and **allocations** for a cloud. It answers the question: "Which providers have the resources to satisfy this request, and which have already been claimed?"

Placement was originally developed as part of Nova and shipped as a separate service from the Rocky release onward. Since Stein, it is a fully independent project deployed under the `placement` package, separate from `nova`.

Other OpenStack services (Neutron for bandwidth, Cyborg for accelerators) can also report to and consume Placement. Nova is still the primary user.

The key benefits of Placement over Nova's older in-scheduler resource counting:

- **Atomic claims**: Placement uses database-level compare-and-swap operations to prevent over-claiming during concurrent scheduling.
- **Nested providers**: Supports hierarchical resource providers (NUMA nodes, SR-IOV PFs/VFs) within a single physical host.
- **Traits**: Qualitative capabilities (CPU instruction sets, disk type, NUMA topology) can be matched alongside quantitative resources.
- **Shared providers**: A shared storage pool can be modelled as a single provider shared across many compute nodes.

---

## Resource Providers

A **resource provider** (RP) is any entity that can provide resources. In a standard Nova deployment:

- Each compute node is a resource provider (UUID matches the hypervisor's `compute_nodes.uuid`).
- Shared storage providers (Ceph RBD pools, NFS filers) can be separate providers associated with compute nodes via aggregates.
- IRB/SR-IOV physical functions (PFs) are nested resource providers under the compute node.

```bash
# List all resource providers
openstack resource provider list

# Filter by name (compute node hostname)
openstack resource provider list --name compute01.example.com

# Show a specific provider
openstack resource provider show <provider-uuid>

# Create a custom resource provider (e.g. external GPU cluster)
openstack resource provider create \
  --uuid 11111111-2222-3333-4444-555555555555 \
  gpu-cluster-01
```

---

## Resource Classes

A **resource class** is a named type of quantifiable resource. Standard classes are prefixed `VCPU`, `MEMORY_MB`, `DISK_GB`. Custom classes must start with `CUSTOM_`.

### Standard Resource Classes

| Class | Unit | Description |
|---|---|---|
| `VCPU` | vCPUs | Virtual CPU count |
| `MEMORY_MB` | MB | RAM |
| `DISK_GB` | GB | Root + ephemeral disk |
| `SRIOV_NET_VF` | VF count | SR-IOV virtual functions |
| `PCI_DEVICE` | devices | PCI passthrough devices |
| `PGPU` | GPUs | Physical GPU passthrough |
| `VGPU` | vGPUs | Virtual GPU slices |
| `NET_BW_EGR_KILOBIT_PER_SEC` | Kbps | Egress bandwidth |
| `NET_BW_IGR_KILOBIT_PER_SEC` | Kbps | Ingress bandwidth |

### Custom Resource Classes

Custom classes allow operators to model arbitrary resources:

```bash
# Create a custom resource class
openstack resource class create CUSTOM_FPGA_ACCELERATOR

# List all resource classes (standard + custom)
openstack resource class list

# Delete a custom class (only if unused)
openstack resource class delete CUSTOM_FPGA_ACCELERATOR
```

Custom classes are referenced in flavor extra specs:

```bash
openstack flavor set hpc.fpga \
  --property resources:CUSTOM_FPGA_ACCELERATOR=1 \
  --property trait:CUSTOM_FPGA_XILINX_U250=required
```

---

## Inventories

An **inventory** records how much of a resource class a provider owns and how it can be allocated.

### Inventory Fields

| Field | Description | Typical Value |
|---|---|---|
| `total` | Physical quantity of the resource | 128 (GB RAM) |
| `reserved` | Amount held back for the host OS / hypervisor | 4096 (MB) |
| `min_unit` | Minimum allocation quantum | 1 |
| `max_unit` | Maximum allocation per consumer | 16 (vCPUs) |
| `step_size` | Allocation must be a multiple of this | 1 |
| `allocation_ratio` | Overcommit multiplier | 1.5 (RAM), 4.0 (CPU) |

**Usable capacity** = `(total - reserved) * allocation_ratio`

Nova's resource tracker automatically reports and updates inventories for `VCPU`, `MEMORY_MB`, and `DISK_GB` on every `update_available_resource` cycle.

### Inspect and Modify Inventories

```bash
# List all inventories for a provider
openstack resource provider inventory list <provider-uuid>

# Show inventory for a specific class
openstack resource provider inventory show <provider-uuid> VCPU

# Manually set inventory for a provider (use with caution — nova-compute overwrites on next cycle)
openstack resource provider inventory set <provider-uuid> \
  --resource-class VCPU \
  --total 64 \
  --reserved 4 \
  --min-unit 1 \
  --max-unit 16 \
  --step-size 1 \
  --allocation-ratio 4.0

# Set custom resource inventory that nova-compute won't overwrite
openstack resource provider inventory set <provider-uuid> \
  --resource-class CUSTOM_FPGA_ACCELERATOR \
  --total 4 \
  --reserved 0 \
  --min-unit 1 \
  --max-unit 4 \
  --step-size 1 \
  --allocation-ratio 1.0
```

---

## Allocations

An **allocation** records that a specific consumer (instance UUID) has claimed a quantity of a resource class from a provider.

Allocations are created atomically by the scheduler when a placement claim succeeds. They are deleted when an instance is terminated.

```bash
# Show allocations for a specific instance (consumer UUID = instance UUID)
openstack resource provider allocation show <instance-uuid>

# Example output:
# +--------------------------------------+-------------------------------+----------+
# | resource_provider                    | resource_class                | used     |
# +--------------------------------------+-------------------------------+----------+
# | 9a0a3a9d-1234-5678-abcd-ef0123456789 | VCPU                          | 2        |
# | 9a0a3a9d-1234-5678-abcd-ef0123456789 | MEMORY_MB                     | 2048     |
# | 9a0a3a9d-1234-5678-abcd-ef0123456789 | DISK_GB                       | 20       |
# +--------------------------------------+-------------------------------+----------+

# Show what a provider has allocated to all consumers
openstack resource provider usage show <provider-uuid>

# Delete an allocation (orphan cleanup — use only if instance is confirmed gone)
openstack resource provider allocation delete <instance-uuid>
```

### Healing Orphaned Allocations

If a compute node crashes and instances are manually removed from the DB, allocation records may remain. Use the Nova healer:

```bash
# Audit and heal allocation mismatches (dry-run)
nova-manage placement heal_allocations --verbose --dry-run

# Apply fixes
nova-manage placement heal_allocations --verbose
```

---

## Traits

**Traits** are qualitative capabilities attached to a resource provider. Unlike resource classes (which count units), traits are boolean: a provider either has the trait or it does not.

Traits are used for:
- CPU instruction set extensions (e.g. `HW_CPU_X86_AVX2`, `HW_CPU_X86_AVX512F`)
- Storage type (`STORAGE_DISK_SSD`, `STORAGE_DISK_HDD`)
- Network capabilities (`HW_NIC_OFFLOAD_TSO`, `HW_NIC_SRIOV`)
- Custom capabilities (`CUSTOM_NUMA_TOPOLOGY_2_NODES`, `CUSTOM_GPU_NVIDIA_A100`)

### Standard Trait Prefixes

| Prefix | Domain |
|---|---|
| `HW_CPU_X86_*` | x86 CPU features (AVX, SSE4, AES-NI, etc.) |
| `HW_CPU_ARM_*` | ARM CPU features |
| `HW_NIC_*` | NIC offload and SR-IOV capabilities |
| `HW_GPU_*` | GPU types |
| `STORAGE_DISK_*` | Storage medium type |
| `COMPUTE_*` | Compute node capabilities (live migration, TPM, vGPU) |
| `CUSTOM_*` | Operator-defined traits |

### Manage Traits

```bash
# List all traits
openstack trait list

# Create a custom trait
openstack trait create CUSTOM_GPU_NVIDIA_A100

# Assign traits to a resource provider
openstack resource provider trait set <provider-uuid> \
  HW_CPU_X86_AVX2 \
  HW_CPU_X86_AVX512F \
  CUSTOM_GPU_NVIDIA_A100

# Show traits on a provider
openstack resource provider trait list <provider-uuid>

# Remove a trait from a provider
openstack resource provider trait delete <provider-uuid> CUSTOM_GPU_NVIDIA_A100
```

### Require Traits in Flavors

```bash
# Require a trait (instance will only schedule to providers with this trait)
openstack flavor set gpu.large \
  --property trait:CUSTOM_GPU_NVIDIA_A100=required

# Forbid a trait (instance will NOT schedule to providers with this trait)
openstack flavor set no-gpu.medium \
  --property trait:CUSTOM_GPU_NVIDIA_A100=forbidden
```

---

## Allocation Candidates

The nova-scheduler calls `GET /placement/allocation_candidates` to get a ranked list of (resource-provider set, allocation plan) pairs that satisfy a request. This is the primary interface between Nova scheduling and Placement.

The request specifies:
- Resource requirements: `resources=VCPU:2,MEMORY_MB:2048,DISK_GB:40`
- Required traits: `required_traits=HW_CPU_X86_AVX2`
- Forbidden traits: `forbidden_traits=CUSTOM_UNDER_MAINTENANCE`
- Aggregate membership: `member_of=<aggregate-uuid>`

```bash
# Query allocation candidates directly (admin / troubleshooting)
openstack allocation candidate list \
  --resource VCPU=2 \
  --resource MEMORY_MB=2048 \
  --resource DISK_GB=40

# With trait requirement
openstack allocation candidate list \
  --resource VCPU=4 \
  --resource MEMORY_MB=8192 \
  --resource DISK_GB=100 \
  --required-trait HW_CPU_X86_AVX2

# With forbidden trait
openstack allocation candidate list \
  --resource VCPU=2 \
  --resource MEMORY_MB=4096 \
  --resource DISK_GB=20 \
  --forbidden-trait CUSTOM_UNDER_MAINTENANCE
```

When this returns zero results, the scheduler logs "No valid host was found" and the instance goes to ERROR state in cell0. This is the first place to check when diagnosing scheduling failures.

---

## Nested Resource Providers

A **nested resource provider** (NRP) is a resource provider that has a parent provider. The parent is typically the compute node; children represent sub-resources like NUMA nodes, SR-IOV physical functions, or NVMe namespaces.

Allocation candidates respect the tree: when a request requires resources from multiple providers in the same tree, the scheduler ensures all allocations are satisfied within the same tree (same physical host).

### Common NRP Use Cases

**SR-IOV Virtual Functions**
```
compute01 (root RP)
├── VCPU: 64, MEMORY_MB: 131072, DISK_GB: 2000
└── ens3f0-PF (child RP)
    └── SRIOV_NET_VF: 64 (CUSTOM_TRAIT_SRIOV_PF_1=true)
```

**NUMA Nodes**
```
compute01 (root RP)
└── compute01-numa0 (child RP)
    └── VCPU: 16, MEMORY_MB: 32768
└── compute01-numa1 (child RP)
    └── VCPU: 16, MEMORY_MB: 32768
```

**GPUs**
```
compute01 (root RP)
└── gpu0 (child RP)
    └── PGPU: 1, trait: CUSTOM_GPU_NVIDIA_A100
└── gpu1 (child RP)
    └── PGPU: 1, trait: CUSTOM_GPU_NVIDIA_A100
```

```bash
# Create a child resource provider
openstack resource provider create \
  --parent-provider <parent-provider-uuid> \
  compute01-numa0

# List all providers in a tree
openstack resource provider list --in-tree <root-provider-uuid>
```

---

## Reshaper

The **reshaper** API (`POST /placement/reshaper`) allows an operator to atomically update multiple resource provider inventories and all related allocations in a single transaction. This prevents a window where inventories are partially updated and allocations are momentarily invalid.

Use the reshaper when:
- Adding NRP structure to an existing provider that already has allocations.
- Moving inventory from a parent to a child provider.
- Introducing a new shared storage provider and redistributing DISK_GB allocations.

The nova-compute driver calls the reshaper automatically during certain upgrade steps (e.g. when introducing nested providers on an existing host). Operators rarely call it directly, but it is available via the Placement REST API at `POST /placement/reshaper`.

---

## Placement Configuration

Placement runs as a WSGI application under Apache httpd or uwsgi. Its configuration file is `/etc/placement/placement.conf`.

```ini
[DEFAULT]
log_dir = /var/log/placement

[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = placement-service-pass

[placement_database]
connection = mysql+pymysql://placement:placement-pass@controller/placement
```

```bash
# Sync the Placement database schema
placement-manage db sync

# Check current schema version
placement-manage db version
```
