# OpenStack Deployment Overview

Survey of all major OpenStack deployment methods. Each method's scope, maturity, and trade-offs are covered so operators can select the right approach before deploying.

**Recommendation up front**: For new production deployments, use **kolla-ansible** or **OpenStack-Ansible**. See the `openstack-kolla` agentic stack for complete kolla-ansible guidance.

---

## Manual / Package-Based Installation

Install OpenStack services directly from distribution packages (APT, DNF/YUM) or from PyPI (`pip`). Each service is configured individually by editing its config files under `/etc`.

### When to Use

- Learning the internals of OpenStack; building understanding of each service.
- Single-node proof-of-concept that is not DevStack.
- Environments with strict change management that prohibit automation tools.

### Not for Production

Manual installs do not provide:
- Automated upgrades or rollbacks.
- HA configuration (HAProxy, Pacemaker, Galera).
- Repeatable re-deployment.

### Install from Packages (Ubuntu 22.04 / 24.04 — Epoxy 2025.1)

```bash
# Add the OpenStack UCA (Ubuntu Cloud Archive) repo
add-apt-repository cloud-archive:epoxy
apt update

# Keystone
apt install -y keystone python3-openstackclient
# Nova
apt install -y nova-api nova-conductor nova-scheduler nova-novncproxy \
               nova-compute libvirt-daemon-system
# Neutron
apt install -y neutron-server neutron-plugin-ml2 \
               neutron-openvswitch-agent neutron-l3-agent \
               neutron-dhcp-agent neutron-metadata-agent
# Glance
apt install -y glance
# Cinder
apt install -y cinder-api cinder-scheduler cinder-volume
# Placement
apt install -y placement-api
```

Each service is then configured via its `/etc/<service>/<service>.conf` file and database-synced:

```bash
nova-manage db sync
neutron-db-manage upgrade heads
glance-manage db_sync
cinder-manage db sync
keystone-manage db_sync
keystone-manage bootstrap ...
```

### Install from pip (any Linux)

```bash
python3 -m venv /opt/openstack/nova
/opt/openstack/nova/bin/pip install nova
```

Not recommended for production — no OS-level service management, no security patches from the distro.

---

## DevStack

**Purpose**: Development and testing only. Not for production.

DevStack is a shell-script-based tool that deploys a single-node OpenStack cloud on an Ubuntu or Fedora VM in 15–30 minutes. It clones each project from Git (HEAD of a branch) and runs services in `screen` windows.

### Constraints

- Destroys and rebuilds the system on each `stack.sh` run.
- Not upgradeable.
- Not HA-capable.
- Designed to be throw-away; data is lost on `unstack.sh`.
- Single machine only (multi-node is possible but complex and not supported for production use).

### Quick Start

```bash
# On a fresh Ubuntu 22.04 / 24.04 VM (minimum 8 GB RAM, 40 GB disk)
git clone https://opendev.org/openstack/devstack
cd devstack
cp samples/local.conf local.conf
# Edit local.conf: set HOST_IP, ADMIN_PASSWORD, DATABASE_PASSWORD, etc.
./stack.sh
```

Minimal `local.conf`:

```ini
[[local|localrc]]
HOST_IP=192.168.122.10
ADMIN_PASSWORD=supersecret
DATABASE_PASSWORD=supersecret
RABBIT_PASSWORD=supersecret
SERVICE_PASSWORD=supersecret
ENABLED_SERVICES=key,n-api,n-cpu,n-cond,n-sch,n-novnc,placement-api,g-api,c-sch,c-api,c-vol,q-svc,q-agt,q-l3,q-dhcp,q-meta,horizon,tempest
```

### After Stack

```bash
source openrc admin admin
openstack service list
openstack endpoint list
```

---

## Kolla-Ansible

**Purpose**: Production-grade containerized deployment. Uses Docker containers for every OpenStack service, managed with Ansible.

### Key Characteristics

