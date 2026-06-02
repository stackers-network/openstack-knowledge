# Swift Operations

## Manage Containers

### Create a container

```bash
openstack container create my-container
```

Create multiple containers in one call:

```bash
openstack container create container-a container-b container-c
```

### List containers

```bash
# List all containers in the account
openstack container list

# Show long format with object count and byte totals
openstack container list --long

# Filter by prefix
openstack container list --prefix logs-
```

### Show container metadata

```bash
openstack container show my-container
```

Output includes: container name, object count, bytes used, read ACL, write ACL, sync key, versioning location, and any custom metadata properties.

### Set container metadata and properties

```bash
# Set a custom metadata property
openstack container set --property location=us-east-1 my-container

# Remove a property
openstack container unset --property location my-container
```

### Delete a container

Containers must be empty before deletion.

```bash
openstack container delete my-container
```

Force-delete a non-empty container (deletes all objects first):

```bash
openstack container delete --recursive my-container
```

---

## Manage Objects

### Upload an object

```bash
# Upload a local file; object name defaults to the filename
openstack object create my-container local-file.txt

# Specify the object name explicitly
openstack object create my-container local-file.txt --name path/to/object.txt

# Set custom metadata on upload
openstack object create my-container local-file.txt \
  --property author=alice \
  --property project=docs
```

### List objects in a container

```bash
# Basic list
openstack object list my-container

# Long format (size, content-type, last-modified)
openstack object list my-container --long

# Filter by prefix (simulate directory listing)
openstack object list my-container --prefix docs/

# Use delimiter to get a pseudo-hierarchical listing
swift list my-container --delimiter / --prefix docs/
```

### Show object metadata

```bash
openstack object show my-container local-file.txt
```

Displays: container name, object name, content-type, content-length, ETag (MD5 of object body), last-modified, and all custom metadata headers.

### Download an object

```bash
# Save to current directory with the original filename
openstack object save my-container local-file.txt

# Save to a specific local path
openstack object save my-container local-file.txt --file /tmp/downloaded.txt

# Download all objects in a container
openstack object save --all my-container
```

### Delete an object

```bash
openstack object delete my-container local-file.txt
```

Delete multiple objects:

```bash
openstack object delete my-container file1.txt file2.txt file3.txt
```

### Set and update object metadata

POST a metadata update without re-uploading the body:

```bash
openstack object set my-container local-file.txt \
  --property reviewed=yes \
  --property reviewer=bob

openstack object unset my-container local-file.txt --property reviewed
```

### Copy an object (server-side)

```bash
# Copy within the same container
swift copy --destination /my-container/copy-of-file.txt my-container local-file.txt

# Copy to another container
swift copy --destination /other-container/local-file.txt my-container local-file.txt

# Copy and replace metadata
swift copy \
  --destination /other-container/local-file.txt \
  --fresh-metadata \
  --header "X-Object-Meta-copied=yes" \
  my-container local-file.txt
```

---

## Upload Large Objects

Swift's maximum single-PUT size is 5 GiB. Files larger than 5 GiB must be uploaded as large objects using either the Static Large Object (SLO) or Dynamic Large Object (DLO) mechanism.

### Static Large Object (SLO) — recommended

SLO stores a manifest file that lists each segment's container, object name, size, and ETag. The manifest is validated at creation time. Segments can be in any container.

Upload a large file as SLO (the Swift CLI segments automatically):

```bash
# Default segment size is 100 MiB; override with --segment-size (bytes)
swift upload --use-slo --segment-size 1073741824 my-container large-file.iso
```

With the OpenStack CLI (wraps the same underlying SLO API):

```bash
openstack object create --use-slo my-container large-file.iso
```

Manual SLO workflow (upload segments first, then create manifest):

```bash
# Upload segment 1
openstack object create my-container-segments large-file.iso/1 --file segment1.bin

# Upload segment 2
openstack object create my-container-segments large-file.iso/2 --file segment2.bin

# Create the SLO manifest (JSON body)
cat > manifest.json <<'EOF'
[
  {
    "path": "/my-container-segments/large-file.iso/1",
    "etag": "<md5-of-segment1>",
    "size_bytes": 1073741824
  },
  {
    "path": "/my-container-segments/large-file.iso/2",
    "etag": "<md5-of-segment2>",
    "size_bytes": 524288000
  }
]
EOF

# PUT the manifest with ?multipart-manifest=put
curl -X PUT \
  -H "X-Auth-Token: $OS_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  "https://swift.example.com/v1/AUTH_<account>/my-container/large-file.iso?multipart-manifest=put" \
  --data-binary @manifest.json
```

Retrieve the raw manifest (without reassembling the object):

```bash
curl -H "X-Auth-Token: $OS_AUTH_TOKEN" \
  "https://swift.example.com/v1/AUTH_<account>/my-container/large-file.iso?multipart-manifest=get"
```

### Dynamic Large Object (DLO)

