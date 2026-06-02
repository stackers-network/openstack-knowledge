# Upgrade Learnings — Real Experience

Hard-won knowledge from real upgrades.

## The Biggest Gotcha: Stock Inventory Must Be Updated

**Every release can add new inventory groups.** Using the old inventory causes fatal precheck errors.

2025.2 added: `neutron-rpc-server`, `neutron-periodic-worker`, `neutron-ovn-maintenance-worker`. Using the 2025.1 inventory caused: `'dict object' has no attribute 'neutron-rpc-server'`.

**Fix:** Always pull the new stock inventory before upgrading and re-apply your host customizations:

```bash
source /opt/kolla-venv/bin/activate
KOLLA_SHARE="/opt/kolla-venv/share/kolla-ansible"

# Save your current host assignments
head -30 inventory/hosts > /tmp/my-hosts.txt

# Get new stock inventory
cp "$KOLLA_SHARE/ansible/inventory/all-in-one" inventory/hosts
# or: cp "$KOLLA_SHARE/ansible/inventory/multinode" inventory/hosts

# Re-apply your host assignments at the top
# (edit inventory/hosts, replace localhost with your hosts)
```

## Breaking Changes by Release

### 2025.1 → 2025.2

- **redis → valkey** — inventory group renamed, `enable_redis` → `enable_valkey` in globals
- **venus removed** — monitoring service and inventory group gone
- **VMware support removed** — `vmware_nsxv`, `vmware_nsxv3`, `vmware_nsxp`, `vmware_dvs` neutron plugins removed, along with related glance/cinder VMware backends
- **New inventory groups** — `neutron-rpc-server`, `neutron-periodic-worker`, `neutron-ovn-maintenance-worker`

### How to Discover Breaking Changes

```bash
# Diff old vs new stock globals
diff globals.yml.stock-old "$KOLLA_SHARE/etc_examples/kolla/globals.yml"

# Diff old vs new stock inventory (look for new groups)
diff <(grep '^\[' inventory/hosts.old | sort) \
     <(grep '^\[' "$KOLLA_SHARE/ansible/inventory/all-in-one" | sort)
```

## MariaDB Backup Before Upgrade

`kolla-ansible mariadb-backup` requires `enable_mariabackup: "yes"` in globals.yml. Without it, the backup playbook is a no-op.

For manual backup without mariabackup:
```bash
docker exec mariadb mariadb-dump --all-databases --single-transaction > backup.sql
# or with podman:
podman exec mariadb mariadb-dump --all-databases --single-transaction > backup.sql
```

## Password Merge is Safe

`kolla-genpwd -p passwords.yml` only fills empty fields — it preserves existing passwords. Safe to run on upgrade. New releases may add new password fields that need to be filled.

## Upgrade vs Deploy

**Always use `kolla-ansible upgrade`, never `deploy` for upgrades.** Upgrade handles:
- Rolling restarts (one service at a time)
- Database migrations
- RPC version pinning/unpinning
- Dependency-ordered service upgrades

Deploy does NOT do these things and can leave services in an inconsistent state.

## Upgrade Order is Automatic

kolla-ansible upgrades services in dependency order:
```
common → mariadb → memcached → rabbitmq → keystone → glance →
placement → OVS/OVN → nova → neutron → heat → horizon
```

**Never use `--limit` with upgrade** — known bugs with partial upgrades.

## Release Availability

Not all releases are on PyPI. Check git branches:
```bash
git ls-remote --heads https://opendev.org/openstack/kolla-ansible | grep stable
```

Install from git:
```bash
pip install 'kolla-ansible @ git+https://opendev.org/openstack/kolla-ansible@stable/2025.2'
```

## Proven Upgrade Checklist

```
1. Read release notes for target release
2. Backup database (enable_mariabackup or manual dump)
3. Record current state (health-check, service versions)
4. Git commit current configs

5. pip install new kolla-ansible version
6. kolla-ansible install-deps  (Galaxy collections update too)

7. Pull new stock inventory → re-apply host customizations
8. Pull new stock globals → diff for breaking changes
9. kolla-genpwd -p passwords.yml  (fills new fields, preserves existing)
10. Copy updated files to /etc/kolla/

11. kolla-ansible prechecks -i inventory/hosts
12. kolla-ansible pull -i inventory/hosts
13. kolla-ansible upgrade -i inventory/hosts
14. kolla-ansible post-deploy -i inventory/hosts

15. Verify: openstack compute service list, network agent list
16. Git commit updated configs
```
