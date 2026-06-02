# Glance Operations

## Upload a Local Image

The most direct method: read image data from a local file and push it directly to Glance.

```bash
openstack image create \
  --disk-format qcow2 \
  --container-format bare \
  --file ubuntu-24.04.qcow2 \
  --public \
  ubuntu-24.04
```

With additional properties set at creation time:

```bash
openstack image create \
  --disk-format qcow2 \
  --container-format bare \
  --file ubuntu-24.04.qcow2 \
  --public \
  --property hw_disk_bus=virtio \
  --property hw_vif_model=virtio \
  --property os_distro=ubuntu \
  --property os_version=24.04 \
  --min-disk 2 \
  --min-ram 512 \
  ubuntu-24.04
```

Common `image create` flags:

| Flag | Description |
|---|---|
| `--disk-format` | Required. qcow2, raw, vmdk, vhd, vhdx, iso, etc. |
| `--container-format` | Required. Usually `bare` |
| `--file` | Path to local file to upload |
| `--public` | Set visibility to public (admin only) |
| `--private` | Set visibility to private (default) |
| `--community` | Set visibility to community |
| `--shared` | Set visibility to shared |
| `--protected` | Mark image as protected (cannot be deleted) |
| `--unprotected` | Mark image as unprotected |
| `--property key=value` | Set an image property |
| `--tag` | Add a tag |
| `--min-disk` | Minimum disk size in GB required to boot |
| `--min-ram` | Minimum RAM in MB required to boot |
| `--project` | Create in a specific project (admin only) |
| `--project-domain` | Domain of the project |

---

## Import an Image with Interoperable Import

The interoperable import workflow is the recommended approach for images that are not available as local files — for example, images hosted on external URLs or staged in Glance's web-upload cache.

### Import from a URL (web-download)

```bash
openstack image create \
  --disk-format qcow2 \
  --container-format bare \
  --uri https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img \
  --import \
  ubuntu-24.04
```

Glance fetches the image data from the URL server-side. The client does not stream the data — Glance's import worker downloads it directly to the store. This avoids saturating the client's uplink for large images.

### Import via glance-direct (stage then import)

Stage the image data to Glance's staging area first, then trigger the import:

```bash
# Step 1: Create the image record (no data yet; status = queued)
openstack image create \
  --disk-format qcow2 \
  --container-format bare \
  ubuntu-24.04-staged

# Step 2: Stage the data (PUT to /v2/images/<id>/stage)
glance image-stage --file ubuntu-24.04.qcow2 <image-id>
# or via curl:
curl -X PUT "http://glance:9292/v2/images/<image-id>/stage" \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @ubuntu-24.04.qcow2

# Step 3: Trigger the import
glance image-import --import-method glance-direct <image-id>
# or via openstack CLI (if import subcommand available):
openstack image create --import <image-id>
```

### Copy an Image Between Stores (copy-image)

When multiple stores are configured, copy an existing active image to an additional store:

```bash
glance image-import --import-method copy-image \
  --stores rbd,swift \
  <image-id>
```

This is used in multi-store deployments to replicate an image to edge stores after initial upload to a central store.

---

## List Images

```bash
# List images visible to the current project
openstack image list

# Include community images
openstack image list --community

# Include shared images
openstack image list --shared

# List public images only
openstack image list --public

# List private images only
openstack image list --private

# Filter by status
openstack image list --status active

# Filter by format
openstack image list --disk-format qcow2

# Long format (includes more columns)
openstack image list --long

# All images across all projects (admin)
openstack image list --all-projects

# Filter by property
openstack image list --property os_distro=ubuntu

# Filter by tag
openstack image list --tag production
```

---

## Show Image Details

```bash
openstack image show ubuntu-24.04
openstack image show <image-id>
```

Output includes: id, name, disk_format, container_format, size, status, visibility, protected, min_disk, min_ram, owner, created_at, updated_at, tags, and all custom properties.

---

## Delete Images

```bash
# Delete by name or ID
openstack image delete ubuntu-24.04
openstack image delete <image-id>

# Delete multiple images
openstack image delete img1 img2 img3

# Delete is blocked if protected=true; unprotect first:
openstack image set --unprotected ubuntu-24.04
openstack image delete ubuntu-24.04
```

---

## Update Image Metadata

