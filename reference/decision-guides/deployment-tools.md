# Choose a Deployment Tool

Several automation tools deploy and manage OpenStack. The choice affects the operator model, upgrade path, and operational day-2 experience more than any other single decision.

## Feature Comparison

| Tool | Production Ready | Containerized | Complexity | Active Development | Best For |
|---|---|---|---|---|---|
| Kolla-Ansible | Yes | Yes (Docker containers) | Medium | Active (OpenStack project) | Most production deployments — widest adoption |
| OpenStack-Ansible | Yes | LXC containers (processes in LXC) | Medium-High | Active (OpenStack project) | Bare metal isolation with LXC, no Docker dependency |
| Charmed OpenStack (Juju) | Yes | No (Juju charms, snaps/debs) | High | Active (Canonical) | Ubuntu-centric organizations, Juju ecosystem familiarity |
| Kayobe | Yes | Yes (via Kolla-Ansible) | Medium-High | Active (OpenStack project) | Bare-metal provisioning + Kolla deployment via Bifrost |
| DevStack | No | No | Low | Active | Development and CI only |
| Manual (packages) | Depends | No | Very High | N/A | Learning, single-node lab, tiny edge deployments |
| TripleO / Red Hat Director | Deprecated | Yes (containers via Heat) | Very High | Discontinued (2023) | Legacy migrations only |

## Tool Profiles

### Kolla-Ansible

**What it is**: Ansible playbooks that deploy OpenStack services as Docker containers, one container per service. Container images are built and published by the Kolla project (available from Docker Hub and Quay.io).

**Architecture**: All services run in Docker containers. The Kolla-Ansible operator runs playbooks from a deploy node against target nodes.

**Strengths**:
- Largest community and most documentation
- All OpenStack services supported
- Clean upgrade path (pull new images, rolling restart)
- Container isolation makes service replacement clean
- `kolla-genpwd` and `passwords.yml` for secrets management

**Weaknesses**:
- Docker on every node (though Podman support is improving)
- Configuration scattered across Ansible inventory + globals.yml + service-specific overrides

**Key files**:
- `/etc/kolla/globals.yml` — global config (network interfaces, VIP, backends)
- `/etc/kolla/passwords.yml` — all generated passwords
- `/etc/kolla/config/<service>/` — service-specific config overrides

```bash
# Bootstrap
kolla-ansible install-deps
kolla-ansible -i inventory/multinode bootstrap-servers
kolla-ansible -i inventory/multinode prechecks
kolla-ansible -i inventory/multinode deploy

# Upgrade
kolla-ansible -i inventory/multinode pull      # pull new images
kolla-ansible -i inventory/multinode prechecks
kolla-ansible -i inventory/multinode upgrade

# Certificate management
kolla-ansible -i inventory/multinode certificates

# Check service health post-deploy
kolla-ansible -i inventory/multinode check
```

For detailed Kolla-Ansible guidance, see the `openstack-kolla` agentic stack.

### OpenStack-Ansible (OSA)

**What it is**: Ansible playbooks that deploy OpenStack services as Python processes inside LXC containers on bare-metal hosts. No Docker dependency.

**Architecture**: Each service group (nova-compute, keystone, etc.) runs in dedicated LXC containers on physical hosts. The network bridge between LXC and physical is managed by OSA.

**Strengths**:
- No Docker — all standard Python packages in LXC
- Fine-grained control over container placement
- Excellent documentation for bare-metal deployments
- Galera/MariaDB and RabbitMQ clusters managed natively

**Weaknesses**:
- More complex networking (bridges, LXC network config)
- Larger footprint per host (LXC overhead)
- Upgrade complexity with LXC container rebuilds

**Key files**:
- `/etc/openstack_deploy/openstack_user_config.yml` — host layout
- `/etc/openstack_deploy/user_variables.yml` — overrides
- `/etc/openstack_deploy/user_secrets.yml` — secrets

```bash
# Initial deployment
cd /opt/openstack-ansible
./scripts/bootstrap-ansible.sh
./scripts/bootstrap-aio.sh  # for all-in-one only
openstack-ansible setup-hosts.yml
openstack-ansible setup-infrastructure.yml
openstack-ansible setup-openstack.yml

# Upgrade (major version)
./scripts/run-upgrade.sh

# Ad-hoc run
openstack-ansible os-nova-install.yml --tags nova-config
```

### Charmed OpenStack (Juju)

