# Shared Filesystems — Manila

Manila is the OpenStack Shared Filesystem service. It provisions and manages shared filesystems (NFS, CIFS/SMB, CephFS, GlusterFS, and others) that multiple compute instances can mount simultaneously. Unlike Cinder, which provides block storage attached to a single instance, Manila exports filesystems accessible to many clients at once.

Manila has been a core OpenStack service since the Kilo release (2015). It is analogous in design to Nova+Cinder but oriented around shared network-accessible filesystems rather than block devices.

## When to Read This Skill

- Deploying Manila for the first time
- Creating and managing NFS or CIFS shares for instances
- Configuring share types, share networks, and access rules
- Understanding DHSS=true vs DHSS=false driver modes
- Setting up CephFS, NetApp, or generic (NFS-Ganesha/Samba) backends
- Troubleshooting "share stuck in creating" or access denied errors
- Configuring share replication across availability zones
- Migrating shares between backends or availability zones
- Using share groups and consistent group snapshots
- Understanding the manila-data service and how share migration works internally

## Sub-Files

| File | What It Covers |
|---|---|
| [architecture.md](architecture.md) | Manila services (api, scheduler, share, data), share types, share networks, share groups, DHSS modes, replication, migration |
| [operations.md](operations.md) | CLI and API: create/manage shares, types, networks, access rules, snapshots, groups, migration, replication; full config reference |
| [internals.md](internals.md) | Driver interface, share server lifecycle, manila-data copy engine, scheduler filters, DB models |

## Quick Reference

```bash
# Create a 10 GiB NFS share
openstack share create --share-type default --name my-share NFS 10

# List shares
openstack share list

# Show share details (including export locations)
openstack share show my-share

# Grant read-write access to a CIDR
openstack share access create my-share ip 10.0.0.0/24 --access-level rw

# List access rules
openstack share access list my-share

# Extend a share to 20 GiB
openstack share extend my-share 20

# Create a snapshot
openstack share snapshot create my-share --name my-snap

# Delete a share
openstack share delete my-share
```

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| Keystone | Authentication and service catalog | Every Manila API call is token-validated |
| Neutron | Tenant network connectivity for share servers | Required for DHSS=true drivers; used to allocate ports for share server NICs |
| Nova | Share server VMs (generic driver) | Only for the generic (reference) driver; not needed for hardware/CephFS backends |
| Cinder | Share server root volumes (generic driver) | Only for the generic driver |
| RabbitMQ (or oslo.messaging backend) | Async RPC between Manila services | Required; api→scheduler→share communication uses the message bus |
| MariaDB 10.6+ or PostgreSQL 14+ | Manila database | Stores all share, type, network, and access metadata |
| Memcached | oslo.cache backend | Recommended; reduces DB load for repeated lookups |
| NFS server / Samba / CephFS | Actual filesystem backend | Provided by the driver; not managed by Manila core |

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

Manila release notes: https://docs.openstack.org/releasenotes/manila/