- Every service runs in its own Docker container from an official Kolla image.
- Services are upgraded by pulling new container images and restarting containers (no OS package changes).
- Supports single-node (all-in-one) and multi-node (inventory file).
- Supports HA (HAProxy + Keepalived for VIPs, Galera for DB, RabbitMQ cluster).
- Active upstream community; supports every current OpenStack release.
- OS support: Ubuntu 22.04, Ubuntu 24.04, Debian 12, Rocky Linux 9, CentOS Stream 9.

### Reference

For complete kolla-ansible guidance — inventory setup, `globals.yml` configuration, deployment commands, upgrade procedures, and day-two operations — see the **`openstack-kolla` agentic stack**.

### Conceptual Workflow

```bash
pip install kolla-ansible
kolla-ansible install-deps
# Edit /etc/kolla/globals.yml (network interface, VIP address, enable_* flags)
# Edit /etc/kolla/passwords.yml (generated via kolla-genpwd)
kolla-genpwd
# For multi-node: populate /etc/kolla/multinode inventory
kolla-ansible bootstrap-servers -i multinode
kolla-ansible prechecks -i multinode
kolla-ansible deploy -i multinode
kolla-ansible post-deploy -i multinode
# Source credentials
. /etc/kolla/admin-openrc.sh
```

### Key Config Files

| File | Purpose |
|---|---|
| `/etc/kolla/globals.yml` | Main deployment configuration (network, VIP, enabled services) |
| `/etc/kolla/passwords.yml` | Auto-generated service passwords |
| `/etc/kolla/multinode` | Ansible inventory for multi-node deployments |
| `/etc/kolla/config/<service>/` | Per-service config overrides merged into containers |

---

## OpenStack-Ansible (OSA)

**Purpose**: Production-grade Ansible-based deployment on bare metal or in LXC containers.

### Key Characteristics

- Each service runs either directly on the host or inside an LXC container.
- Deep Ansible integration — every aspect of the deployment is an Ansible role.
- Supports full HA: Galera, RabbitMQ cluster, HAProxy.
- Active community; long-standing production use.
- OS support: Ubuntu 22.04, Ubuntu 24.04.
- More complex than kolla-ansible; steeper learning curve.

### Quick Start

```bash
# On the deployment host (can be one of the target hosts or a separate host)
git clone -b 2025.1.0 https://opendev.org/openstack/openstack-ansible /opt/openstack-ansible
cd /opt/openstack-ansible
scripts/bootstrap-ansible.sh
scripts/bootstrap-aio.sh          # For all-in-one; skip for multi-node
# For multi-node: populate /etc/openstack_deploy/openstack_user_config.yml
cd /opt/openstack-ansible/playbooks
openstack-ansible setup-hosts.yml
openstack-ansible setup-infrastructure.yml
openstack-ansible setup-openstack.yml
```

### Key Config Files

| File | Purpose |
|---|---|
| `/etc/openstack_deploy/openstack_user_config.yml` | Inventory: hosts, networks, services |
| `/etc/openstack_deploy/user_variables.yml` | Variable overrides |
| `/etc/openstack_deploy/user_secrets.yml` | Service passwords |

### LXC vs Bare Metal

By default OSA places each service group (e.g., all Nova API instances) in LXC containers. This provides isolation and repeatability. Container mode can be disabled for services where bare-metal deployment is preferred.

---

## TripleO / Director

**Status**: Deprecated upstream. Not recommended for new deployments.

TripleO (OpenStack-on-OpenStack) was Red Hat OpenStack Platform's deployment method. It uses an "undercloud" (a small OpenStack installation) to deploy and manage the "overcloud" (the production cloud) via Heat templates and Ironic.

- Upstream TripleO development ended in 2023.
- Red Hat OpenStack Services on OpenShift (RHOSO) replaces it for RHEL-based deployments.
- Existing TripleO deployments should plan migration to an alternative tool.

**Do not use TripleO for new deployments.**

---

## Kayobe

**Purpose**: End-to-end deployment tool that extends kolla-ansible with bare metal provisioning, switch configuration, and seed/infra management.

