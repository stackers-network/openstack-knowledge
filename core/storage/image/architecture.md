# Glance Architecture

## Understand the glance-api Process

Glance runs as a single service process: `glance-api`. It handles all inbound REST API requests — image CRUD, data upload and download, import orchestration, task management, and metadef operations.

The `glance-registry` service existed in older releases to proxy metadata operations. It was deprecated in Rocky (2018) and removed in Train (2019). Any documentation or deployment guide referencing `glance-registry` is obsolete.

`glance-api` is a Python WSGI application deployed behind Apache httpd (mod_wsgi or mod_proxy_uwsgi) or Nginx. In smaller deployments it can run directly under Eventlet's built-in HTTP server, but the WSGI-server approach is required for production TLS termination and process management.

### API Versions

Glance exposes only the **Image Service v2 API** (stable since Icehouse). The v1 API was removed in Queens. All CLI commands and integrations use v2.

Base URL pattern: `http://<glance-host>:9292/v2/`

### Internal Architecture

```
Client (openstack CLI / Nova / Horizon)
        │
        ▼
  [Keystone Middleware]  ← validates token, injects request context
        │
        ▼
  glance-api (WSGI)
   ├── Router (routes)
   ├── Image Controller
   ├── Tasks Controller
   ├── Metadef Controller
   └── Store Proxy
        │
        ├── Image DB (SQLAlchemy → MariaDB / PostgreSQL)
        └── glance_store (backend driver)
              ├── file
              ├── swift
              ├── rbd (Ceph)
              ├── s3
              ├── cinder
              └── http (read-only)
```

All image metadata (name, format, status, properties, visibility, tags, locations) lives in the Glance database. Image binary data lives in the store backend identified by its location URI.

---

## Understand Image Stores

Glance uses the **glance_store** library to abstract backend storage. Multiple stores can be configured simultaneously. Each image location record contains a URI that encodes which store holds the data and where.

### File Store

Stores images as flat files on a local filesystem directory.

```ini
[glance_store]
stores = file
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

Simple and fast for single-node deployments. Not suitable for multi-node API deployments without a shared filesystem (NFS, CephFS) mounted at the same path on all nodes.

### Swift Store

Stores images as objects in OpenStack Swift. Supports very large images (chunked upload via Dynamic Large Objects). Horizontally scalable and the traditional HA choice before Ceph.

```ini
[glance_store]
stores = file,swift
default_store = swift
swift_store_auth_address = http://keystone:5000/v3
swift_store_user = service:glance
swift_store_key = <service-password>
swift_store_container = glance
swift_store_create_container_on_put = true
swift_store_large_object_size = 5120          # MB; objects larger than this are segmented
swift_store_large_object_chunk_size = 200      # MB per segment
```

### Ceph/RBD Store

Stores images as Ceph RBD block device images. Recommended for production deployments that also use Ceph for Cinder block storage and Nova ephemeral disks, because Nova can clone images directly in Ceph without copying data across the network (copy-on-write clone from Glance RBD image to Nova instance disk).

```ini
[glance_store]
stores = rbd
default_store = rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8                       # MB
```

Requires the `python-rbd` (librbd) Python bindings and a Ceph cluster accessible from the Glance host. The `glance` Ceph user must have `rwx` caps on the images pool.

### S3 Store

Stores images in an S3-compatible object store (AWS S3, MinIO, Ceph RGW, Rook).

```ini
[glance_store]
stores = s3
default_store = s3
s3_store_host = https://s3.amazonaws.com
s3_store_access_key = <access-key>
s3_store_secret_key = <secret-key>
s3_store_bucket = glance-images
s3_store_create_bucket_on_put = true
s3_store_large_object_size = 100              # MB; multipart upload threshold
s3_store_large_object_chunk_size = 10         # MB per part
```

### Cinder Store

Stores images as Cinder volumes. Useful when the deployment has fast Cinder-backed storage (e.g., all-flash array) and wants to serve images directly from that tier.

```ini
[glance_store]
stores = cinder
default_store = cinder
cinder_store_auth_address = http://keystone:5000/v3
cinder_store_user_name = glance
cinder_store_password = <password>
cinder_store_project_name = service
cinder_volume_type = image-tier
```

The Glance service account must have the `admin` role in the `service` project so it can attach Cinder volumes to a staging host during upload and download.

### HTTP Store (Read-Only)

Allows Glance to register an external HTTP/HTTPS URL as an image location without copying the data in. Used for referencing images hosted externally without importing them. Read-only — no upload through this store.

```ini
[glance_store]
stores = file,http
```

Image location URI looks like: `http://external-host.example.com/images/ubuntu.qcow2`

