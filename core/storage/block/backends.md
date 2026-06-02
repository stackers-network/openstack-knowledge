# Cinder Backend Drivers

## LVM (Reference Backend)

LVM (Logical Volume Manager) is Cinder's reference implementation. It uses a local LVM volume group to provide block storage via iSCSI (or iSER, or FC). It is the simplest backend to set up and is used for development, testing, and small deployments, but does not support active-active cinder-volume or live migration between hosts.

### How It Works

- Each Cinder volume becomes an LVM logical volume inside the configured volume group.
- Snapshots use LVM's built-in snapshot mechanism (copy-on-write thin LV or old-style thick snapshot).
- The volume is exported to compute nodes via an iSCSI target using one of: `lioadm` (LIO, recommended), `tgtadm`, or `nvmet`.
- os-brick on the compute node uses `iscsiadm` to discover and log in to the target.

### Setup

```bash
# 1. Create a physical volume and volume group on the storage node
pvcreate /dev/sdb
vgcreate cinder-volumes /dev/sdb

# 2. Install required packages (RHEL/CentOS)
dnf install -y targetcli python3-rtslib lvm2

# 3. Install required packages (Debian/Ubuntu)
apt-get install -y tgt lvm2

# 4. Enable and start services
systemctl enable --now target   # LIO target daemon
```

### Configuration

```ini
[DEFAULT]
enabled_backends = lvm

[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_backend_name = lvm
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = lioadm
volumes_dir = /var/lib/cinder/volumes

# Optional: use thin provisioning (requires thin-provisioning-tools)
lvm_type = thin
lvm_mirrors = 0

# Optional: iSCSI portal IP (defaults to my_ip from [DEFAULT])
target_ip_address = 10.0.0.30
target_port = 3260
target_prefix = iqn.2010-10.org.openstack

# Optional: CHAP authentication for iSCSI
use_chap_auth = true

# Optional: multiple iSCSI portals for multipath
iscsi_secondary_ip_addresses = 10.0.1.30
```

### Volume Type

```bash
openstack volume type create lvm-standard \
  --property volume_backend_name=lvm
```

### Caveats

- LVM snapshots have a fixed COW overhead; over-filling the snapshot LV causes the snapshot to be invalidated. Use thin LVs (`lvm_type = thin`) to avoid this.
- Not suitable for active-active HA (the volume group is local to one host). Use Pacemaker/Corosync for active-passive HA.
- iSCSI target must be reachable from all compute nodes (storage VLAN required).
- `lioadm` (LIO) is preferred over `tgtadm`; `tgtadm` is deprecated.
- Thin provisioning requires `thin-provisioning-tools` package and the `dm-thin-pool` kernel module.

---

## Ceph RBD

Ceph RADOS Block Device (RBD) is the most widely used Cinder backend in production OpenStack deployments. It provides native thin provisioning, snapshots, cloning, and replication (via Ceph itself). It is well-suited for active-active cinder-volume because the Ceph cluster is inherently multi-access.

### How It Works

- Each Cinder volume becomes an RBD image in the configured Ceph pool.
- Compute nodes access volumes directly via the `librbd` library (no iSCSI; the kernel RBD driver or QEMU native RBD driver is used).
- Snapshots are created as RBD snapshots; clones are created as RBD clone images (copy-on-write, zero-copy until modified).
- Image-to-volume copies can be accelerated by placing Glance images in a Ceph pool and cloning.

### Setup

```bash
# 1. Create Ceph pools (on Ceph admin node)
ceph osd pool create volumes 128
ceph osd pool create volumes-backup 64
rbd pool init volumes
rbd pool init volumes-backup

# 2. Create Ceph users
ceph auth get-or-create client.cinder \
  mon 'profile rbd' \
  osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd-read-only pool=images' \
  mgr 'profile rbd pool=volumes' \
  > /etc/ceph/ceph.client.cinder.keyring

ceph auth get-or-create client.cinder-backup \
  mon 'profile rbd' \
  osd 'profile rbd pool=volumes-backup, profile rbd pool=volumes' \
  mgr 'profile rbd pool=volumes-backup' \
  > /etc/ceph/ceph.client.cinder-backup.keyring

# 3. Distribute ceph.conf and keyrings to storage node and all compute nodes
# On storage node:
scp /etc/ceph/ceph.conf ceph.client.cinder.keyring storage01:/etc/ceph/
# On compute nodes (for RBD attach without iSCSI):
scp /etc/ceph/ceph.conf ceph.client.cinder.keyring compute01:/etc/ceph/

# 4. Create libvirt secret for Nova on each compute node
# First get the key:
ceph auth get-key client.cinder > /tmp/client.cinder.key
# Then on each compute node:
cat > /tmp/secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>457eb676-33da-42ec-9a8c-9293d545c337</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF
virsh secret-define --file /tmp/secret.xml
virsh secret-set-value --secret 457eb676-33da-42ec-9a8c-9293d545c337 \
  --base64 "$(cat /tmp/client.cinder.key)"
```

