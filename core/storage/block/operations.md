# Cinder Operations

## Configuration Reference

### Minimal `/etc/cinder/cinder.conf`

```ini
[DEFAULT]
transport_url = rabbit://openstack:rabbitpass@controller01:5672/
auth_strategy = keystone
my_ip = 10.0.0.30
enabled_backends = lvm
default_volume_type = standard
glance_api_servers = http://controller01:9292
log_dir = /var/log/cinder

[database]
connection = mysql+pymysql://cinder:cinderdbpass@controller01/cinder?charset=utf8mb4
max_pool_size = 10
max_overflow = 20
pool_timeout = 30

[keystone_authtoken]
www_authenticate_uri = http://controller01:5000
auth_url = http://controller01:5000
memcached_servers = controller01:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = cinder
password = cinderpass
service_token_roles = service
service_token_roles_required = true

[oslo_messaging_rabbit]
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
rabbit_interval_max = 30
rabbit_ha_queues = true

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp

[coordination]
# Required for active-active cinder-volume
# backend_url = etcd3+http://etcd01:2379

[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = lioadm
volumes_dir = /var/lib/cinder/volumes

[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = 5
rbd_user = cinder
rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337

[keymgr]
# Required for volume encryption
backend = barbican

[barbican]
auth_endpoint = http://controller01:5000/v3
barbican_endpoint_type = internal
```

### Multi-Backend Config Pattern

```ini
[DEFAULT]
enabled_backends = ceph-ssd,ceph-hdd,lvm-backup

[ceph-ssd]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph-ssd
rbd_pool = volumes-ssd
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337

[ceph-hdd]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph-hdd
rbd_pool = volumes-hdd
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337

[lvm-backup]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_backend_name = lvm-backup
volume_group = cinder-backup-vg
target_protocol = iscsi
target_helper = lioadm
```

---

## Database Initialization

```bash
# Populate the Cinder database (run once after install)
cinder-manage db sync

# Verify DB version
cinder-manage db version

# Run online DB migrations (required between major releases)
cinder-manage db online_data_migrations
```

---

## Volume Operations

### Create Volumes

```bash
# Basic volume
openstack volume create --size 50 my-data-vol

# With a specific type
openstack volume create --size 50 --type ssd-replicated my-data-vol

# From an image
openstack volume create --size 50 --image ubuntu-24.04 --type ssd-replicated boot-vol

# From a snapshot
openstack volume create --size 50 --snapshot my-data-snap restored-vol

# From another volume (clone)
openstack volume create --size 50 --source my-data-vol cloned-vol

# With availability zone
openstack volume create --size 50 --availability-zone az-storage-1 my-data-vol

# Multi-attach volume
openstack volume create --size 10 --type multiattach-type shared-vol

# Bootable volume (mark existing volume bootable)
openstack volume set --bootable my-data-vol
```

### List and Show Volumes

```bash
# List volumes in current project
openstack volume list

# Long listing with extra details
openstack volume list --long

# All projects (admin)
openstack volume list --all-projects

# Filter by status
openstack volume list --status available
openstack volume list --status in-use
openstack volume list --status error

# Filter by name pattern
openstack volume list --name my-data*

# Show a specific volume
openstack volume show my-data-vol

# Show volume in JSON (useful for scripting)
openstack volume show my-data-vol -f json
```

### Modify Volumes

```bash
# Extend a volume (volume must be available or in-use with online-extend support)
openstack volume set --size 100 my-data-vol

# Set name and description
openstack volume set --name new-name --description "production data" my-data-vol

# Change bootable flag
openstack volume set --bootable my-data-vol
openstack volume set --non-bootable my-data-vol

# Set read-only
openstack volume set --read-only my-data-vol
openstack volume set --read-write my-data-vol

# Add/update metadata
openstack volume set --property environment=production --property team=platform my-data-vol

# Remove a metadata key
openstack volume unset --property environment my-data-vol

# Change volume type (retype — may trigger migration)
openstack volume retype --migration-policy on-demand my-data-vol ssd-replicated
```

