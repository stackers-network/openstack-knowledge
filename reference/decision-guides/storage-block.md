# Choose a Cinder Block Storage Backend

Cinder supports many storage backends via a driver model. The decision affects performance, HA, operational complexity, and cost. The most common choices are LVM/iSCSI (the reference driver), Ceph RBD, NFS, and vendor-specific drivers.

## Feature Comparison

| Feature | LVM/iSCSI | Ceph RBD | NFS | Vendor (Pure/NetApp/Dell EMC/HPE) |
|---|---|---|---|---|
| High availability | No (single cinder-volume node) | Yes (distributed, N+K redundancy) | Depends on NFS server HA | Yes (array HA built in) |
| Performance | Local disk speed | Distributed, scales with OSDs | Network + NFS server speed | Varies — often excellent, NVMe arrays very fast |
| Snapshots | LVM snapshots (slow, CoW) | Copy-on-write (near-instant) | Backend dependent | Native (near-instant) |
| Volume cloning | LVM full copy (slow) | Shallow clone (fast, CoW) | Full copy | Native (fast) |
| Replication | No | Built-in (mirroring) | Backend dependent | Yes (synchronous/asynchronous) |
| Live migration support | Limited (iSCSI shared access issues) | Full (shared RBD pool) | Yes (shared filesystem) | Depends on driver |
| Boot from volume | Yes | Yes (preferred with Nova RBD) | Yes | Yes |
| Thin provisioning | Yes (LVM thin pools) | Yes | Depends | Yes |
| Multi-attach | No (iSCSI single-initiator default) | Yes (with RBD multi-attach) | Yes | Depends |
| Encryption (LUKS) | Yes | Yes | Yes | Depends |
| Max tested volume size | ~16 TB (LVM practical) | Petabytes | NFS server limit | Array dependent |
| Complexity | Low | High (Ceph cluster required) | Low | Medium (vendor driver + array) |
| Cost | Low (commodity hardware) | Medium-High (dedicated Ceph cluster) | Low (if NFS already exists) | High (storage array + licenses) |
| Operational tooling | vgdisplay, lvdisplay, iscsiadm | ceph, rbd, ceph dashboard | Standard NFS tools | Vendor CLI/GUI |
| Best for | Dev/test, small single-node clouds | Production scale-out clouds | Simple shared storage | Enterprise with existing array |

## Recommendation by Use Case

### Development, Testing, All-in-One
**Use LVM/iSCSI.** Ships as the Cinder default, zero additional infrastructure. Use thin-provisioned LVM to avoid pre-allocating disk.

```bash
# Check if a thin pool exists
lvdisplay | grep "LV Pool"
# If not, create one:
lvcreate --thin -L 100G vg-name/thinpool-name
```

### Production, Scale-Out Cloud (primary recommendation)
**Use Ceph RBD.** Provides distributed redundancy, fast snapshots, fast clones, and native live-migration support. If you are already running Ceph for images (Glance) or compute ephemeral disks (Nova), adding Cinder RBD is low incremental cost.

Prerequisites:
- A running Ceph cluster with a dedicated pool for Cinder (e.g. `volumes`)
- A Ceph client keyring for Cinder: `ceph auth get-or-create client.cinder`
- A Ceph client keyring for Nova (to access Cinder volumes during attach): `ceph auth get-or-create client.nova`
- `librbd` and `ceph-common` installed on all compute nodes

```bash
# Verify Ceph pool exists
ceph osd lspools | grep volumes

# Check Cinder can reach the Ceph cluster
sudo -u cinder rbd --id cinder --keyring /etc/ceph/ceph.client.cinder.keyring ls volumes
```

### Simple Shared Storage (existing NFS)
**Use NFS.** If your organization already runs a robust NFS server (NetApp, EMC, or even Linux NFS with DRBD), the Cinder NFS driver requires minimal setup. Performance and HA depend entirely on the NFS server.

### Enterprise with Existing Storage Array
**Use the vendor driver.** Pure Storage, NetApp ONTAP, Dell EMC PowerStore/PowerMax, HPE Nimble, and others have certified Cinder drivers. These provide native array features (instant snapshots, deduplication, replication) with OpenStack management.

Check the [Cinder driver support matrix](https://docs.openstack.org/cinder/latest/reference/support-matrix.html) for your specific array and OpenStack release.

### Mixed: Dev/test and Production in Same Cloud
**Use multiple backends.** Cinder supports multiple backends simultaneously. Use LVM for a `standard` volume type and Ceph RBD for a `performance` volume type:

```ini
# /etc/cinder/cinder.conf
[DEFAULT]
enabled_backends = lvm-backend,ceph-backend
default_volume_type = standard

[lvm-backend]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
volume_backend_name = LVM

[ceph-backend]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
rbd_user = cinder
rbd_secret_uuid = <LIBVIRT_SECRET_UUID>
volume_backend_name = Ceph
```

Then create volume types:

```bash
openstack volume type create standard
openstack volume type set standard --property volume_backend_name=LVM

openstack volume type create performance
openstack volume type set performance --property volume_backend_name=Ceph
```

## iSCSI Configuration Reference

```ini
# /etc/cinder/cinder.conf — LVM backend
[lvm-backend]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
iscsi_protocol = iscsi
iscsi_helper = lioadm   # or tgtadm
volume_backend_name = LVM
```

```bash
# Verify iSCSI target service
systemctl status tgt   # tgtd
systemctl status rtslib-fb-targetctl  # LIO

# List targets
tgtadm --lld iscsi --mode target --op show  # tgtd
targetcli ls  # LIO

# Check active iSCSI sessions from compute nodes
iscsiadm -m session
```

## Ceph RBD Configuration Reference

```ini
# /etc/cinder/cinder.conf — Ceph backend
[ceph-backend]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
rbd_user = cinder
rbd_secret_uuid = <LIBVIRT_SECRET_UUID>
volume_backend_name = Ceph
```

```bash
# Create the Ceph pool
ceph osd pool create volumes 128
ceph osd pool application enable volumes rbd

# Create the Cinder keyring
ceph auth get-or-create client.cinder \
  mon 'allow r' \
  osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rx pool=images'

# Register the secret in libvirt (on each compute node)
cat > /tmp/secret.xml <<EOF
<secret ephemeral="no" private="no">
  <uuid><LIBVIRT_SECRET_UUID></uuid>
  <usage type="ceph">
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF
virsh secret-define --file /tmp/secret.xml
virsh secret-set-value --secret <LIBVIRT_SECRET_UUID> \
  --base64 $(ceph auth get-key client.cinder)
```

## Operational Checks

```bash
# List Cinder backends and their status
openstack volume service list

# Check volume counts per backend
openstack volume list --all-projects -f json | \
  jq 'group_by(."Volume Type") | map({type: .[0]."Volume Type", count: length})'

# Ceph pool usage
ceph df
rados df

# LVM usage
vgdisplay cinder-volumes
lvdisplay | grep -E "LV Name|LV Size|Pool"
```
