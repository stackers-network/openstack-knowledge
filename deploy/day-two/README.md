# Day Two Operations

Managing a running OpenStack cloud — adding nodes, reconfiguring, upgrading, and operational tasks.

## Adding Compute Nodes

### 1. Prepare the new host

- Install OS (same as existing compute nodes)
- Configure management NIC with static IP
- Bring up external NIC (no IP)
- Ensure SSH access from deploy host

### 2. Update inventory

Add the new host to `inventory/hosts.yml`:

```ini
[compute]
comp-01 ansible_host=10.0.1.20
comp-02 ansible_host=10.0.1.21
comp-03 ansible_host=10.0.1.22    # existing
comp-04 ansible_host=10.0.1.23    # new
```

### 3. Bootstrap and deploy

```bash
# Bootstrap just the new host
kolla-ansible bootstrap-servers -i inventory/hosts.yml --limit comp-04

# Prechecks
kolla-ansible prechecks -i inventory/hosts.yml

# Pull images to the new host
kolla-ansible pull -i inventory/hosts.yml --limit comp-04

# Deploy to the new host
kolla-ansible deploy -i inventory/hosts.yml --limit comp-04
```

### 4. Verify

```bash
source /etc/kolla/admin-openrc.sh
openstack compute service list
# New compute agent should appear
openstack hypervisor list
# New hypervisor should show resources
```

## Reconfiguring Services

When changing `globals.yml` or service overrides:

```bash
# 1. Make config changes
vim globals.yml
# or edit files in config/

# 2. Always precheck
kolla-ansible prechecks -i inventory/hosts.yml

# 3. Reconfigure (restarts affected services)
kolla-ansible reconfigure -i inventory/hosts.yml

# 4. Targeted reconfigure (faster)
kolla-ansible reconfigure -i inventory/hosts.yml --tags nova
kolla-ansible reconfigure -i inventory/hosts.yml --tags neutron
```

**What reconfigure does:** Regenerates config files inside containers and restarts services that changed. Does NOT re-pull images or re-run initialization.

## Upgrading OpenStack Releases

### Planning

1. Check current version and target version
2. OpenStack supports **sequential upgrades only** (one release at a time)
3. SLURP releases (every other release) may support skip-level upgrades
4. Always read the release notes for breaking changes

### Upgrade Process

```bash
# 1. Backup database
kolla-ansible mariadb-backup -i inventory/hosts.yml

# 2. Update kolla-ansible to target version
pip install --upgrade 'kolla-ansible @ git+https://opendev.org/openstack/kolla-ansible@stable/2025.2'
kolla-ansible install-deps

# 3. Merge new passwords
#    New releases may add password fields — kolla-genpwd fills them
kolla-genpwd -p passwords.yml

# 4. Update globals.yml
#    - Change openstack_release if pinned
#    - Review new options in reference globals.yml
#    - Check for deprecated options

# 5. Prechecks with new config
kolla-ansible prechecks -i inventory/hosts.yml

# 6. Pull new images
kolla-ansible pull -i inventory/hosts.yml

# 7. Run upgrade (rolling restart with DB migrations)
kolla-ansible upgrade -i inventory/hosts.yml

# 8. Regenerate admin credentials
kolla-ansible post-deploy -i inventory/hosts.yml

# 9. Verify
source /etc/kolla/admin-openrc.sh
openstack compute service list
openstack network agent list
openstack versions show
```

### If Upgrade Fails

1. **Don't panic** — read the error
2. Fix the specific issue (usually a config problem or missing password)
3. Re-run the upgrade — it's designed to be re-runnable
4. If you need to rollback: reinstall old kolla-ansible version, re-run deploy with old images

### Important Upgrade Notes

- **Never CTRL-C** during an upgrade
- **Never use `--limit`** with upgrade — known bugs with partial upgrades
- **Always backup the database** before upgrading
- When `use_preconfigured_databases` is enabled, set `log_bin_trust_function_creators = 1` on the database
- Password merging: `kolla-mergepwd --old old_passwords.yml --new new_passwords.yml --final passwords.yml`

## Enabling New Services

```bash
# 1. Add to globals.yml
echo 'enable_octavia: "yes"' >> globals.yml

# 2. Prechecks
kolla-ansible prechecks -i inventory/hosts.yml

# 3. Pull images for the new service
kolla-ansible pull -i inventory/hosts.yml

# 4. Deploy just the new service
kolla-ansible deploy -i inventory/hosts.yml --tags octavia

# 5. Verify
openstack loadbalancer list
```

## Removing Hosts

```bash
# 1. Disable the compute service
openstack compute service set --disable comp-04 nova-compute

# 2. Migrate VMs off the host
openstack server migrate <vm-id> --live-migration

# 3. Remove from inventory
vim inventory/hosts.yml  # remove the host

# 4. Clean up containers on the removed host
ssh comp-04 docker stop $(docker ps -q --filter label=kolla_service)
ssh comp-04 docker rm $(docker ps -aq --filter label=kolla_service)
```

## Generating TLS Certificates

```bash
kolla-ansible certificates -i inventory/hosts.yml
kolla-ansible reconfigure -i inventory/hosts.yml
```

## Central Logging

When `enable_opensearch: "yes"`:

- Dashboards: `http://<vip>:<opensearch_dashboards_port>`
- Username: `opensearch`
- Password: from `passwords.yml` (`opensearch_dashboards_password`)
