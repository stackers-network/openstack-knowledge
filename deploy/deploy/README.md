# Deploy

The full kolla-ansible deployment lifecycle. Every command in order, what to expect, and what to do when things fail.

## Prerequisites

Before deploying:
1. All hosts are reachable via SSH from the deploy host
2. `globals.yml` is configured (see config-build skill)
3. `inventory/hosts` has all hosts and groups (use stock kolla inventory as base)
4. `passwords.yml` is generated (seeded from template, then `kolla-genpwd`)
5. Ansible and kolla-ansible are installed in a virtual environment
6. Check for **leftover state** on hosts from previous attempts (OVS bridges, Docker containers, etc.)

## Important: Always Activate the Venv

`openstack`, `kolla-ansible`, and `kolla-genpwd` are all inside `/opt/kolla-venv/`. The `admin-openrc.sh` only sets environment variables — it does NOT activate the venv. Every command needs:

```bash
source /opt/kolla-venv/bin/activate
source /etc/kolla/admin-openrc.sh  # only after post-deploy
```

## Installation (on the deploy host)

```bash
# Create virtual environment
python3 -m venv /opt/kolla-venv
source /opt/kolla-venv/bin/activate

# Install kolla-ansible
pip install -U pip
pip install 'kolla-ansible @ git+https://opendev.org/openstack/kolla-ansible@stable/2025.1'

# Install Ansible Galaxy dependencies
kolla-ansible install-deps

# Set up config directory
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
cp globals.yml /etc/kolla/globals.yml
cp passwords.yml /etc/kolla/passwords.yml
cp -r config/ /etc/kolla/config/
```

## Host Requirements

- **OS:** Ubuntu 22.04/24.04, Rocky Linux 9, Debian 12
- **Minimum:** 2 NICs, 8GB RAM, 40GB disk per host
- **Production:** 2 NICs, 64GB+ RAM (controllers), 32GB+ RAM (compute), SSD for OS
- **Networking:** management NIC with IP, external NIC up but no IP

### Host Preparation (manual, before bootstrap)

```bash
# On each host:
# 1. Configure management interface with static IP
# 2. Bring up external interface without IP
ip link set eth1 up

# 3. Ensure SSH access from deploy host
# 4. Ensure Python 3 is installed
```

## Deployment Steps

### Step 1: Bootstrap Servers

Installs Docker, configures host networking, prepares hosts for deployment.

```bash
kolla-ansible bootstrap-servers -i inventory/hosts.yml
```

**Expected:** Installs Docker, creates kolla user, configures sysctl. Takes 5-10 minutes per host.

**If it fails:**
- Check SSH connectivity: `ansible -i inventory/hosts.yml all -m ping`
- Check sudo access on remote hosts
- Check network connectivity between hosts
- Read the failed task name — it tells you which host and what step failed

**Important:** If bootstrap fails partway through, fix the issue and re-run. It's idempotent.

### Step 2: Prechecks

Validates the deployment configuration before actually deploying.

```bash
kolla-ansible prechecks -i inventory/hosts.yml
```

**Expected:** Checks ports, Docker status, config validity. Takes 2-3 minutes.

**If it fails:**
- Port conflict: another service using a required port
- Docker not running: bootstrap didn't complete
- Config error: globals.yml has invalid value
- Missing inventory group: kolla-ansible expects certain groups

### Step 3: Pull Images

Downloads container images to all hosts. Do this before deploy to catch image issues early.

```bash
kolla-ansible pull -i inventory/hosts.yml
```

**Expected:** Downloads 30-60+ container images per host. Takes 10-30 minutes depending on bandwidth.

**If it fails:**
- Network connectivity to Quay.io (or configured registry)
- Disk space on hosts
- Docker daemon issues

### Step 4: Deploy

The main deployment. Creates containers, runs initialization, starts services.

```bash
kolla-ansible deploy -i inventory/hosts.yml
```

**Expected:** Deploys all services. Takes 15-45 minutes for a full deployment.

**If it fails:**
- Read the error carefully — the failed task tells you which service
- Check the specific service container logs: `docker logs kolla_<service>`
- Common: database not ready (MariaDB Galera needs all nodes), RabbitMQ cluster formation
- For multinode: check inter-host connectivity on management network
- **DO NOT run `destroy` and start over** unless you understand what failed. Most failures can be fixed and re-deployed.

**Re-running deploy:** It's safe to re-run deploy after fixing an issue. It's idempotent — already-deployed services are skipped.

### Step 5: Post-Deploy

Generates admin credentials and initialization scripts.

```bash
kolla-ansible post-deploy -i inventory/hosts.yml
```

**Expected:** Creates `/etc/kolla/clouds.yaml` and `admin-openrc.sh` for CLI access.

### Step 6: Verify

```bash
# Source admin credentials
source /etc/kolla/admin-openrc.sh
# Or use clouds.yaml:
export OS_CLOUD=kolla-admin

# Verify services
openstack token issue
openstack service list
openstack compute service list
openstack network agent list
openstack volume service list

# Quick smoke test
openstack image list
openstack network list
openstack flavor list
```

## Reconfigure (Applying Config Changes)

When you change `globals.yml` or files in `config/`:

```bash
# Always precheck first
kolla-ansible prechecks -i inventory/hosts.yml

# Apply changes (restarts affected containers)
kolla-ansible reconfigure -i inventory/hosts.yml
```

**What reconfigure does:** Regenerates service configs inside containers and restarts services that changed. It does NOT rebuild containers or re-run initialization.

**Targeted reconfigure** (faster — only specific service):
```bash
kolla-ansible reconfigure -i inventory/hosts.yml --tags nova
kolla-ansible reconfigure -i inventory/hosts.yml --tags neutron
```

## Destroying an Environment

```bash
# WARNING: This removes ALL containers and volumes. Data is lost.
kolla-ansible destroy -i inventory/hosts.yml --yes-i-really-really-mean-it
```

**Never run this without explicit operator confirmation.**

## Important Notes

- **Never CTRL-C** during a deploy or upgrade. Let it finish or fail naturally.
- **Never use `--limit`** with kolla-ansible upgrade — it has known bugs with partial runs.
- All kolla-ansible commands accept `-e @passwords.yml` if passwords aren't in `/etc/kolla/`.
- Use `--configdir` to point to a custom config directory instead of `/etc/kolla/`.
- Multiple inventories can be specified with repeated `-i` flags.