DLO assembles segments at GET time by listing all objects sharing a common prefix. No manifest validation is performed at upload. Segments must all be in the same container (or a designated segment container).

```bash
# DLO upload via swift CLI (default for large files unless --use-slo specified)
swift upload --segment-size 1073741824 my-container large-file.iso
```

This creates:
- Segments in `my-container_segments/large-file.iso/<timestamp>/<total-size>/NNNNNNNNNN` naming pattern.
- A manifest object at `my-container/large-file.iso` with `X-Object-Manifest: my-container_segments/large-file.iso/<timestamp>/<total-size>/`.

DLO is simpler to use but provides weaker consistency guarantees — a new segment upload overlapping a read can return a mixed result.

---

## Configure Container ACLs

Swift ACLs are set on containers via the `X-Container-Read` and `X-Container-Write` headers. Each ACL is a comma-separated list of ACL elements.

### Make a container publicly readable (anonymous read)

```bash
openstack container set --property 'read-acl=.r:*' my-container
```

`.r:*` means any HTTP client (including unauthenticated) can GET objects. The container listing remains private.

### Allow both anonymous read and public container listing

```bash
openstack container set --property 'read-acl=.r:*,.rlistings' my-container
```

### Restrict read to a specific referrer domain

```bash
openstack container set \
  --property 'read-acl=.r:www.example.com,.r:cdn.example.com' \
  my-container
```

### Grant read to all authenticated users in the project

```bash
openstack container set \
  --property "read-acl=<project-id>:*" \
  my-container
```

### Grant read to a specific user

```bash
openstack container set \
  --property "read-acl=<project-id>:<user-id>" \
  my-container
```

### Grant write access

```bash
# Allow all users in a project to write
openstack container set \
  --property "write-acl=<project-id>:*" \
  my-container
```

### Remove ACLs (make container private again)

```bash
openstack container unset --property read-acl my-container
openstack container unset --property write-acl my-container
```

### Inspect current ACLs via swift CLI

```bash
swift stat my-container
# Look for X-Container-Read and X-Container-Write in the output
```

---

## Configure Object Versioning

Swift supports two versioning modes: history mode (preserves all versions) and stack mode (only the most recent N versions).

### Enable versioning on a container (history mode)

Create a separate archive container to hold old versions:

```bash
openstack container create my-container-archive

openstack container set \
  --property 'versions-location=my-container-archive' \
  my-container
```

Now every PUT to an existing object in `my-container` moves the current version to `my-container-archive` before storing the new version. DELETE on an object in `my-container` removes the current version and restores the most recent archived version.

### List archived versions

```bash
openstack object list my-container-archive
# Archived objects are named: <object-name-length><object-name>/<timestamp>
```

### Enable versioning (stack mode — newer API)

Use `X-Versions-Enabled` instead of `X-Versions-Location`:

```bash
swift post --header 'X-Versions-Enabled: true' my-container
```

In stack mode, a GET on `?version_id=<id>` retrieves a specific version. A DELETE creates a delete marker rather than restoring the previous version.

### List versions of an object (stack mode)

```bash
swift list my-container --versions
```

### Disable versioning

```bash
openstack container unset --property versions-location my-container
# or
swift post --header 'X-Versions-Enabled: false' my-container
```

---

## Generate Temp URLs

Temp URLs allow time-limited access to a specific object without requiring a Keystone token. The URL is signed with an HMAC-SHA256 signature derived from a secret key.

### Set a temp URL key on the account

```bash
swift post -m "Temp-URL-Key: my-secret-key-value"
# Verify it was set
swift stat | grep Temp-URL-Key
```

A second key can be set simultaneously to allow key rotation without invalidating existing URLs:

```bash
swift post -m "Temp-URL-Key-2: my-second-secret-key"
```

### Generate a temp URL

```bash
# GET access, valid for 3600 seconds (1 hour)
swift tempurl GET 3600 /v1/AUTH_<account-id>/my-container/local-file.txt my-secret-key-value
```

Output is the full path with `temp_url_sig` and `temp_url_expires` query parameters appended:

```
/v1/AUTH_<account-id>/my-container/local-file.txt?temp_url_sig=<hmac>&temp_url_expires=<epoch>
```

Prepend the Swift endpoint to get a usable URL:

```bash
echo "https://swift.example.com$(swift tempurl GET 3600 /v1/AUTH_<account-id>/my-container/local-file.txt my-secret-key-value)"
```

### Generate a PUT temp URL (allow anonymous upload)

```bash
swift tempurl PUT 3600 /v1/AUTH_<account-id>/my-container/upload-target.txt my-secret-key-value
```

### Temp URL with filename override

Add `&filename=download.txt` to the URL to set the `Content-Disposition` header in the response, controlling the browser's Save As filename.

---

## Use Static Web Middleware

Swift can serve a container as a static website when the `staticweb` middleware is in the proxy pipeline.

