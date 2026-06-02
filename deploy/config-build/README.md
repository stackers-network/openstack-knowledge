# Config Build

How to build kolla-ansible configuration from operator requirements. This is the first thing to do when setting up a new deployment.

## Output Files

A kolla-ansible deployment needs these files:

| File | Purpose |
|------|---------|
| `globals.yml` | Main kolla-ansible configuration — networking, storage, services, VIPs |
| `inventory/hosts.yml` | Ansible inventory — which hosts run which roles |
| `config/` | Per-service config overrides (INI files copied into containers) |
| `passwords.yml` | Generated credentials (encrypted at rest) |

## Building globals.yml

Ask the operator about their requirements, then build `globals.yml`. Here's the structure:

### Base Settings

```yaml
# Distribution and images
kolla_base_distro: "rocky"           # rocky, ubuntu, debian
# Images default to Quay.io pre-built containers
# openstack_release: "2025.1"       # defaults to the installed kolla-ansible version

# Docker namespace (default: quay.io/openstack.kolla)
# docker_namespace: "quay.io/openstack.kolla"
```

### Networking — The Most Important Decision

```yaml
# Management network interface (API endpoints, inter-service communication)
network_interface: "eth0"

# External network interface (Neutron external/provider networks)
# This interface should be UP but have NO IP address
neutron_external_interface: "eth1"

# VIP addresses — must be unused IPs on the management network
kolla_internal_vip_address: "10.0.1.100"
# kolla_external_vip_address: "10.0.1.101"  # if separate external API access needed

# Neutron plugin (OVN recommended for new deployments)
neutron_plugin_agent: "ovn"          # ovn | openvswitch | linuxbridge
```

**Key networking decisions to discuss with operator:**
- OVN (recommended) — distributed virtual router, built-in DHCP, no separate L3/DHCP agents
- Open vSwitch — traditional, well-understood, requires separate L3 and DHCP agents
- LinuxBridge — simplest, no OVS dependency, limited feature set

### Storage

```yaml
# Cinder volume backend
# enable_cinder: "yes"              # enabled by default
# cinder_backend_ceph: "yes"        # for Ceph integration

# Glance image backend
# glance_backend_ceph: "yes"        # store images in Ceph

# Nova instance storage
# nova_backend_ceph: "yes"          # live migration support, shared storage
```

**Storage options:**
- **LVM** (default) — local volumes on each node, simple, no shared storage
- **Ceph** — distributed, supports live migration, requires external Ceph cluster or enable_ceph
- **NFS** — shared storage via NFS mounts

### HA Settings

```yaml
# For HA deployments (3+ controllers)
# HAProxy + keepalived are enabled by default when multiple controllers exist
enable_haproxy: "yes"

# For single-node (AIO) deployments
# enable_haproxy: "no"
```

### Service Selection

Enable additional services as needed:

```yaml
# Core services (enabled by default)
# enable_keystone, enable_nova, enable_neutron, enable_glance, enable_cinder, enable_horizon

# Optional services
enable_heat: "yes"                   # orchestration
enable_octavia: "yes"                # load balancing
enable_magnum: "yes"                 # container orchestration
enable_manila: "yes"                 # shared filesystems
enable_designate: "yes"              # DNS
enable_barbican: "yes"               # key management
enable_skyline: "yes"                # modern dashboard (alternative to Horizon)
enable_prometheus: "yes"             # monitoring
enable_grafana: "yes"                # dashboards
enable_opensearch: "yes"             # centralized logging
```

### TLS

```yaml
# Internal TLS (recommended for production)
kolla_enable_tls_internal: "yes"
kolla_enable_tls_external: "yes"
# kolla_copy_ca_into_containers: "yes"  # if using private CA
```

### Multiple Config Files

For complex deployments, globals can be split across `/etc/kolla/globals.d/`:
```
/etc/kolla/globals.d/
├── 01-base.yml
├── 02-networking.yml
├── 03-storage.yml
└── 04-services.yml
```

## Building Inventory

### All-in-One (Development/Testing)

Use the built-in `all-in-one` inventory — all roles on localhost:

```ini
[control]
localhost ansible_connection=local

[network]
localhost ansible_connection=local

[compute]
localhost ansible_connection=local

[storage]
localhost ansible_connection=local

[monitoring]
localhost ansible_connection=local
```

### Multinode