### Configuration

```ini
[DEFAULT]
enabled_backends = ceph

[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = 5
rados_connection_retries = 3
rados_connection_interval = 5
rbd_user = cinder
rbd_secret_uuid = 457eb676-33da-42ec-9a8c-9293d545c337

# Enable image cloning from Glance (requires images in same Ceph cluster)
rbd_image_compression_algorithm = none

# Thin provisioning — always enabled for RBD
# Report stats correctly:
report_dynamic_total_capacity = true

# For Ceph replication (uses Ceph mirroring, not Cinder replication v2.1)
# replication_device = backend_id:ceph-dr,conf:/etc/ceph/ceph-dr.conf,...
```

### Volume Type

```bash
openstack volume type create ceph-standard \
  --property volume_backend_name=ceph

openstack volume type create ceph-ssd \
  --property volume_backend_name=ceph \
  --property volume_type=ssd
```

### Glance + Ceph Integration (Fast Boot)

When Glance also uses the same Ceph cluster as its image store, Cinder can clone images into volumes without copying data:

In `glance-api.conf`:
```ini
[glance_store]
stores = rbd
default_store = rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
```

Cinder automatically detects when the image is stored in Ceph and performs a native RBD clone instead of downloading the image. This reduces boot time from minutes to seconds.

### Caveats

- The `rbd_secret_uuid` must be set up on every compute node as a libvirt secret (see Setup above). A mismatch causes attach failures.
- `rbd_max_clone_depth = 5` limits the depth of the clone chain. When exceeded, Cinder flattens the clone (copies all parent data), which can take significant time for large volumes. Tune based on your snapshot/clone usage.
- Ceph pool replication factor (typically 3x) means 1 GiB of Cinder volume consumes ~3 GiB of raw Ceph storage.
- For high-performance workloads, use a separate Ceph pool backed by NVMe OSDs and create a volume type pointing to that pool.
- The `rados_connect_timeout` should be set; default is -1 (infinite) which can cause cinder-volume to hang indefinitely if Ceph monitors are unreachable.

---

## NFS

The NFS driver stores each Cinder volume as a file (sparse or full) on an NFS share. It is simpler than iSCSI for shared-access storage and works well with NAS appliances.

### How It Works