### Delete Volumes

```bash
# Delete a volume (must be in 'available' state)
openstack volume delete my-data-vol

# Delete multiple volumes
openstack volume delete vol1 vol2 vol3

# Force-delete a volume stuck in error state (admin)
openstack volume delete --force my-data-vol

# Purge soft-deleted volumes from DB (admin)
cinder-manage db purge 30   # purge records older than 30 days
```

### Attach and Detach

```bash
# Attach to an instance
openstack server add volume my-instance my-data-vol
openstack server add volume my-instance my-data-vol --device /dev/vdb

# Detach from an instance
openstack server remove volume my-instance my-data-vol

# List volumes attached to an instance
openstack server show my-instance -c volumes_attached

# Force-detach (admin — use only when normal detach fails)
openstack volume set --state available my-data-vol   # reset state
# Then detach via Nova:
openstack server remove volume my-instance my-data-vol
```

---

## Snapshot Operations

```bash
# Create a snapshot (volume must be available or use --force for in-use)
openstack volume snapshot create --volume my-data-vol my-data-snap

# Force-snapshot an in-use volume (application-crash-consistent)
openstack volume snapshot create --volume my-data-vol --force my-data-snap-live

# List snapshots
openstack volume snapshot list
openstack volume snapshot list --volume my-data-vol

# Show snapshot
openstack volume snapshot show my-data-snap

# Set metadata on snapshot
openstack volume snapshot set --property backup-level=daily my-data-snap

# Rename a snapshot
openstack volume snapshot set --name new-snap-name my-data-snap

# Delete a snapshot
openstack volume snapshot delete my-data-snap

# Force-delete a snapshot stuck in error state
openstack volume snapshot delete --force my-data-snap
```

---

## Backup Operations

```bash
# Create a full backup
openstack volume backup create --name my-data-backup my-data-vol

# Create an incremental backup (requires a prior full backup)
openstack volume backup create --name my-data-incr-backup --incremental my-data-vol

# Create a backup from a snapshot
openstack volume backup create --name snap-backup --snapshot my-data-snap my-data-vol

# Create backup of in-use volume (application-crash-consistent)
openstack volume backup create --name live-backup --force my-data-vol

# List backups
openstack volume backup list
openstack volume backup list --volume-id <volume-id>

# Show backup details
openstack volume backup show my-data-backup

# Restore a backup to a new volume
openstack volume backup restore my-data-backup

# Restore a backup to a specific volume
openstack volume backup restore my-data-backup --volume my-data-vol

# Delete a backup
openstack volume backup delete my-data-backup

# Force-delete a backup stuck in error state
openstack volume backup delete --force my-data-backup

# Export backup metadata (for cross-cloud restore)
openstack volume backup export my-data-backup

# Import backup record
openstack volume backup import <driver-name> <backup-metadata-json>
```

Backup service config in `cinder.conf`:

```ini
[DEFAULT]
backup_driver = cinder.backup.drivers.swift.SwiftBackupDriver
backup_swift_url = http://controller01:8080/v1/AUTH_
backup_swift_auth = per_user
backup_swift_auth_version = 3
backup_swift_auth_insecure = false
backup_swift_container = cinder_backup
backup_swift_object_size = 52428800    # 50 MiB chunks
backup_swift_block_size = 32768
backup_swift_enable_progress_timer = true
backup_compression_algorithm = zlib
backup_max_operations = 15
```

For Ceph backup backend:

```ini
[DEFAULT]
backup_driver = cinder.backup.drivers.ceph.CephBackupDriver
backup_ceph_conf = /etc/ceph/ceph.conf
backup_ceph_user = cinder-backup
backup_ceph_chunk_size = 134217728    # 128 MiB
backup_ceph_pool = cinder-backup
backup_ceph_stripe_unit = 0
backup_ceph_stripe_count = 0
restore_discard_excess_bytes = true
```

---

## Volume Type Operations

