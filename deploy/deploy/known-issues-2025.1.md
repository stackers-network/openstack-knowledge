# Known Issues — kolla-ansible 21.0.0 (2025.1 Epoxy)

Apply these workarounds BEFORE deploying. These are confirmed issues from real deployments.

## Pre-Deploy Workarounds (add to globals.yml)

### 1. ProxySQL crashes with `KeyError: 'schemaname'`

Nova cell database rule missing `schemaname` field. Fix merged upstream but not in 21.0.0.

```yaml
# globals.yml
enable_proxysql: "no"
```

For AIO, also set `database_address` directly to the host's management IP.

### 2. Keystone WSGI provider must be `apache`

Quay.io kolla images use httpd, not uwsgi. The default `keystone_wsgi_provider: "uwsgi"` generates incompatible config.

```yaml
# globals.yml
keystone_wsgi_provider: "apache"
```

### 3. `kolla_install_type` must be `binary`

Quay.io images bundle httpd. Setting `source` generates uwsgi configs that don't match the images.

```yaml
# globals.yml
kolla_install_type: "binary"
```

### 4. Rocky Linux 10 needs explicit distro version

The default `kolla_base_distro_version` for rocky is "9". If deploying on Rocky 10, all pulled images will be rocky-9 based unless you set this explicitly.

```yaml
# globals.yml
kolla_base_distro: "rocky"
kolla_base_distro_version: "10"
```

Rocky Linux 10 IS supported in kolla-ansible 2025.1+.

### 5. Inventory must include all kolla groups

Minimal inventory with just control/network/compute/storage fails with errors like `object has no attribute 'neutron-bgp-dragent'`. Kolla-ansible expects ~176 host groups.

**Workaround:** Use kolla's full reference inventory as a base:

```bash
KOLLA_PATH=$(python3 -c "import kolla_ansible; import os; print(os.path.dirname(kolla_ansible.__file__))")
cp "$KOLLA_PATH/ansible/inventory/all-in-one" inventory/hosts
# or for multinode:
cp "$KOLLA_PATH/ansible/inventory/multinode" inventory/hosts
```

Then edit the host assignments at the top — don't modify the group structure below.

## Post-Deploy Fixes

### 6. Skyline `Unsupported database backend`

Skyline only accepts `mysql://` as URL scheme, but kolla generates `mysql+pymysql://`.

```bash
sed -i "s|mysql+pymysql://|mysql://|" /etc/kolla/skyline-apiserver/skyline.yaml
podman restart skyline_apiserver
# or: docker restart skyline_apiserver
```

### 7. Horizon log permissions

Horizon may fail with `Permission denied` on log file.

```bash
chown -R 42420:42420 /var/log/kolla/horizon
podman restart horizon
# or: docker restart horizon
```

### 8. MariaDB healthcheck shows `unhealthy`

Healthcheck may report unhealthy even when the database works. Services function fine. This is cosmetic — ignore it unless queries are actually failing.

## Before Any Deployment

**Always read the release notes** for the target release before starting:
https://docs.openstack.org/releasenotes/kolla-ansible/

Release notes contain breaking changes, new defaults, deprecated options, and known issues that can save hours of debugging.