**What it is**: Canonical's Juju operator framework deploying OpenStack via "charms" — encapsulated operational knowledge units. Charms manage installation, configuration, scaling, and upgrades.

**Architecture**: Juju controller manages units (deployed application instances) across machines. Relations between charms handle inter-service wiring (e.g., nova-compute relates to rabbitmq-server and mysql-innodb-cluster).

**Strengths**:
- Excellent for Ubuntu-centric organizations already using Juju
- Strong MAAS integration for bare-metal provisioning
- Declarative bundle files for reproducible deployments
- Operator actions for common tasks (`juju run nova-compute/0 pause`)

**Weaknesses**:
- Juju learning curve is significant
- Less portable outside Ubuntu ecosystem
- Debugging requires Juju knowledge on top of OpenStack knowledge
- Not containerized — runs deb packages or snaps directly

```bash
# Deploy from bundle
juju deploy ./openstack-bundle.yaml

# Check status
juju status --color

# Scale out nova-compute
juju add-unit nova-compute -n 2 --to <MAAS_TAG>

# Upgrade a charm
juju upgrade-charm nova-compute --channel 2025.1/stable

# Run an action
juju run nova-compute/0 openstack-upgrade

# SSH to a unit
juju ssh nova-compute/0
```

### Kayobe

**What it is**: An Ansible-based tool for provisioning bare-metal infrastructure (via Bifrost/Ironic) and then deploying OpenStack via Kolla-Ansible. Kayobe sits above Kolla-Ansible, handling the physical layer first.

**Architecture**: Kayobe manages the seed node (Bifrost), overcloud hardware inventory, OS provisioning, and then hands off to Kolla-Ansible for OpenStack deployment.

**Strengths**:
- End-to-end automation from bare metal to working cloud
- Bifrost integration for IPMI/PXE provisioning
- Good for repeatable rack deployments

**Weaknesses**:
- Double learning curve (Kayobe + Kolla-Ansible)
- More moving parts than Kolla-Ansible alone
- Best when you control bare metal; less useful for VM-based labs

```bash
# Initialize environment
kayobe control host bootstrap
kayobe overcloud inventory discover
kayobe overcloud provision

# Deploy OpenStack (via Kolla)
kayobe overcloud service deploy

# Upgrade
kayobe overcloud service upgrade
```

### DevStack

**For development and CI only.** Do not use in production.

```bash
git clone https://opendev.org/openstack/devstack
cd devstack
cp samples/local.conf local.conf
# Edit local.conf: ADMIN_PASSWORD, DATABASE_PASSWORD, HOST_IP, etc.
./stack.sh
```

### Manual Package Deployment

For learning purposes only. Package names vary by OS and release.

```bash
# Ubuntu — install service packages
apt install nova-api nova-compute nova-scheduler nova-conductor \
  neutron-server neutron-openvswitch-agent neutron-l3-agent neutron-dhcp-agent \
  cinder-api cinder-volume cinder-scheduler \
  glance keystone python3-openstackclient

# Run DB migrations
nova-manage api_db sync
nova-manage db sync
neutron-db-manage upgrade heads
cinder-manage db sync
glance-manage db_sync
keystone-manage db_sync
```

## Decision Matrix

Answer these questions to choose:

| Question | If Yes → Consider |
|---|---|
| Do you want Docker containers? | Kolla-Ansible |
| Do you want no Docker? | OpenStack-Ansible |
| Are you a Canonical/Ubuntu shop? | Charmed OpenStack |
| Do you provision bare metal from scratch? | Kayobe (uses Kolla under the hood) |
| Is this a dev or CI environment? | DevStack |
| Are you learning how OpenStack works? | Manual packages (then switch to Kolla for real deployments) |
| Is this a legacy Red Hat/CentOS deployment? | Evaluate migration from TripleO to Kolla-Ansible |

## Migration from TripleO

TripleO (and its Red Hat Director derivative) was deprecated and support ended in 2023. Clouds still running TripleO deployments should plan migration to Kolla-Ansible or OpenStack-Ansible:

1. Document the current topology (controller, compute, storage roles).
2. Export Keystone data (`keystone-manage export`), database dumps, and Ceph cluster configs.
3. Deploy a parallel environment using Kolla-Ansible.
4. Migrate workloads progressively (live-migrate VMs, replicate Ceph pools).
5. Cut over DNS and VIP.

There is no automated TripleO → Kolla migration tool. This is a full redeployment with data migration.