```bash
# Create a volume type
openstack volume type create standard

# Create with extra specs
openstack volume type create ssd-replicated \
  --property volume_backend_name=ceph-ssd \
  --property replication_enabled='<is> True'

# Create a public type visible to all projects
openstack volume type create shared-hdd --public

# Create a private type (only accessible to specific projects)
openstack volume type create private-nvme --private

# Grant project access to a private type
openstack volume type set --project engineering private-nvme

# List types
openstack volume type list
openstack volume type list --long   # includes extra specs and QoS

# Show a type
openstack volume type show ssd-replicated

# Set/update extra specs on an existing type
openstack volume type set --property multiattach='<is> True' ssd-replicated
openstack volume type set --property thin_provisioning_support='<is> True' ssd-replicated

# Remove an extra spec
openstack volume type unset --property multiattach ssd-replicated

# Set the default type (alternatively set in cinder.conf)
openstack volume type set --default standard

# Delete a type (must not be in use)
openstack volume type delete standard
```

---

## QoS Spec Operations

```bash
# Create a QoS spec (front-end = enforced by hypervisor)
openstack volume qos create high-iops \
  --consumer front-end \
  --property total_iops_sec=5000 \
  --property read_iops_sec=3000 \
  --property write_iops_sec=3000 \
  --property total_bytes_sec=524288000

# Create back-end QoS spec (enforced by storage array)
openstack volume qos create backend-throttle \
  --consumer back-end \
  --property maxIOPS=10000 \
  --property maxBWS=1073741824

# Both consumer = both front-end and back-end
openstack volume qos create balanced-qos \
  --consumer both \
  --property total_iops_sec=8000 \
  --property total_bytes_sec=1073741824

# List QoS specs
openstack volume qos list

# Show a QoS spec
openstack volume qos show high-iops

# Update QoS spec properties
openstack volume qos set high-iops --property total_iops_sec=8000

# Remove a property from a QoS spec
openstack volume qos unset high-iops --property read_iops_sec

# Associate QoS spec with a volume type
openstack volume qos associate high-iops ssd-replicated

# Disassociate QoS spec from a volume type
openstack volume qos disassociate high-iops ssd-replicated

# Disassociate from all types
openstack volume qos disassociate --all high-iops

# Delete a QoS spec
openstack volume qos delete high-iops
```

---

## Volume Transfer Operations

Volume transfers allow moving ownership of a volume between projects without copying data.

```bash
# Create a transfer request (source project)
openstack volume transfer request create --name transfer-to-billing my-data-vol
# Returns: transfer ID and auth_key — share auth_key with recipient

# List pending transfer requests
openstack volume transfer request list

# Show a transfer request
openstack volume transfer request show <transfer-id>

# Accept a transfer (destination project)
openstack volume transfer request accept <transfer-id> --auth-key <auth-key>

# Delete a transfer request (cancel before acceptance)
openstack volume transfer request delete <transfer-id>
```

---

## Volume Manage and Unmanage

These operations allow Cinder to take over management of pre-existing backend volumes (e.g., volumes created directly on the storage array) or to release a volume from Cinder management without deleting the data.

```bash
# Unmanage a volume (remove from Cinder DB without deleting data)
openstack volume unmanage my-data-vol

# Manage an existing backend volume (bring under Cinder control)
# The --ref argument is driver-specific (e.g., LVM LV name, Ceph image name)
openstack volume manage \
  --name imported-vol \
  --description "existing volume from array" \
  --volume-metadata source=array1 \
  --ref source-name=cinder-volumes/existing-lv \
  storage01@lvm

# List manageable volumes on a backend (admin)
openstack volume summary --all-projects   # shows totals
# Use the Cinder API directly to list manageable volumes:
# GET /v3/{project_id}/manageable_volumes?host=storage01@lvm
```

---

## Volume Migration

Migration moves a volume's data to a different backend or pool. Two migration types:

- **Host-assisted migration**: cinder-volume reads data from source and writes to destination (slow, works across any backends)
- **Driver-assisted migration**: The storage driver performs the migration natively (fast, both backends must be the same driver and support it)

