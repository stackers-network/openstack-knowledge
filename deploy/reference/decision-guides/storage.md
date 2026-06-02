# Storage Decision Guide

## Comparison

| Feature | Ceph | LVM | NFS |
|---|---|---|---|
| **Recommended for** | Production | Dev/test, small deployments | Shared storage with existing NFS |
| **Shared storage** | Yes (distributed) | No (local only) | Yes (network) |
| **Live migration** | Yes | No (unless shared backend) | Yes |
| **Scalability** | Scales horizontally | Single-node only | Depends on NFS server |
| **Complexity** | High (separate Ceph cluster) | Low | Low-Medium |
| **Redundancy** | Built-in (3x replication or EC) | None (single disk) | Depends on NFS server |
| **Performance** | High (distributed I/O) | High (local disk) | Variable (network-dependent) |
| **Cinder volumes** | Yes (RBD) | Yes (LVM) | Yes (NFS) |
| **Nova ephemeral** | Yes (RBD) | Yes (local) | No |
| **Glance images** | Yes (RBD) | File-backed | File-backed |
| **Manila (shared FS)** | CephFS | No | NFS |
| **Minimum hosts** | 3 (for replication) | 1 | 1 (NFS server) |

## Recommendation by Use Case

### Production → **Ceph**

Ceph is the standard production storage backend. It provides:
- Shared storage across all compute nodes (enables live migration)
- Built-in replication and fault tolerance
- Unified backend for block (Cinder), object (RGW), and file (CephFS) storage
- Horizontal scaling by adding OSDs

In `globals.yml`:
```yaml
enable_ceph: "no"  # kolla deploys don't manage Ceph — use external Ceph
glance_backend_ceph: "yes"
cinder_backend_ceph: "yes"
nova_backend_ceph: "yes"
```

Note: Kolla-ansible does not deploy Ceph itself. Use cephadm, Rook, or a standalone Ceph deployment. Then configure kolla-ansible to use external Ceph by providing the ceph.conf and keyrings in the `config/` directory.

### Dev/test / Single-node → **LVM**

LVM is the simplest storage option. No external dependencies. Good for:
- All-in-one (AIO) deployments
- Development and testing
- Environments where live migration is not needed

In `globals.yml`:
```yaml
enable_cinder_backend_lvm: "yes"
cinder_volume_group: "cinder-volumes"
```

Requires a volume group named `cinder-volumes` on the cinder-volume host. Create it before deploying:
```bash
pvcreate /dev/sdX
vgcreate cinder-volumes /dev/sdX
```

### Existing NFS infrastructure → **NFS**

Use NFS when you have an existing NFS server or NAS appliance and want to avoid the complexity of Ceph. Supports live migration since storage is shared.

In `globals.yml`:
```yaml
enable_cinder_backend_nfs: "yes"
```

Provide NFS share details in `config/nfs_shares`:
```
nfs-server:/path/to/share
```

## Migration Paths

| From | To | Path |
|---|---|---|
| LVM → Ceph | Migrate Cinder volumes with `cinder migrate`. Requires Ceph cluster to be deployed and configured first. |
| NFS → Ceph | Same as LVM → Ceph. Migrate volumes individually. |
| LVM → NFS | Migrate Cinder volumes. Simpler than Ceph migration but still requires maintenance window. |

**Key rule:** Choose Ceph for production from the start if you can. Migrating storage backends later is possible but involves per-volume data migration and a maintenance window.