---

## Understand Image Properties

Image properties are free-form key-value pairs stored in the Glance database alongside core image fields. They serve two purposes:

1. **Scheduler and hypervisor hints** — Nova reads specific well-known properties and passes them to the hypervisor or scheduler.
2. **Informational metadata** — operators and users annotate images with release version, OS type, source, etc.

### Well-Known Hardware Properties (read by Nova/libvirt)

| Property | Values | Effect |
|---|---|---|
| `hw_disk_bus` | `virtio`, `scsi`, `ide`, `sata`, `usb` | Sets the bus type for the root disk |
| `hw_scsi_model` | `virtio-scsi`, `lsilogic` | Sets the SCSI controller model |
| `hw_vif_model` | `virtio`, `e1000`, `rtl8139` | Sets the virtual NIC model |
| `hw_machine_type` | `pc`, `pc-i440fx-2.11`, `q35` | Sets the QEMU machine type |
| `hw_firmware_type` | `bios`, `uefi` | Selects BIOS or UEFI firmware |
| `hw_cpu_sockets` | integer | Number of CPU sockets exposed to the guest |
| `hw_cpu_cores` | integer | Number of cores per socket |
| `hw_cpu_threads` | integer | Number of threads per core |
| `hw_video_model` | `vga`, `virtio`, `cirrus`, `qxl`, `none` | Virtual display adapter model |
| `hw_watchdog_action` | `disabled`, `reset`, `poweroff`, `pause` | Watchdog device behavior |
| `hw_rng_model` | `virtio` | Attach a hardware RNG device |
| `hw_mem_page_size` | `small`, `large`, `any`, `2048`, `1048576` | Request huge pages for guest memory |
| `hw_numa_nodes` | integer | Number of NUMA nodes to expose |
| `hw_pmu` | `true`, `false` | Enable/disable performance monitoring unit |
| `os_type` | `linux`, `windows` | OS family (affects some hypervisor defaults) |
| `os_distro` | `ubuntu`, `centos`, `windows` etc. | OS distribution name |
| `os_version` | string | OS version string |
| `os_require_quiesce` | `yes` | Require filesystem quiesce on snapshot |

### Setting Properties

```bash
openstack image set --property hw_disk_bus=virtio --property hw_vif_model=virtio <image>
openstack image set --property os_distro=ubuntu --property os_version=24.04 <image>
```

Properties can also be set at image creation time:

```bash
openstack image create --property hw_disk_bus=virtio --property hw_vif_model=virtio ...
```

---

## Understand Image Visibility

Every image has a `visibility` field that controls which projects can see and use it.

| Visibility | Who Can See | Who Can Use | Notes |
|---|---|---|---|
| `public` | All projects | All projects | Only admins can set; shown in `openstack image list` for all users |
| `community` | All projects | All projects | Any user can set; not shown by default in `openstack image list` |
| `shared` | Owner + explicitly added members | Owner + accepted members | Default after sharing with `image add project` |
| `private` | Owner project only | Owner project only | Default for newly created images |

### Visibility Transitions

```
private  ──► shared      (image add project)
private  ──► community   (openstack image set --community)
private  ──► public      (admin only: openstack image set --public)
shared   ──► private     (remove all members, then set private)
public   ──► private     (admin: openstack image set --private)
```

---

## Understand Image Status Lifecycle