```bash
# Rename an image
openstack image set --name ubuntu-24.04-LTS ubuntu-24.04

# Change visibility (admin required for public)
openstack image set --public ubuntu-24.04
openstack image set --private ubuntu-24.04
openstack image set --community ubuntu-24.04

# Set protection
openstack image set --protected ubuntu-24.04
openstack image set --unprotected ubuntu-24.04

# Add or update properties
openstack image set --property hw_disk_bus=virtio ubuntu-24.04
openstack image set --property hw_vif_model=virtio --property os_distro=ubuntu ubuntu-24.04

# Remove a property
openstack image unset --property hw_disk_bus ubuntu-24.04

# Add tags
openstack image set --tag production --tag tested ubuntu-24.04

# Remove tags
openstack image unset --tag tested ubuntu-24.04

# Deactivate (blocks non-admin downloads and Nova use)
openstack image set --deactivate ubuntu-24.04

# Reactivate
openstack image set --reactivate ubuntu-24.04
```

---

## Save (Download) an Image

```bash
# Download to a local file
openstack image save --file downloaded.qcow2 ubuntu-24.04

# Download by ID
openstack image save --file /tmp/image.qcow2 <image-id>
```

Deactivated images cannot be downloaded by non-admin users. Admins can still download them.

---

## Manage Image Sharing

Images with `visibility = shared` can be explicitly shared with other projects using the image member API.

```bash
# Share an image with a target project
openstack image add project ubuntu-24.04 <target-project-id>

# The target project must accept the shared image
# (run as a user in the target project)
openstack image set --status accepted ubuntu-24.04

# Or reject it
openstack image set --status rejected ubuntu-24.04

# List members of a shared image (owner view)
openstack image member list ubuntu-24.04

# Remove a project from the member list
openstack image remove project ubuntu-24.04 <target-project-id>
```

Member status values:

| Status | Meaning |
|---|---|
| `pending` | Owner shared the image; member has not yet accepted |
| `accepted` | Member accepted the share; image appears in their `image list` |
| `rejected` | Member rejected the share; image does not appear in their `image list` |

---

## Manage Image Properties

### Set Properties at Create or Update Time

```bash
openstack image set --property hw_disk_bus=virtio ubuntu-24.04
openstack image set --property hw_vif_model=virtio ubuntu-24.04
openstack image set --property hw_firmware_type=uefi ubuntu-24.04
openstack image set --property hw_machine_type=q35 ubuntu-24.04
openstack image set --property os_type=linux ubuntu-24.04
openstack image set --property os_distro=ubuntu ubuntu-24.04
openstack image set --property os_version=24.04 ubuntu-24.04
openstack image set --property os_admin_user=ubuntu ubuntu-24.04
openstack image set --property os_require_quiesce=yes ubuntu-24.04
```

### Remove Properties

```bash
openstack image unset --property hw_disk_bus ubuntu-24.04
```

### Show All Properties

```bash
openstack image show ubuntu-24.04
# Properties are shown in the 'properties' column
```

---

## Use the Task API

The task API (`/v2/tasks`) is used internally by the import workflow. Operators can inspect tasks to debug stuck imports.

```bash
# List tasks (requires glance CLI)
glance task-list

# Show a specific task
glance task-show <task-id>
```

Task types:

| Type | Description |
|---|---|
| `import` | Created for each `image-import` invocation |

Task statuses: `pending`, `processing`, `success`, `failure`

---

## Run Database Maintenance

```bash
# Sync the database schema
glance-manage db_sync

# Load metadef namespace definitions from bundled JSON files
glance-manage db_load_metadefs

# Unload all metadef definitions
glance-manage db_unload_metadefs

# Export metadef definitions to JSON
glance-manage db_export_metadefs

# Purge deleted image records (logical deletes older than N days)
glance-manage db_purge --age_in_days 30

# Show current DB version
glance-manage db_version
```

---

## Configuration Reference

Glance configuration lives in `/etc/glance/glance-api.conf`. The file uses oslo.config INI format.

### [DEFAULT]

```ini
[DEFAULT]
# Host the API binds to
bind_host = 0.0.0.0
bind_port = 9292

# Worker processes (0 = auto-detect CPU count)
workers = 4

# Logging
log_file = /var/log/glance/api.log
log_dir = /var/log/glance

# Enable delayed delete (data kept until scrubber runs)
delayed_delete = false
scrub_time = 43200           # seconds before scrubber deletes data
scrubber_datadir = /var/lib/glance/scrubber

# Show image count in list responses
send_identity_headers = true

# Maximum image size in bytes (0 = unlimited)
image_size_cap = 1099511627776    # 1 TB

# Enable image import
enable_v2_api = true
enable_image_import = true

# Staging area for interoperable import
node_staging_uri = file:///var/lib/glance/staging

# Use this to override the public endpoint reported in API responses
public_endpoint = https://glance.example.com
```

### [glance_store]