```bash
# Migrate a volume to a different backend (admin)
# Volume must be in 'available' state
openstack volume migrate my-data-vol --host storage02@ceph#ceph-ssd

# Force host-assisted migration even if driver-assisted is available
openstack volume migrate my-data-vol \
  --host storage02@ceph#ceph-ssd \
  --force-host-copy

# Check migration status
openstack volume show my-data-vol -c migration_status
# Possible values: migrating, completing, success, error

# For in-use volumes, use retype with on-demand migration policy
openstack volume retype --migration-policy on-demand my-data-vol new-type
```

---

## Backend and Capacity Operations

```bash
# List all backend pools and their capacity (admin)
openstack volume backend pool list --long

# Show scheduler stats for a specific backend
openstack volume backend pool list --format json

# List volume services
openstack volume service list

# Disable a service (drain before maintenance)
openstack volume service set --disable --disable-reason "hardware maintenance" \
  cinder-volume storage01@lvm

# Re-enable a service
openstack volume service set --enable cinder-volume storage01@lvm
```

---

## Quota Operations

```bash
# Show quotas for a project
openstack quota show --project engineering

# Set volume quotas
openstack quota set \
  --volumes 200 \
  --snapshots 400 \
  --gigabytes 10240 \
  --backups 100 \
  --backup-gigabytes 5120 \
  --per-volume-gigabytes 2048 \
  engineering

# Reset quotas to defaults
openstack quota delete --project engineering

# Show quota usage
openstack quota show --usage engineering
```

---

## Volume Reset State

Use these admin commands to recover volumes stuck in transitional states:

```bash
# Reset volume to available
openstack volume set --state available my-data-vol

# Reset volume to error (to mark it for investigation)
openstack volume set --state error my-data-vol

# Reset snapshot state
openstack volume snapshot set --state available my-data-snap

# Reset backup state
openstack volume backup set --state available my-data-backup

# Force-detach a volume (remove all attachment records)
# Use when instance is deleted but volume is still 'in-use'
openstack volume set --state available my-data-vol
# Then update Nova attachment if needed via:
# nova volume-detach <instance-id> <volume-id>
```

---

## Encryption Type Management

```bash
# Create an encryption type for a volume type
openstack volume encryption type create \
  --provider nova.volume.encryptors.luks.LuksEncryptor \
  --cipher aes-xts-plain64 \
  --key-size 256 \
  --control-location front-end \
  encrypted-ssd

# Show encryption settings for a volume type
openstack volume encryption type show encrypted-ssd

# Update an encryption type
openstack volume encryption type set \
  --key-size 512 \
  encrypted-ssd

# Delete an encryption type
openstack volume encryption type delete encrypted-ssd
```

---

## Availability Zone Operations

```bash
# List storage availability zones
openstack availability zone list --volume

# Create a volume in a specific AZ
openstack volume create --size 50 --availability-zone az2 my-az2-vol

# Move a volume to a different AZ via migration
openstack volume migrate --host storage02@ceph#ceph-ssd my-data-vol
# Note: AZ is tied to the backend host; migrating to a backend in az2 moves the volume's AZ
```

---

## Useful Diagnostic Commands

```bash
# Show all volumes in error state (admin)
openstack volume list --all-projects --status error

# Show volume counts by status (admin)
openstack volume list --all-projects -f json | \
  python3 -c "import json,sys; vols=json.load(sys.stdin); \
  [print(v['Status']) for v in vols]" | sort | uniq -c

# Check if a volume has pending operations
openstack volume show my-data-vol -c migration_status -c status -c attachments

# Tail Cinder logs (on controller/storage node)
journalctl -u cinder-api -f
journalctl -u cinder-volume -f
journalctl -u cinder-scheduler -f

# Check RabbitMQ queue depth (on RabbitMQ node)
rabbitmqctl list_queues name messages consumers | grep cinder

# DB query — find volumes stuck in creating (older than 1 hour)
# Run via cinder-manage or direct DB access:
cinder-manage db purge 0   # dry-run to see what would be purged
```