```
queued
  │
  ▼  (upload begins / import initiated)
saving
  │
  ▼  (upload completes successfully)
active ◄──────────────── (reactivate)
  │                               │
  ▼  (admin deactivate)           │
deactivated ────────────────────►─┘
  │
  ▼  (upload fails or store error)
killed
  │
  ▼  (delete)
deleted
  │
  ▼  (scrubber removes data)
(pending_delete)  ← only when delayed_delete = true
```

| Status | Meaning |
|---|---|
| `queued` | Image record created; no data uploaded yet |
| `saving` | Data is currently being uploaded to the store |
| `active` | Data fully uploaded and available; image is usable |
| `deactivated` | Admin has deactivated the image; download blocked for non-admins |
| `killed` | Upload failed or store error; image data is corrupt or missing |
| `deleted` | Logically deleted; metadata retained if `delayed_delete = true` |
| `pending_delete` | Awaiting scrubber to remove data from the store |

Useful transitions for operators:

```bash
# Deactivate an image (blocks Nova from using it)
openstack image set --deactivate <image-id>

# Reactivate a deactivated image
openstack image set --reactivate <image-id>
```

---

## Understand Metadef Namespaces

Metadef (metadata definitions) is a catalog of structured property definitions that describe what properties an image (or server, flavor, volume) may have. It provides a schema for property keys, their types, allowed values, and human-readable descriptions. Horizon uses metadef to render structured property editors in the UI.

Metadef resources:

- **Namespace** — top-level grouping (e.g., `OS::Glance::CommonImageProperties`)
- **Object** — a named group of properties within a namespace (e.g., `Image Properties`)
- **Property** — a typed field definition (name, type, enum values, description)
- **Tag** — a predefined tag string
- **Resource type association** — maps a namespace to a resource type (`OS::Nova::Server`, `OS::Glance::Image`, etc.)

```bash
# List metadef namespaces
openstack --os-image-api-version 2 metric namespace list   # via REST; no OSC metadef commands in standard release

# Glance ships default namespaces from:
/usr/share/glance/metadefs/
```

Namespaces are seeded into the database at install time:

```bash
glance-manage db_load_metadefs
```

---

## Understand Protected Images

Any image can be marked `protected = true`. Protected images cannot be deleted until the flag is cleared, even by admins.

```bash
# Protect an image
openstack image set --protected <image>

# Remove protection
openstack image set --unprotected <image>
```

This is typically used for golden base images that should not be accidentally deleted.

---

## Understand the Image Format Fields

Every image record has two format fields that are set at creation time and cannot be changed afterward:

### disk_format

Describes the format of the image data on disk.

| Value | Description |
|---|---|
| `qcow2` | QEMU Copy-On-Write v2; sparse, supports snapshots and compression |
| `raw` | Unformatted binary; fastest for Ceph RBD |
| `vmdk` | VMware Virtual Machine Disk |
| `vhd` | Microsoft Hyper-V Virtual Hard Disk v1 |
| `vhdx` | Microsoft Hyper-V Virtual Hard Disk v2 |
| `vdi` | VirtualBox Virtual Disk Image |
| `iso` | ISO 9660 optical disk image |
| `ploop` | Parallels disk image |
| `aki` | Amazon Kernel Image |
| `ari` | Amazon Ramdisk Image |
| `ami` | Amazon Machine Image |

### container_format

Describes the container (outer wrapper) around the disk image. For most VM images this is `bare` (no outer container).

| Value | Description |
|---|---|
| `bare` | No container; the image is just the disk data |
| `ovf` | Open Virtualization Format wrapper |
| `ova` | OVA archive (OVF + VMDK) |
| `aki` | Amazon Kernel format |
| `ari` | Amazon Ramdisk format |
| `ami` | Amazon Machine Image format |
| `docker` | Docker container filesystem tarball |

### Allowed Format Combinations

The `[image_format]` configuration section controls which formats Glance will accept:

```ini
[image_format]
disk_formats = ami,ari,aki,vhd,vhdx,vmdk,raw,qcow2,vdi,iso,ploop
container_formats = ami,ari,aki,bare,ovf,ova,docker
```