```ini
[glance_store]
# Comma-separated list of enabled stores
stores = file,http

# Default store for new images
default_store = file

# Multiple store configuration (multi-store feature)
enabled_backends = fast:file,slow:file

# --- File store ---
filesystem_store_datadir = /var/lib/glance/images/

# Multiple directories (weight-based selection)
filesystem_store_datadirs = /var/lib/glance/images/:200,/mnt/nfs/glance/:100

# --- Swift store ---
swift_store_auth_address = http://keystone:5000/v3
swift_store_auth_version = 3
swift_store_user = service:glance
swift_store_key = <password>
swift_store_container = glance
swift_store_create_container_on_put = true
swift_store_large_object_size = 5120
swift_store_large_object_chunk_size = 200
swift_store_multi_tenant = false

# --- RBD/Ceph store ---
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8

# --- S3 store ---
s3_store_host = https://s3.amazonaws.com
s3_store_access_key = <access-key>
s3_store_secret_key = <secret-key>
s3_store_bucket = glance-images
s3_store_create_bucket_on_put = true
s3_store_large_object_size = 100
s3_store_large_object_chunk_size = 10

# --- Cinder store ---
cinder_store_auth_address = http://keystone:5000/v3
cinder_store_user_name = glance
cinder_store_password = <password>
cinder_store_project_name = service
cinder_volume_type = image-tier
cinder_store_replication_enabled = false
```

### [database]

```ini
[database]
connection = mysql+pymysql://glance:<password>@db-host/glance
max_pool_size = 10
max_overflow = 20
pool_timeout = 10
```

### [keystone_authtoken]

```ini
[keystone_authtoken]
www_authenticate_uri = http://keystone:5000
auth_url = http://keystone:5000
memcached_servers = memcache1:11211,memcache2:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = <glance-service-password>
```

### [paste_deploy]

```ini
[paste_deploy]
flavor = keystone
```

### [image_format]

```ini
[image_format]
# Disk formats that Glance will accept on upload
disk_formats = ami,ari,aki,vhd,vhdx,vmdk,raw,qcow2,vdi,iso,ploop

# Container formats Glance will accept
container_formats = ami,ari,aki,bare,ovf,ova,docker
```

### [task]

```ini
[task]
# Number of seconds a task is retained after completion before being purged
task_time_to_live = 2592000     # 30 days

# Executor type for tasks: taskflow (default)
task_executor = taskflow

# Logging for tasks
work_dir = /var/lib/glance/work

# Number of taskflow workers
eventlet_executor_pool_size = 1000
```

### [taskflow_executor]

```ini
[taskflow_executor]
# Engine mode: serial (default) or parallel
engine_mode = serial

# Maximum number of concurrent workers in parallel mode
max_workers = 10
```

### [oslo_messaging_rabbit] (for task notifications)

```ini
[oslo_messaging_rabbit]
rabbit_host = rabbitmq
rabbit_port = 5672
rabbit_userid = openstack
rabbit_password = <password>
rabbit_virtual_host = /
```

---

## Inspect Glance from the REST API Directly

```bash
export TOKEN=$(openstack token issue -f value -c id)
export GLANCE=http://glance:9292

# List images
curl -s -H "X-Auth-Token: $TOKEN" $GLANCE/v2/images | python3 -m json.tool

# Show image
curl -s -H "X-Auth-Token: $TOKEN" $GLANCE/v2/images/<image-id> | python3 -m json.tool

# Upload image data (PUT)
curl -X PUT $GLANCE/v2/images/<image-id>/file \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @image.qcow2

# Download image data
curl -H "X-Auth-Token: $TOKEN" \
  $GLANCE/v2/images/<image-id>/file -o downloaded.qcow2

# Create an image record (POST)
curl -X POST $GLANCE/v2/images \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "ubuntu-24.04",
    "disk_format": "qcow2",
    "container_format": "bare",
    "visibility": "public"
  }'

# Update image properties (PATCH)
curl -X PATCH $GLANCE/v2/images/<image-id> \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/openstack-images-v2.1-json-patch" \
  -d '[
    {"op": "add", "path": "/hw_disk_bus", "value": "virtio"},
    {"op": "replace", "path": "/name", "value": "ubuntu-24.04-LTS"}
  ]'

# Trigger import (web-download)
curl -X POST $GLANCE/v2/images/<image-id>/import \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "method": {"name": "web-download", "uri": "https://example.com/image.qcow2"}
  }'

# List tasks
curl -s -H "X-Auth-Token: $TOKEN" $GLANCE/v2/tasks | python3 -m json.tool

# Show task
curl -s -H "X-Auth-Token: $TOKEN" $GLANCE/v2/tasks/<task-id> | python3 -m json.tool
```
