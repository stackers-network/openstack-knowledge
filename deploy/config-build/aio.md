# All-in-One (AIO) Configuration

Specific guidance for single-node deployments.

## Key AIO Settings

For an all-in-one deployment, several settings differ from multinode:

```yaml
# VIP address = the host's own IP (no floating VIP on a single node)
kolla_internal_vip_address: "10.0.200.79"  # same as the host's management IP

# Disable HAProxy — no need for load balancing on a single node
enable_haproxy: "no"
```

**Why disable HAProxy?** On a single-node deployment, HAProxy would just proxy to localhost. Disabling it removes an unnecessary layer and avoids the need for keepalived/VRRP.

## AIO Inventory

Use INI format (kolla-ansible's native format):

```ini
# inventory/hosts
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

Or for a remote AIO (deploying from a separate machine):

**Important:** Use the stock kolla inventory as a base (it has ~176 groups that kolla expects). Only modify the host assignments at the top:

```bash
KOLLA_SHARE="/opt/kolla-venv/share/kolla-ansible"
cp "$KOLLA_SHARE/ansible/inventory/all-in-one" inventory/hosts
```

Then edit the top of `inventory/hosts` to point at the remote host:

```ini
# At the top of the stock all-in-one inventory, change localhost to your host:
[control]
aio ansible_host=10.0.200.79 ansible_user=root

[network]
aio

[compute]
aio

[storage]
aio

[monitoring]
aio

# Keep [deployment] as localhost — this is where ansible runs FROM
[deployment]
localhost ansible_connection=local

# DO NOT modify the rest of the file — it has ~170 group mappings kolla needs
```

**Key pattern:** Define the host once in `[control]` with `ansible_host=` and `ansible_user=`, then reference by alias in all other groups.

## Expected Timing (AIO)

Based on real deployments:

| Step | Time (cold) | Time (cached) |
|------|-------------|---------------|
| Bootstrap | 5-10 min | ~1 min |
| Prechecks | 2-3 min | ~2 min |
| Pull | 10-30 min | ~3 min |
| Deploy | 15-45 min | ~15 min |
| Post-deploy | ~30 sec | ~30 sec |

## Host Preparation Script

Create a `prepare-host.sh` to run ON the target host before deploying:

```bash
#!/usr/bin/env bash
# Run this ON the target host as root
set -euo pipefail

echo "=== 1. Ensure external interface is UP with no IP ==="
ip link set <external_interface> up
ip addr flush dev <external_interface> 2>/dev/null || true

echo "=== 2. Install prerequisites ==="
# Rocky Linux / CentOS
dnf install -y python3 python3-pip python3-devel gcc libffi-devel openssl-devel git

# Ubuntu
# apt update && apt install -y python3 python3-pip python3-venv python3-dev gcc libffi-dev libssl-dev git

echo "=== 3. Create kolla-ansible virtual environment ==="
python3 -m venv /opt/kolla-venv
source /opt/kolla-venv/bin/activate
pip install -U pip

echo "=== 4. Install kolla-ansible ==="
pip install 'kolla-ansible @ git+https://opendev.org/openstack/kolla-ansible@stable/2025.1'

echo "=== 5. Install Ansible Galaxy dependencies ==="
kolla-ansible install-deps

echo "=== 6. Set up /etc/kolla ==="
mkdir -p /etc/kolla
cp globals.yml /etc/kolla/globals.yml
if [ -d config ]; then
    cp -r config/ /etc/kolla/config/
fi

echo "=== 7. Generate passwords ==="
KOLLA_SHARE="/opt/kolla-venv/share/kolla-ansible"
if [ ! -f /etc/kolla/passwords.yml ]; then
    # kolla-genpwd requires the template file to exist first
    cp "$KOLLA_SHARE/etc_examples/kolla/passwords.yml" /etc/kolla/passwords.yml
    kolla-genpwd -p /etc/kolla/passwords.yml
    echo "passwords.yml generated — copy back to your local repo!"
else
    echo "passwords.yml already exists, skipping"
fi

echo "=== 8. Set up self-SSH (required for AIO) ==="
if [ ! -f ~/.ssh/id_rsa ]; then
    ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa
fi
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
sort -u ~/.ssh/authorized_keys -o ~/.ssh/authorized_keys
HOST_IP=$(hostname -I | awk '{print $1}')
ssh -o StrictHostKeyChecking=no root@"$HOST_IP" 'echo "Self-SSH OK"'

echo ""
echo "=== 9. Check for leftover state ==="
# Check for OVS bridges, Docker containers from previous attempts
if command -v ovs-vsctl &>/dev/null; then
    echo "OVS bridges: $(ovs-vsctl list-br 2>/dev/null || echo 'none')"
fi
if command -v docker &>/dev/null; then
    KOLLA_CONTAINERS=$(docker ps -aq --filter label=kolla_service 2>/dev/null | wc -l)
    echo "Existing kolla containers: $KOLLA_CONTAINERS"
fi

echo ""
echo "=== Preparation complete ==="
echo ""
echo "Next steps:"
echo "  source /opt/kolla-venv/bin/activate"
echo "  ./deploy.sh"
echo ""
echo "IMPORTANT: Copy /etc/kolla/passwords.yml back to your local repo!"
```

## Deploy Script

Create a `deploy.sh` for repeatable deployments:

```bash
#!/usr/bin/env bash
set -euo pipefail

INVENTORY="inventory/hosts"

echo "=== Step 1: Bootstrap servers ==="
kolla-ansible bootstrap-servers -i "$INVENTORY"

echo "=== Step 2: Prechecks ==="
kolla-ansible prechecks -i "$INVENTORY"

echo "=== Step 3: Pull images ==="
kolla-ansible pull -i "$INVENTORY"

echo "=== Step 4: Deploy ==="
kolla-ansible deploy -i "$INVENTORY"

echo "=== Step 5: Post-deploy ==="
kolla-ansible post-deploy -i "$INVENTORY"

echo "=== Deployment complete ==="
echo "Run 'source /etc/kolla/admin-openrc.sh' to use the OpenStack CLI."
```

## AIO Resource Requirements

- **Minimum:** 8GB RAM, 40GB disk, 2 NICs
- **Recommended:** 16GB+ RAM, 100GB+ SSD, 2 NICs
- **OS:** Rocky Linux 9/10, Ubuntu 22.04/24.04

## Using the Full globals.yml Template

For AIO deployments, it's often better to start with the **full kolla-ansible globals.yml template** rather than a minimal one. This gives the operator visibility into all available options:

```bash
# Get the full template
pip show kolla-ansible | grep Location
# Copy from: <location>/kolla_ansible/etc_examples/kolla/globals.yml
```

Then uncomment and modify only what you need. The key settings for AIO are:
- `kolla_base_distro`
- `kolla_internal_vip_address` (= host's IP)
- `network_interface`
- `neutron_external_interface`
- `neutron_plugin_agent`
- `enable_haproxy: "no"`