### Key Characteristics

- Built on kolla-ansible — all kolla-ansible functionality is available.
- Adds: bare metal provisioning via **Bifrost** (Ironic-based PXE boot), network switch configuration, seed host bootstrapping.
- Opinionated directory layout and configuration hierarchy.
- Best suited for greenfield deployments where you also control the physical infrastructure.
- OS support: Ubuntu 22.04, Ubuntu 24.04.

### Conceptual Workflow

```bash
pip install kayobe
# Initialise a Kayobe configuration repository (separate from the codebase)
kayobe configure init
# Edit config/kayobe.yml, networks.yml, inventory/
kayobe control host bootstrap
kayobe seed host configure
kayobe seed service deploy       # Deploys Bifrost on the seed
kayobe overcloud inventory discover  # PXE-boots and inspects bare metal nodes
kayobe overcloud host configure
kayobe overcloud service deploy  # Runs kolla-ansible deploy under the hood
```

### Relationship to kolla-ansible

Kayobe invokes `kolla-ansible` commands internally. `globals.yml` and container images are the same. kolla-ansible expertise transfers directly to Kayobe deployments.

---

## Charmed OpenStack (Juju / Canonical)

**Purpose**: Canonical's operator framework for deploying and operating OpenStack using Juju charms.

### Key Characteristics

- Each OpenStack service is packaged as a Juju charm (an operator).
- Juju handles service relations, upgrades, scaling, and day-two operations.
- Tightly integrated with MAAS (Metal as a Service) for bare metal provisioning.
- Production-grade HA built into charm relations.
- OS support: Ubuntu only (22.04, 24.04).
- Supported by Canonical with commercial support options.

### Quick Start Concept

```bash
# Install Juju and bootstrap a controller
juju bootstrap maas-cloud maas-controller
juju add-model openstack
# Deploy a bundle
juju deploy openstack-base   # or a custom bundle YAML
juju status --watch 5s
```

Charms communicate through relations, e.g., the nova-compute charm relates to the mysql-innodb-cluster charm for its database connection automatically.

---

## Comparison Table

| Method | Production Ready | Complexity | Containerized | HA Support | Community Activity | Supported OS |
|---|---|---|---|---|---|---|
| Manual / Packages | No | Low (single node) / Very High (HA) | No | Manual only | N/A | Any (distro packages) |
| DevStack | No | Very Low | No | No | Very Active | Ubuntu, Fedora |
| Kolla-Ansible | Yes | Medium | Yes (Docker) | Yes (built-in) | Very Active | Ubuntu 22/24, Debian 12, Rocky 9 |
| OpenStack-Ansible | Yes | High | Partial (LXC) | Yes (built-in) | Active | Ubuntu 22/24 |
| TripleO / Director | **Deprecated** | Very High | Partial | Yes | Dead upstream | RHEL/CentOS (legacy) |
| Kayobe | Yes | High | Yes (Docker via kolla) | Yes | Active | Ubuntu 22/24 |
| Charmed OpenStack | Yes | Medium (Juju model) | No | Yes | Active (Canonical) | Ubuntu only |

---

## Recommendation

For new production deployments, use **kolla-ansible** or **OpenStack-Ansible (OSA)**:

- **kolla-ansible**: Prefer when you want containerized services, fast upgrades via image replacement, and broad OS support. Lower operational complexity than OSA for most teams.
- **OSA**: Prefer when you want a fully Ansible-native approach without Docker, or when your team has deep Ansible expertise and prefers LXC-based isolation.
- **Kayobe**: Prefer when you control the physical hardware and need integrated bare metal provisioning as part of the same workflow.

For complete kolla-ansible operational guidance, see the **`openstack-kolla` agentic stack**.

---

## Cross-Reference

- Architecture overview → `core/foundation/architecture/`
- Common configuration patterns → `core/foundation/configuration/`
- Health verification after deployment → `operations/health-check/`
- Upgrades after initial deployment → `operations/upgrades/`
