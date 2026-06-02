# Image — Glance

Glance is the OpenStack Image service. It provides a registry and delivery mechanism for virtual machine disk images, kernel images, ramdisks, and ISO files. It does not store images itself by default — it delegates persistence to a pluggable store backend (local filesystem, Swift, Ceph/RBD, S3, Cinder, or HTTP). Nova, Ironic, and other services retrieve images from Glance when provisioning compute resources.

Glance has been a core OpenStack service since the Bexar release (2011). The registry service (`glance-registry`) was deprecated in the Rocky release (2018) and removed in Train (2019). All metadata is now handled directly by the `glance-api` process.

## When to Read This Skill

- Deploying Glance for the first time or adding a new store backend
- Uploading, importing, or managing VM images
- Configuring image visibility (public, private, shared, community)
- Understanding the interoperable image import workflow
- Using image properties to control Nova scheduling or hardware configuration
- Managing image metadata namespaces and property definitions
- Setting up image caching on compute or API nodes
- Troubleshooting stuck images (queued, saving, killed states)
- Performing image conversion or decompression at import time
- Managing image sharing between projects

## Sub-Files

| File | What It Covers |
|---|---|
| [architecture.md](architecture.md) | glance-api, store backends, image properties, visibility model, status lifecycle, metadef namespaces, protected images |
| [operations.md](operations.md) | CLI and API: image upload, import, list, show, delete, set, save, sharing, property management, full config reference |
| [internals.md](internals.md) | glance_store library, image caching, task API, import plugins, location strategy, delayed delete and scrubber, image conversion, image decompression |

## Quick Reference

```bash
# Upload a local image
openstack image create \
  --disk-format qcow2 \
  --container-format bare \
  --file ubuntu-24.04.qcow2 \
  --public \
  ubuntu-24.04

# List all images visible to your project
openstack image list

# Show image details
openstack image show ubuntu-24.04

# Import an image from a URL (web-download)
openstack image create \
  --disk-format qcow2 \
  --container-format bare \
  --uri https://cloud.example.com/ubuntu-24.04.qcow2 \
  --import \
  ubuntu-24.04

# Set image properties for hardware configuration
openstack image set --property hw_disk_bus=virtio --property hw_vif_model=virtio ubuntu-24.04

# Share an image with another project
openstack image add project ubuntu-24.04 <target-project-id>
openstack image member set --status accepted ubuntu-24.04 <target-project-id>

# Save (download) an image locally
openstack image save --file downloaded.qcow2 ubuntu-24.04

# Delete an image
openstack image delete ubuntu-24.04
```

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| Keystone | Authentication and service catalog | All Glance API calls validated against Keystone |
| MariaDB 10.6+ or PostgreSQL 14+ | Image metadata database | Image data is stored in the configured backend, not the DB |
| Store backend (file / Swift / Ceph / S3 / Cinder) | Image data persistence | File store is default; Ceph RBD recommended for production HA |
| Apache httpd or Nginx | WSGI front-end for glance-api | mod_wsgi or uwsgi; direct eventlet server supported but not recommended |
| oslo.messaging / RabbitMQ | Task notifications and inter-service events | Required for task API and import workflow notifications |
| python-glanceclient | Python bindings | Used internally by Nova and other services |
| python-openstackclient | Unified CLI | Wraps the Glance v2 REST API |

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

Glance release notes: https://docs.openstack.org/releasenotes/glance/