- cinder-volume mounts each configured NFS share.
- Volumes are stored as raw image files (QEMU `qcow2` or `raw` format) within the mount.
- Compute nodes re-mount the same NFS share and attach the file as a disk to the instance (via QEMU's file-backed disk).
- Snapshots use QEMU's qcow2 snapshot mechanism (backing files).

### Setup

```bash
# 1. On the NFS server, create an export
echo "/exports/cinder 10.0.0.0/24(rw,sync,no_root_squash,no_subtree_check)" >> /etc/exports
exportfs -ra

# 2. On the storage node, create the shares file
mkdir -p /etc/cinder
echo "10.0.0.50:/exports/cinder" > /etc/cinder/nfs_shares

# 3. Set permissions
chmod 640 /etc/cinder/nfs_shares
chown root:cinder /etc/cinder/nfs_shares
```

### Configuration

```ini
[DEFAULT]
enabled_backends = nfs

[nfs]
volume_driver = cinder.volume.drivers.nfs.NfsDriver
volume_backend_name = nfs
nfs_shares_config = /etc/cinder/nfs_shares
nfs_mount_point_base = /var/lib/cinder/mnt
nfs_snapshot_support = true

# Use qcow2 for thin provisioning and snapshot support
nfs_sparsed_volumes = true
nas_secure_file_operations = false
nas_secure_file_permissions = false

# Optional: NFS mount options
nfs_mount_options = vers=4.1,nconnect=4,sec=sys

# Optional: minimum free space threshold (% of share)
nfs_used_ratio = 0.95
nfs_oversub_ratio = 1.0
```

### Volume Type

```bash
openstack volume type create nfs-standard \
  --property volume_backend_name=nfs
```

### Caveats

- NFS volumes do not support multi-attach.
- `nas_secure_file_operations = false` and `nas_secure_file_permissions = false` may be required for compatibility with some NFS servers but reduce security; set to `auto` or `true` when the NFS server supports it.
- Snapshot chains can grow deep with qcow2 backing files; consider periodic merge operations.
- NFS v4.1 with `nconnect` (multiple TCP connections) significantly improves throughput on Linux 5.3+.
- Not suitable for latency-sensitive workloads; NFS adds protocol overhead vs. block storage.

---

## Pure Storage FlashArray

Pure Storage is a popular all-flash SAN vendor with native Cinder drivers for iSCSI, Fibre Channel, and NVMe-oF.

### How It Works

- cinder-volume communicates with the FlashArray REST API to create/delete/manage volumes.
- Volumes are exported to compute nodes via iSCSI, FC, or NVMe-oF, depending on the driver variant.
- Snapshots are created as FlashArray snapshots (space-efficient, nearly instant).
- Supports synchronous replication between FlashArray pairs (ActiveCluster) and async replication (Active DR).

### Installation

```bash
# Install the Pure Storage driver package
pip install purestorage
# Or via OS package manager:
dnf install python3-purestorage
```

### Configuration (iSCSI)

```ini
[DEFAULT]
enabled_backends = pure-iscsi

[pure-iscsi]
volume_driver = cinder.volume.drivers.pure.PureISCSIDriver
volume_backend_name = pure-iscsi
san_ip = 192.168.10.20
pure_api_token = T-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
use_chap_auth = true
pure_host_personality = esxi   # or 'windows', 'aix', 'esxi', 'oracle-vm-server', 'vms' (leave unset for Linux)

# Multipath (strongly recommended)
pure_iscsi_cidr = 192.168.10.0/24
pure_iscsi_cidr_list = 192.168.10.0/24,192.168.11.0/24

# Thin provisioning (always enabled on Pure)
max_over_subscription_ratio = 10.0

# Auto-delete volumes with last snapshot on array when Cinder deletes volume
pure_eradicate_on_delete = false
```

### Configuration (Fibre Channel)

```ini
[pure-fc]
volume_driver = cinder.volume.drivers.pure.PureFCDriver
volume_backend_name = pure-fc
san_ip = 192.168.10.20
pure_api_token = T-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
pure_eradicate_on_delete = false
```

### Configuration (NVMe-oF/RoCE)

```ini
[pure-nvme]
volume_driver = cinder.volume.drivers.pure.PureNVMEDriver
volume_backend_name = pure-nvme
san_ip = 192.168.10.20
pure_api_token = T-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
pure_nvme_transport = roce   # or 'tcp'
pure_nvme_cidr = 192.168.20.0/24
```

### Volume Type

```bash
openstack volume type create pure-nvme \
  --property volume_backend_name=pure-nvme \
  --property replication_enabled='<is> True'
```

### Replication (ActiveDR)

```ini
[pure-iscsi]
# ... (as above) ...
replication_device = backend_id:pure-dr-array,san_ip:192.168.10.21,api_token:T-yyyyyyyy-...,type:async
pure_replication_pg_name = cinder-replication-pg
pure_replication_pod_name = cinder::cinder-pod
```

### Caveats

- `pure_eradicate_on_delete = false` (default) means deleted volumes are soft-deleted on the array and moved to the Pure "destroyed" state. They consume no space but can be recovered. Set to `true` if you want immediate space reclamation.
- CHAP authentication (`use_chap_auth = true`) is strongly recommended for iSCSI to prevent unauthorized access.
- The API token should be for a dedicated `cinder` API user on the FlashArray with minimum required permissions, not the array admin.
- FlashArray requires at minimum Purity//FA 5.3.0 for the features supported by current Cinder drivers.

---

## NetApp ONTAP

NetApp ONTAP is a widely-deployed enterprise storage platform with comprehensive Cinder driver support (iSCSI, NFS, FC, and NVMe-oF). The driver communicates with ONTAP via its REST API (preferred in 22.x+) or ZAPI.

### Installation

```bash
pip install netapp-lib
# Or:
dnf install python3-netapp-lib
```

### Configuration (NFS with ONTAP)

```ini
[DEFAULT]
enabled_backends = netapp-nfs

[netapp-nfs]
volume_driver = cinder.volume.drivers.netapp.common.NetAppDriver
volume_backend_name = netapp-nfs
netapp_storage_family = ontap_cluster
netapp_storage_protocol = nfs
netapp_server_hostname = ontap-cluster.example.com
netapp_server_port = 443
netapp_login = admin
netapp_password = ontapadminpass
netapp_vserver = svm-cinder
nfs_shares_config = /etc/cinder/netapp_nfs_shares
netapp_nfs_mount_options = vers=4.1,nconnect=4

# Thin provisioning
netapp_thin_provisioned = true
max_over_subscription_ratio = 10.0
reserved_percentage = 5
```

### Configuration (iSCSI with ONTAP)

```ini
[netapp-iscsi]
volume_driver = cinder.volume.drivers.netapp.common.NetAppDriver
volume_backend_name = netapp-iscsi
netapp_storage_family = ontap_cluster
netapp_storage_protocol = iscsi
netapp_server_hostname = ontap-cluster.example.com
netapp_server_port = 443
netapp_login = admin
netapp_password = ontapadminpass
netapp_vserver = svm-cinder

# QoS (maps to ONTAP QoS policies)
netapp_qos_policy_group_is_adaptive = false

# Volume replication (SnapMirror)
netapp_replication_volume_snapmirror_timeouts = 30
```

### NFS Shares File (for ONTAP NFS)

```
# /etc/cinder/netapp_nfs_shares
ontap-svm-nfs.example.com:/vol/cinder_vol_1
ontap-svm-nfs.example.com:/vol/cinder_vol_2
```

### Volume Type

```bash
openstack volume type create netapp-nfs \
  --property volume_backend_name=netapp-nfs

# ONTAP QoS integration
openstack volume type create netapp-iscsi-gold \
  --property volume_backend_name=netapp-iscsi \
  --property netapp:qos_policy_group=gold-qos
```

### Caveats

- ONTAP REST API is available from ONTAP 9.6+; use `netapp_api_version = REST` (the default in recent drivers). ZAPI will be deprecated.
- The `netapp_vserver` parameter is required for cluster-mode ONTAP (cDOT). Do not use the cluster management LIF — create a dedicated data SVM.
- NFS exports must be pre-created on ONTAP and listed in the NFS shares file; Cinder does not auto-create FlexVols.
- For large deployments, use a dedicated `cinder` user on ONTAP with minimum RBAC permissions rather than the `admin` user.
- ONTAP FlexClone (used for volume cloning) requires the FlexClone license; without it, Cinder falls back to full data copy.

---

## Choosing a Backend

| Criteria | LVM | Ceph RBD | NFS | Pure Storage | NetApp ONTAP |
|---|---|---|---|---|---|
| Production suitability | Dev/small | Yes | Medium | Yes | Yes |
| Active-active HA | No | Yes | With shared storage | Yes | Yes |
| Thin provisioning | Optional | Always | Always (qcow2) | Always | Optional |
| Native snapshots | LVM COW | RBD snapshots | qcow2 backing | FA snapshots | ONTAP snapshots |
| Cloning speed | Slow (copy) | Fast (COW) | Slow (copy) | Fast | Fast (FlexClone) |
| Multi-attach | No | Yes (with driver cap) | No | Yes | Yes |
| Replication | No | Ceph mirroring | No | ActiveDR/ActiveCluster | SnapMirror |
| Typical use case | CI/CD, lab | Private cloud at scale | NAS-backed workloads | High-perf enterprise | Enterprise/mixed |

---

## Common Driver Configuration Options

These options apply across most drivers and are set in the backend stanza:

| Option | Default | Description |
|---|---|---|
| `volume_backend_name` | (stanza name) | Name reported to scheduler; must match extra spec |
| `volume_driver` | — | Full Python class path of the driver |
| `max_over_subscription_ratio` | `20.0` | Thin provisioning overcommit ratio |
| `reserved_percentage` | `0` | Percentage of capacity reserved (not reported as free) |
| `filter_function` | `None` | Python expression for DriverFilter |
| `goodness_function` | `None` | Python expression for GoodnessWeigher |
| `use_multipath_for_image_xfer` | `false` | Use multipath when copying images to/from volumes |
| `enforce_multipath_for_image_xfer` | `false` | Fail if multipath not available during image transfer |
| `num_volume_device_scan_tries` | `3` | Retries when scanning for attached device |
| `volume_copy_bps_limit` | `0` | Bandwidth cap for volume copy operations (bytes/sec; 0 = unlimited) |
| `volume_copy_timeout` | `0` | Timeout for volume copy via dd (seconds; 0 = unlimited) |
