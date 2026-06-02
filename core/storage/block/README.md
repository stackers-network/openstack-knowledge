# Block Storage — Cinder

Cinder is the OpenStack Block Storage service. It provisions, manages, and exposes persistent block devices to instances (and to bare-metal via Ironic). Volumes appear inside instances as raw block devices (`/dev/vdb`, `/dev/sdb`, etc.) and are independent of instance lifecycle — a volume survives server deletion and can be re-attached to any instance in the same availability zone.

Cinder has been a core OpenStack service since the Folsom release (2012). It originated as the block-storage portion of Nova ("nova-volume") before being extracted into its own project.

## When to Read This Skill

- Deploying Cinder for the first time or adding a new backend
- Creating volumes, snapshots, and backups via CLI or API
- Configuring volume types, QoS specs, and encryption
- Setting up multi-backend or replicated storage
- Understanding how Cinder integrates with Nova (attach/detach) and Glance (image-backed volumes)
- Diagnosing volume creation failures, attach errors, or quota exhaustion
- Migrating volumes between backends or availability zones
- Configuring Ceph RBD, LVM, NFS, or vendor drivers
- Implementing active-active HA for cinder-volume

## Sub-Files

| File | What It Covers |
|---|---|
| [architecture.md](architecture.md) | cinder-api, cinder-scheduler, cinder-volume, cinder-backup; volume types, QoS, encryption; Nova/Glance integration; active-active HA |
| [operations.md](operations.md) | CLI and API: volumes, snapshots, backups, types, QoS, transfers, migration, manage/unmanage; full config reference |
| [internals.md](internals.md) | Driver interface, multi-backend config, replication v2.1, scheduler filters and weighers, DB models, RPC flow |
| [backends.md](backends.md) | LVM, Ceph RBD, NFS, Pure Storage, NetApp — config sections, setup steps, caveats |

## Quick Reference

```bash
# Create a volume
openstack volume create --size 50 --type ssd-replicated my-data-vol

# List volumes
openstack volume list

# Attach a volume to an instance
openstack server add volume my-instance my-data-vol --device /dev/vdb

# Detach a volume
openstack server remove volume my-instance my-data-vol

# Create a snapshot
openstack volume snapshot create --volume my-data-vol my-data-snap

# Create a backup
openstack volume backup create --name my-data-backup my-data-vol

# Extend a volume
openstack volume set --size 100 my-data-vol

# Show volume details
openstack volume show my-data-vol

# List volume types
openstack volume type list --long

# List volume backends (admin)
openstack volume backend pool list --long
```

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| Keystone | Authentication and service catalog | Every Cinder API call validated against Keystone |
| Nova | Volume attach/detach, instance-boot-from-volume | Nova calls Cinder to reserve and export volumes |
| Glance | Upload volume as image; create volume from image | Optional but needed for image-backed volumes |
| Barbican | Volume encryption key management | Required when volume encryption is configured |
| RabbitMQ (or oslo.messaging backend) | Async RPC between Cinder services | Required; all inter-service calls go through message bus |
| MariaDB 10.6+ or PostgreSQL 14+ | Cinder database | Stores volume, snapshot, backup, type metadata |
| Memcached | API-layer cache | Strongly recommended; reduces DB load |
| Storage backend | Actual data storage | LVM, Ceph, NFS, iSCSI array, or vendor driver |

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

Cinder release notes: https://docs.openstack.org/releasenotes/cinder/
Cinder admin guide: https://docs.openstack.org/cinder/latest/admin/
Cinder API reference: https://docs.openstack.org/api-ref/block-storage/v3/
