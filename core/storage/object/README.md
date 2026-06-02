# Object Storage — Swift

Swift is the OpenStack Object Storage service. It stores and retrieves unstructured data — files, blobs, backups, logs, media — as named objects inside containers inside accounts. There is no filesystem hierarchy, no block device, and no POSIX interface. Objects are addressed by a three-level namespace: `account/container/object`.

Swift was one of the first OpenStack projects and predates the OpenStack Foundation. It is a fully distributed, eventually consistent system with no single point of failure. Every component is horizontally scalable. Data durability is achieved through configurable replication (default: 3 replicas) or erasure coding across failure domains (zones, regions).

## When to Read This Skill

- Deploying Swift for the first time or adding capacity
- Creating containers and uploading objects via CLI, API, or SDK
- Configuring container ACLs, object versioning, temp URLs
- Uploading large objects (> 5 GiB) using SLO or DLO
- Building or rebalancing rings after adding or removing hardware
- Understanding replication, auditor, and updater background processes
- Configuring erasure coding storage policies
- Setting up encryption at rest with keymaster middleware
- Diagnosing object loss, ring imbalance, or replication lag
- Tuning proxy pipeline middleware (rate limiting, temp URL, SLO, DLO)

## Sub-Files

| File | What It Covers |
|---|---|
| [architecture.md](architecture.md) | Proxy server, account/container/object servers, ring architecture (partitions, replicas, zones, regions), consistent hashing, eventually consistent model, storage policies, WSGI pipeline |
| [operations.md](operations.md) | CLI and API: containers, objects, large objects (SLO/DLO), ACLs, versioning, temp URLs, bulk operations, static websites; swift-ring-builder overview |
| [internals.md](internals.md) | Ring building in detail, replicator/auditor/updater processes, consistency engine (hashes.pkl), erasure coding (liberasurecode), storage policy configuration, proxy pipeline middleware, encryption at rest, account reaper |

## Quick Reference

```bash
# Create a container
openstack container create my-container

# Upload an object
openstack object create my-container local-file.txt

# List objects in a container
openstack object list my-container

# Download an object
openstack object save my-container local-file.txt

# Delete an object
openstack object delete my-container local-file.txt

# Delete a container
openstack container delete my-container

# Make a container publicly readable
openstack container set --property 'read-acl=.r:*,.rlistings' my-container

# Generate a temp URL (valid 1 hour)
swift tempurl GET 3600 /v1/AUTH_<account-id>/my-container/local-file.txt <temp-url-key>

# Show container metadata
openstack container show my-container

# Upload a large file as a Static Large Object
openstack object create --use-slo my-container large-file.iso
```

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| Keystone | Authentication and account mapping | Swift maps Keystone project ID to account; authtoken middleware validates tokens |
| Memcached | Auth token cache, account/container metadata cache in proxy | Required in production; dramatically reduces account/container DB lookups |
| SQLite (per-server) | Account and container databases | One SQLite DB per account, per container; stored on local disk alongside objects |
| `python-openstackclient` + `python-swiftclient` | Unified CLI and Swift-specific CLI | `openstack` for standard ops; `swift` CLI for advanced Swift-specific features |
| `liberasurecode` | Erasure coding library | Required only when EC storage policies are configured |
| `python-keystonemiddleware` | Auth token validation in proxy pipeline | Validates Keystone tokens; required for Keystone-integrated deployments |

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

Swift release notes: https://docs.openstack.org/releasenotes/swift/
Swift admin guide: https://docs.openstack.org/swift/latest/admin_guide.html
Swift API reference: https://docs.openstack.org/api-ref/object-store/
Swift ring overview: https://docs.openstack.org/swift/latest/overview_ring.html