```ini
[control]
ctrl-01 ansible_host=10.0.1.10
ctrl-02 ansible_host=10.0.1.11
ctrl-03 ansible_host=10.0.1.12

[network]
ctrl-01
ctrl-02
ctrl-03

[compute]
comp-01 ansible_host=10.0.1.20
comp-02 ansible_host=10.0.1.21
comp-03 ansible_host=10.0.1.22
comp-04 ansible_host=10.0.1.23

[storage]
stor-01 ansible_host=10.0.1.30
stor-02 ansible_host=10.0.1.31
stor-03 ansible_host=10.0.1.32

[monitoring]
ctrl-01

# Group mappings (kolla-ansible specific)
[baremetal:children]
control
network
compute
storage
monitoring
```

**Host group rules:**
- `control` — runs Keystone, Nova API, Neutron server, Glance, Cinder API, Horizon, MariaDB, RabbitMQ, Memcached, HAProxy
- `network` — runs Neutron agents (or OVN controllers). Usually same as control nodes
- `compute` — runs Nova compute, OVN/OVS agent
- `storage` — runs Cinder volume. Falls back to compute nodes if not specified
- `monitoring` — runs Prometheus, Grafana, OpenSearch. Usually first control node

### HA Requirements

For HA: minimum 3 control nodes (MariaDB Galera + RabbitMQ clustering). Odd number required for quorum.

## Per-Service Config Overrides

Kolla-ansible reads config overrides from `/etc/kolla/config/` on the deploy host. The local project keeps them in `config/`.

**Critical path rules:**
- Local project: `config/nova.conf`
- Must end up at: `/etc/kolla/config/nova.conf` on the host
- **NOT** `/etc/kolla/config/config/nova.conf` — watch out for `cp -r` creating nested directories

**Correct copy command:**
```bash
# From project root, copy CONTENTS of config/ into /etc/kolla/config/
scp config/* root@<host>:/etc/kolla/config/
# For subdirectories:
scp -r config/ root@<host>:/etc/kolla/
# OR on the host:
cp -r /path/to/project/config/* /etc/kolla/config/
```

**Directory structure** — mirrors the service layout:

```
config/                                    # local project dir
├── nova.conf                              # → /etc/kolla/config/nova.conf → all nova containers
├── nova/
│   └── nova-compute.conf                  # → nova_compute container only
├── neutron.conf                           # → all neutron containers
├── neutron/
│   └── ml2_conf.ini                       # → neutron_server + agents
├── cinder.conf                            # → all cinder containers
├── glance.conf                            # → all glance containers
└── haproxy/
    └── services.d/
        └── custom.cfg                     # custom HAProxy frontend/backend
```

**Format:** Standard INI format. These are merged with kolla's generated configs — they don't replace them.

**Common overrides:**

```ini
# config/nova.conf — CPU/RAM overcommit tuning
[DEFAULT]
cpu_allocation_ratio = 4.0
ram_allocation_ratio = 1.5
reserved_host_memory_mb = 4096
max_instances_per_host = 20

# config/neutron/ml2_conf.ini — ML2 type drivers
[ml2]
type_drivers = flat,vlan,vxlan,geneve

# config/cinder.conf — Volume type defaults
[DEFAULT]
default_volume_type = standard
```

**Applying overrides:**

```bash
# 1. Create/edit the override file locally
mkdir -p config
echo -e "[DEFAULT]\nmax_instances_per_host = 20" > config/nova.conf

# 2. Copy to host (watch the path — must be /etc/kolla/config/, not nested)
scp config/nova.conf root@<host>:/etc/kolla/config/nova.conf

# 3. Reconfigure just that service
ssh root@<host> 'source /opt/kolla-venv/bin/activate && kolla-ansible reconfigure -i inventory/hosts --tags nova'

# 4. Verify the override took effect
ssh root@<host> 'docker exec nova_compute grep max_instances_per_host /etc/nova/nova.conf'
```

## Generating Passwords

```bash
kolla-genpwd -p passwords.yml
```

This generates random passwords for all services. Run once per environment — each environment must have unique passwords.

**Encrypt at rest** (recommended):
```bash
sops --encrypt --in-place passwords.yml
```

## Validation

After building config, verify before deploying:

```bash
# Check kolla-ansible can parse everything
kolla-ansible prechecks -i inventory/hosts.yml

# Verify no conflicts
grep -r "CHANGEME\|TODO\|FIXME" globals.yml config/
```