```bash
# Enable web listing on a container
swift post -m 'Web-Listings: true' my-website
swift post -m 'Web-Index: index.html' my-website
swift post -m 'Web-Error: error.html' my-website

# Make the container publicly readable
openstack container set --property 'read-acl=.r:*,.rlistings' my-website
```

Objects are then accessible at `https://swift.example.com/v1/AUTH_<account>/my-website/path`.

---

## Perform Bulk Operations

The `bulk` middleware enables two special operations: bulk delete and bulk archive extract.

### Bulk delete

Send a list of objects to delete in a single request (up to 10,000 per request):

```bash
# Create a file listing objects to delete (one per line: container/object)
cat > to-delete.txt <<'EOF'
my-container/old-file-1.txt
my-container/old-file-2.txt
my-container/old-file-3.txt
EOF

curl -X DELETE \
  -H "X-Auth-Token: $OS_AUTH_TOKEN" \
  -H "Content-Type: text/plain" \
  "https://swift.example.com/v1/AUTH_<account>?bulk-delete" \
  --data-binary @to-delete.txt
```

### Bulk extract (upload a tar archive)

Upload a tar archive and have Swift extract it into containers/objects matching the archive's directory structure:

```bash
# Create a tar archive
tar -czf upload.tar.gz -C /local/data .

# Extract into Swift under my-container/
curl -X PUT \
  -H "X-Auth-Token: $OS_AUTH_TOKEN" \
  -H "Content-Type: application/tar+gzip" \
  "https://swift.example.com/v1/AUTH_<account>/my-container?extract-archive=tar.gz" \
  --data-binary @upload.tar.gz
```

The archive's top-level directories become object name prefixes inside `my-container`.

---

## Manage Storage Policies via CLI

### Create a container on a non-default storage policy

```bash
# Using the swift CLI
swift post --policy ec-policy my-ec-container

# Using curl
curl -X PUT \
  -H "X-Auth-Token: $OS_AUTH_TOKEN" \
  -H "X-Storage-Policy: ec-policy" \
  "https://swift.example.com/v1/AUTH_<account>/my-ec-container"
```

The policy is recorded in the container's metadata and cannot be changed after creation.

### Check which policy a container uses

```bash
swift stat my-container
# Look for X-Storage-Policy in the output

openstack container show my-container
# Look for storage_policy in the output
```

---

## Use the swift CLI for Advanced Operations

The `swift` CLI (from `python-swiftclient`) provides access to Swift-specific features not always available through `openstack container/object` commands.

### Upload with explicit segment container

```bash
swift upload \
  --use-slo \
  --segment-size 1073741824 \
  --segment-container my-segments \
  --object-name large-file.iso \
  my-container \
  /local/path/large-file.iso
```

### Upload entire directory tree

```bash
swift upload my-container /local/directory/
# Objects are named relative to the parent of /local/directory/
```

### Download entire container

```bash
swift download my-container --output-dir /local/output/
```

### Show account statistics

```bash
swift stat
# Shows: Account, Containers, Objects, Bytes, X-Account-Meta-*, Temp-URL-Key
```

### Post account-level metadata

```bash
swift post -m "X-Account-Meta-Owner: ops-team"
```

---

## Work with the Ring Builder

`swift-ring-builder` is the command-line tool for creating and managing rings. It operates on `.builder` files. Detailed ring building is in [internals.md](internals.md); this section covers the most common operational commands.

### Show ring contents

```bash
swift-ring-builder /etc/swift/object.builder
```

Output shows: device count, partition count, replica count, min part hours, ring balance, and a table of all devices with their assigned partition count and weight.

### Verify ring balance

A balanced ring has `balance` close to 0. A balance of 10 means some devices hold 10% more partitions than their weight would suggest.

```bash
swift-ring-builder /etc/swift/object.builder
# Look for "Ring file object.ring.gz is up-to-date" and "balance: X.xx"
```

### Search for a specific device

```bash
swift-ring-builder /etc/swift/object.builder search --ip 10.0.0.5
swift-ring-builder /etc/swift/object.builder search --device sdb
```

### Remove a device (set weight to 0 before removing)

```bash
# First, set weight to 0 to drain partitions gradually
swift-ring-builder /etc/swift/object.builder set_weight --ip 10.0.0.5 --device sdb 0
swift-ring-builder /etc/swift/object.builder rebalance

# After replication completes and device is empty, remove it
swift-ring-builder /etc/swift/object.builder remove --ip 10.0.0.5 --device sdb
swift-ring-builder /etc/swift/object.builder rebalance
```

### Write the ring file and distribute

```bash
swift-ring-builder /etc/swift/object.builder write_ring
# Distribute the .ring.gz to all proxy and storage nodes
scp /etc/swift/object.ring.gz storage1:/etc/swift/
scp /etc/swift/object.ring.gz storage2:/etc/swift/
scp /etc/swift/object.ring.gz proxy1:/etc/swift/
```

The proxy and storage servers reload the ring file automatically (checks modification time on each request by default, or on SIGHUP).
