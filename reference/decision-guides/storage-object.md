# Choose an Object Storage Backend

OpenStack object storage can be served by Swift (the native OpenStack object storage service) or by Ceph RADOS Gateway (RadosGW/RGW), which provides S3-compatible and Swift-compatible APIs on top of a Ceph cluster.

## Feature Comparison

| Feature | Swift | Ceph RadosGW (RGW) |
|---|---|---|
| S3 API | Partial (via `s3api` middleware, most S3 operations) | Full S3 compatibility including ACLs, multipart, versioning |
| Swift API | Native | Compatible (Swift API implemented on top of RGW) |
| Standalone deployment | Yes (no other services needed) | No (requires a running Ceph cluster) |
| Erasure coding | Yes (EC policies) | Yes (per-pool EC) |
| Multi-site replication | Yes (native async replication between regions) | Yes (multi-zone, multi-region with sync agents) |
| Object versioning | Yes (X-Versions-Location) | Yes (S3 versioning API) |
| Object expiration | Yes (X-Delete-At / X-Delete-After) | Yes (lifecycle policies) |
| Static website hosting | Yes | Yes |
| Encryption at rest | Yes (server-side, KMIP/Barbican) | Yes (SSE-S3, SSE-KMS) |
| Access control | Swift ACLs, Keystone integration | S3 ACLs, bucket policies, Keystone integration |
| Performance | Good (scales with nodes) | Good (scales with Ceph OSDs) |
| Minimum nodes | 5 (3 storage + 2 proxy recommended) | Depends on Ceph cluster (min 3 OSD nodes) |
| Operational model | Dedicated Swift cluster | Shared Ceph cluster (RGW is one Ceph component) |
| Monitoring | Swift-specific metrics | Ceph dashboard, Prometheus exporter |
| Complexity | Medium (Swift-specific rings and config) | High if Ceph is new; Low if Ceph already deployed |
| Best when | Dedicated object storage needed, no Ceph, maximum Swift API fidelity | Already running Ceph for block (Cinder RBD) or images (Glance) |

## Recommendation by Use Case

### No Existing Ceph, Object Storage Only
**Use Swift.** Swift is purpose-built for object storage, requires no other infrastructure, and has excellent multi-site replication support. It handles very large clusters (petabytes) well with consistent hashing (rings).

### Already Running Ceph for Block or Images
**Use Ceph RadosGW.** Adding RGW to an existing Ceph cluster is low incremental cost — the OSDs are already there. You gain S3 compatibility with a single infrastructure investment.

### Strong S3 API Compatibility Required
**Use Ceph RadosGW.** RGW's S3 compatibility is broad and frequently tested against the S3 test suite. Swift's `s3api` middleware covers most operations but has gaps, especially around ACL syntax and some multipart edge cases.

### OpenStack-Native Swift API Required
**Use Swift.** Applications using Swift-specific features (DLO, SLO, bulk operations, form POST middleware, Swift container sync) should use Swift natively. RGW's Swift compatibility is good but not identical.

### Multi-site Object Storage
Both support multi-site, but the topology models differ:

- **Swift**: Ring-based consistency, async replication between regions via `container-sync` or Global Cluster (separate ring per region with inter-region proxy).
- **Ceph RGW**: Zone/zonegroup model with RGW sync agents. Easier to manage via the `radosgw-admin` CLI and Ceph dashboard.

For new multi-site deployments with existing Ceph, prefer RGW. For pure object-storage multi-site without Ceph, prefer Swift.

## Swift Architecture Overview

Swift has three storage daemons (account, container, object) and one or more proxy servers:

```
Clients → Swift Proxy (HAProxy/LB) → Swift Proxy Nodes
                                    → Account/Container/Object nodes
                                    → (Ceph, local disks, or EC storage policies)
```

### Key Swift Commands

```bash
# Check cluster health
swift-ring-builder /etc/swift/object.builder
swift-ring-builder /etc/swift/container.builder
swift-ring-builder /etc/swift/account.builder

# Check dispersion
swift-dispersion-report

# Check replication status
swift-recon --replication --verbose
swift-recon --diskusage

# List containers and objects
openstack container list
openstack object list <CONTAINER>
openstack object show <CONTAINER> <OBJECT>

# Upload and download
openstack object create <CONTAINER> <LOCAL_FILE>
openstack object save <CONTAINER> <OBJECT> --file <LOCAL_PATH>
```

## Ceph RadosGW Architecture Overview

RGW is a Ceph component that exposes S3 and Swift HTTP APIs backed by RADOS:

```
Clients → RGW (Beast frontend, multiple instances behind HAProxy)
        → RADOS (shared with RBD/CephFS if present)
        → OSD nodes
```

### Key RGW Commands

```bash
# RGW daemon status
systemctl status ceph-radosgw@rgw.<hostname>.*

# List users
radosgw-admin user list
radosgw-admin user info --uid=<USER>

# List buckets
radosgw-admin bucket list
radosgw-admin bucket stats --bucket=<BUCKET>

# Zone/zonegroup status
radosgw-admin zone get
radosgw-admin zonegroup get
radosgw-admin sync status

# Pool usage
rados df | grep "rgw\|default"

# RGW log
journalctl -u ceph-radosgw@rgw.$(hostname).rgw0 --since "10 minutes ago"
```

## Glance with Object Storage Backend

Both Swift and RGW can back Glance image storage:

**Swift backend:**

```ini
# /etc/glance/glance-api.conf
[glance_store]
stores = swift
default_store = swift
swift_store_auth_address = http://<KEYSTONE_HOST>:5000/v3
swift_store_user = service:glance
swift_store_key = <GLANCE_PASSWORD>
swift_store_container = glance
swift_store_create_container_on_put = true
```

**RGW backend (S3 interface):**

```ini
# /etc/glance/glance-api.conf
[glance_store]
stores = s3
default_store = s3
s3_store_host = http://<RGW_HOST>:7480
s3_store_access_key = <ACCESS_KEY>
s3_store_secret_key = <SECRET_KEY>
s3_store_bucket = glance
s3_store_create_bucket_on_put = true
```

## Operational Checks

```bash
# Swift — overall health
swift-recon -a  # all checks
swift-recon --md5  # ring file consistency

# Swift — per-node disk usage
swift-recon --diskusage | sort -k5 -rn | head -20

# RGW — S3 API health check
curl -s http://<RGW_HOST>:7480/
# Should return S3 ListAllMyBuckets XML

# RGW — sync status (multi-site)
radosgw-admin sync status
# Look for: "realm", "data is caught up with source", no errors
```
