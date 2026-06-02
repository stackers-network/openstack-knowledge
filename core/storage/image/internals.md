# Glance Internals

## Understand the glance_store Library

`glance_store` is a standalone Python library (`python-glanceclient`'s storage peer) that abstracts all backend I/O behind a common interface. It is maintained as a separate project (`openstack/glance_store`) and is imported by `glance-api` at runtime.

### Store Interface

Every store backend implements the `glance_store.driver.Store` base class. The interface has four core operations:

| Method | Description |
|---|---|
| `add(image_id, image_file, image_size, context)` | Write image data; returns `(location_uri, bytes_written, checksum)` |
| `get(location, offset, chunk_size, context)` | Read image data; returns an iterator of chunks |
| `delete(location, context)` | Remove image data from the backend |
| `get_size(location, context)` | Return the size of stored data without reading it |

The `location` object encapsulates the backend URI and any credentials needed to access it (e.g., Swift auth tokens, Ceph keyring path).

### Location URIs

Each image has one or more `locations` entries in the database. A location is a JSON object with a `url` field (the backend URI) and a `metadata` field (backend-specific extra data).

| Store | URI Format |
|---|---|
| File | `file:///var/lib/glance/images/<image-id>` |
| Swift | `swift+https://<auth-url>/v1/AUTH_<tenant>/<container>/<image-id>` |
| Ceph/RBD | `rbd://<cluster-fsid>/<pool>/<image-id>/snap` |
| S3 | `s3://<bucket>/<image-id>` |
| Cinder | `cinder://<volume-id>` |
| HTTP | `http://<host>/path/to/image.qcow2` |

### Multiple Locations

A single image can have multiple location entries (one per store in multi-store deployments). Glance tries locations in order. If the first location fails (e.g., store unreachable), it falls back to the next. This allows live migration of image data between stores: add a new location, verify it, then delete the old one.

```bash
# Show all location URIs for an image (admin, includes sensitive store URIs)
glance image-show --show-multiple-locations <image-id>
```

The `show_multiple_locations` and `show_image_direct_url` options in `[DEFAULT]` control whether location URIs are exposed in API responses. Exposing them allows Nova to use copy-on-write cloning directly from the backend (required for Ceph RBD copy-on-write). They are disabled by default for security.

```ini
[DEFAULT]
show_image_direct_url = true       # Expose primary location URI
show_multiple_locations = true     # Expose all location URIs
```

---

## Understand Image Caching

Glance's image cache stores copies of image data on a local disk path on the Glance API node. When a download request comes in for a cached image, Glance serves it from the local cache instead of reading from the backend store (e.g., Swift or Ceph). This reduces latency and store backend load for frequently requested images.

### Cache Configuration

```ini
[DEFAULT]
# Enable the cache middleware in the pipeline (paste.ini)
# flavor must be set to 'keystone+cachemanagement' or 'cachemanagement'
image_cache_dir = /var/lib/glance/image-cache/
image_cache_max_size = 10737418240     # bytes; 10 GB default
image_cache_stall_time = 86400        # seconds; discard stalled downloads
image_cache_invalid_entry_grace_period = 3600
```

The cache is activated by enabling the `cache` filter in the `glance-api-paste.ini` pipeline:

```ini
# glance-api-paste.ini
[pipeline:glance-api-keystone+cachemanagement]
pipeline = cors healthcheck http_proxy_to_wsgi versionnegotiation osprofiler authtoken context cache cachemanage rootapp
```

### Cache Directories

The image cache directory has this layout:

```
/var/lib/glance/image-cache/
├── <image-id>              ← fully cached image (complete and valid)
├── incomplete/             ← in-progress download to cache
├── invalid/                ← downloads that failed or were interrupted
└── queue/                  ← images queued for pre-caching
```

### Cache Management Commands

```bash
# Show cache status and list cached images
glance-cache-manage list-cached

# Show images that are queued for caching
glance-cache-manage list-queued

# Queue an image for caching (downloads it to cache proactively)
glance-cache-manage queue-image <image-id>

# Remove a specific image from cache
glance-cache-manage delete-cached-image <image-id>

# Delete all cached images
glance-cache-manage delete-all-cached-images

# Remove a specific image from the queue
glance-cache-manage delete-queued-image <image-id>

# Remove all queued images
glance-cache-manage delete-all-queued-images

# Prune the cache — removes invalid and incomplete entries
glance-cache-pruner

# Clean the cache — removes images that have been deleted from Glance
glance-cache-cleaner
```

### Cache Pruner and Cleaner

Two separate background processes maintain the cache:

- **`glance-cache-pruner`** — enforces the `image_cache_max_size` limit by evicting least-recently-used images when the cache is full. Run periodically via cron:

  ```bash
  # /etc/cron.d/glance-cache
  */30 * * * * glance glance-cache-pruner
  0    * * * * glance glance-cache-cleaner
  ```

- **`glance-cache-cleaner`** — removes cache entries for images that have been deleted from the Glance database. Prevents stale data from accumulating.

---

## Understand the Task API

The Task API (`/v2/tasks`) provides an asynchronous job model for long-running image operations. Tasks are created internally by the import workflow and externally by operators who call the older `POST /v2/tasks` endpoint.

### Task Lifecycle

```
pending ──► processing ──► success
                      └──► failure
```

Tasks are retained in the database for `task_time_to_live` seconds (configurable; default 30 days) after they reach a terminal state, then purged.

### Task Executor — TaskFlow

Glance uses the **TaskFlow** library (also an OpenStack project) to build and execute import pipelines as directed acyclic graphs (DAGs) of small `Task` objects. Each `Task` has `execute()` and `revert()` methods. If any task in the pipeline fails, TaskFlow calls `revert()` in reverse order to clean up partial work.

The `[task]` and `[taskflow_executor]` config sections control the executor. In `serial` mode, all tasks run in one thread in sequence. In `parallel` mode, independent branches of the DAG run concurrently.

---

## Understand the Interoperable Import Workflow

Interoperable import is a multi-step protocol that separates image record creation, data staging, and data ingestion. It replaced the legacy single-step `PUT /v2/images/<id>/file` upload for environments that need pre-import processing.

### Import Methods

| Method | Description |
|---|---|
| `glance-direct` | Client stages data to Glance's staging URI, then calls `/import` |
| `web-download` | Client provides a URI; Glance's import worker fetches it server-side |
| `copy-image` | Copies an already-active image to additional store backends |

### Import Pipeline (TaskFlow DAG)

The import pipeline is a sequence of plugins. The built-in plugins are:

1. **`_ImportToStaging`** — fetches or moves data to the staging area
2. **`_InjectMetadataProperties`** — injects operator-defined properties (from `[image_import_opts]`)
3. **`_DecompressImage`** — decompresses the image if it is gzip/bz2/xz compressed (optional plugin)
4. **`_ConvertImage`** — converts the image to the target format (optional plugin, e.g., qcow2 → raw)
5. **`_ValidateImage`** — verifies format and basic integrity using `qemu-img`
6. **`_ImportToStore`** — moves data from staging to the configured store backend
7. **`_SaveImage`** — updates the image record status to `active`
8. **`_NotifyImportTask`** — sends a notification event on the message bus

### Configuring Import Plugins

```ini
[image_import_opts]
# Ordered list of plugins to run during import
image_import_plugins = [
  glance.async_.flows._plugins.inject_image_metadata,
  glance.async_.flows._plugins.image_decompression,
  glance.async_.flows._plugins.image_conversion,
  glance.async_.flows._plugins.ovf_process
]
```

### Staging Area

The staging area is a transient local directory (or URI) where image data is held between the stage and import phases.

```ini
[DEFAULT]
node_staging_uri = file:///var/lib/glance/staging
```

In a multi-node glance-api deployment, all API nodes must share the same staging directory (NFS mount or similar) because the stage and import calls may hit different API nodes. Alternatively, use a load balancer with session affinity for the `/stage` endpoint.

---

## Understand Image Conversion

The image conversion import plugin converts image data to a different `disk_format` during import, before writing to the store. This is typically used to convert qcow2 to raw for Ceph RBD deployments (raw images are more efficient in Ceph).

### Enable Image Conversion

```ini
[image_import_opts]
image_import_plugins = [
  glance.async_.flows._plugins.image_conversion
]

[image_conversion]
output_format = raw
```

Requirements:
- `qemu-img` must be installed on the Glance host
- The staging area must have enough space to hold both the original and converted images simultaneously

Conversion happens in the staging area before the image is committed to the store. The original format metadata in the image record is updated to reflect the converted format.

---

## Understand Image Decompression

The image decompression plugin automatically decompresses compressed images (gzip, bz2, xz) at import time, before conversion and storage.

```ini
[image_import_opts]
image_import_plugins = [
  glance.async_.flows._plugins.image_decompression,
  glance.async_.flows._plugins.image_conversion
]
```

Decompression runs before conversion in the pipeline. The decompressed image is written to a temporary file in the staging area.

---

## Understand Delayed Delete and the Scrubber

### Delayed Delete

When `delayed_delete = true`, Glance does not immediately remove image data from the store when an image is deleted. Instead:

1. The image record is marked `pending_delete` in the database.
2. The image data remains in the store.
3. The `glance-scrubber` daemon periodically checks for `pending_delete` images and removes their data.

This is a safety net: if an image is accidentally deleted, operators have a recovery window (controlled by `scrub_time`) before the data is gone.

```ini
[DEFAULT]
delayed_delete = true
scrub_time = 43200          # seconds (12 hours); data is removed after this delay
scrub_pool_size = 1000      # number of scrubber workers
scrubber_datadir = /var/lib/glance/scrubber
```

### glance-scrubber

`glance-scrubber` is a separate process that runs as a cron job or daemon. It:

1. Queries the database for images in `pending_delete` status whose `scrub_time` has elapsed.
2. Calls `glance_store.delete()` for each pending location.
3. Marks the image `deleted` in the database once all store locations are cleaned.

Running the scrubber:

```bash
# Run once and exit
glance-scrubber

# Run as a daemon
glance-scrubber --daemon
```

Scrubber configuration (`/etc/glance/glance-scrubber.conf`):

```ini
[DEFAULT]
daemon = false
wakeup_time = 300         # seconds between scrub cycles when running as daemon
scrub_time = 43200
scrubber_datadir = /var/lib/glance/scrubber
metadata_encryption_key = <same-key-as-glance-api>
```

The scrubber must have access to the same store backend credentials as `glance-api`.

---

## Understand Location Strategy

When an image has multiple locations (multi-store), Glance selects which location to serve a download from using the `location_strategy` setting.

```ini
[DEFAULT]
# Strategy for selecting a store when reading: location_order or store_type
location_strategy = location_order
```

| Strategy | Behavior |
|---|---|
| `location_order` | Try locations in the order they were added (first location wins) |
| `store_type` | Prefer locations in a specified store type order |

The `store_type` strategy requires an additional option:

```ini
[location_strategy]
store_type_preference = rbd,file,swift
```

---

## Understand Metadata Encryption

Glance can encrypt sensitive data stored in location URIs (e.g., Swift credentials embedded in a swift+https:// URI) using a symmetric key.

```ini
[DEFAULT]
metadata_encryption_key = 16-char-aes-key-here
```

The key must be exactly 16, 24, or 32 bytes (AES-128, AES-192, AES-256). The same key must be used in both `glance-api` and `glance-scrubber`. Changing this key invalidates all existing encrypted location URIs.

---

## Understand RBAC and Policy

Glance uses oslo.policy for authorization. Policy rules are defined in `/etc/glance/policy.yaml` (or the older `policy.json`).

Key policy rules:

| Rule | Default | Description |
|---|---|---|
| `add_image` | `""` (any authenticated user) | Create a new image record |
| `upload_image` | `""` | Upload image data |
| `download_image` | `""` | Download image data |
| `delete_image` | `""` | Delete an image |
| `modify_image` | `""` | Update image metadata |
| `publicize_image` | `"role:admin"` | Set image visibility to public |
| `communitize_image` | `""` | Set image visibility to community |
| `manage_image_cache` | `"role:admin"` | Use cache management API |
| `get_image_location` | `"role:admin"` | Read raw store location URIs |
| `set_image_location` | `"role:admin"` | Add/remove store locations |
| `deactivate` | `"role:admin"` | Deactivate an image |
| `reactivate` | `"role:admin"` | Reactivate a deactivated image |

Scope-based RBAC (system scope, project scope) is available in 2023.1+ releases via `[oslo_policy] enforce_scope = true`.

---

## Understand the Paste Pipeline

Glance's WSGI middleware stack is configured via `glance-api-paste.ini`. The available pipeline flavors are:

| Flavor | Middleware Stack | Use Case |
|---|---|---|
| `keystone` | authtoken + context | Standard production |
| `keystone+cachemanagement` | authtoken + context + cache + cachemanage | Production with image cache |
| `testing` | fake auth | Development and testing only |

Set the active flavor in `glance-api.conf`:

```ini
[paste_deploy]
flavor = keystone
config_file = /etc/glance/glance-api-paste.ini
```

---

## Understand the Image Signature Verification

Glance supports storing a cryptographic signature of image data, which Nova can verify before using the image. This prevents tampered images from being booted.

Image properties used for signatures:

| Property | Description |
|---|---|
| `img_signature` | Base64-encoded signature of the image data |
| `img_signature_hash_method` | Hash algorithm: `SHA-256`, `SHA-384`, `SHA-512` |
| `img_signature_key_type` | Key type: `RSA-PSS`, `DSA`, `ECC-SECT571K1`, etc. |
| `img_signature_certificate_uuid` | UUID of the certificate stored in Barbican |

Nova reads these properties and calls Barbican to retrieve the certificate to verify the signature before downloading the image for booting.

```bash
# Set signature properties when creating an image
openstack image set \
  --property img_signature=<base64-sig> \
  --property img_signature_hash_method=SHA-256 \
  --property img_signature_key_type=RSA-PSS \
  --property img_signature_certificate_uuid=<barbican-cert-uuid> \
  <image-id>
```

Nova configuration to enable signature verification:

```ini
# nova.conf
[glance]
verify_glance_signatures = true
```
