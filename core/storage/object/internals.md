# Swift Internals

## Build Rings with swift-ring-builder

The ring builder is the off-line tool that constructs the partition-to-device mapping consumed by the proxy and all storage servers. It operates on a `.builder` state file and outputs a `.ring.gz` binary. All ring changes must be done on the builder file; the `.ring.gz` is a read-only artefact derived from it.

### Create a new ring

```bash
# Syntax: swift-ring-builder <builder-file> create <partition-power> <replicas> <min-part-hours>
swift-ring-builder /etc/swift/account.builder   create 18 3 1
swift-ring-builder /etc/swift/container.builder create 18 3 1
swift-ring-builder /etc/swift/object.builder    create 18 3 1
```

Parameters:

| Parameter | Meaning |
|---|---|
| `partition-power` | `2^N` partitions. 18 → 262,144 partitions. Larger clusters need higher values. |
| `replicas` | How many copies of each partition to keep. Default is 3. Can be a float (e.g., 2.5) for intermediate replica counts during migrations. |
| `min-part-hours` | Minimum hours before a partition can be moved again after a rebalance. Prevents excessive movement during rolling rebalances. Typically 1 for testing, 24 for production. |

### Add devices to the ring

```bash
# Syntax:
# swift-ring-builder <builder-file> add
#   --region <region-id>
#   --zone <zone-id>
#   --ip <ip>
#   --port <port>
#   --device <device-name>
#   --weight <weight>
#   [--meta <annotation>]

# Account ring (port 6202)
swift-ring-builder /etc/swift/account.builder add \
  --region 1 --zone 1 --ip 10.0.0.1 --port 6202 --device sdb --weight 100
swift-ring-builder /etc/swift/account.builder add \
  --region 1 --zone 2 --ip 10.0.0.2 --port 6202 --device sdb --weight 100
swift-ring-builder /etc/swift/account.builder add \
  --region 1 --zone 3 --ip 10.0.0.3 --port 6202 --device sdb --weight 100

# Container ring (port 6201)
swift-ring-builder /etc/swift/container.builder add \
  --region 1 --zone 1 --ip 10.0.0.1 --port 6201 --device sdb --weight 100
swift-ring-builder /etc/swift/container.builder add \
  --region 1 --zone 2 --ip 10.0.0.2 --port 6201 --device sdb --weight 100
swift-ring-builder /etc/swift/container.builder add \
  --region 1 --zone 3 --ip 10.0.0.3 --port 6201 --device sdb --weight 100

# Object ring (port 6200)
swift-ring-builder /etc/swift/object.builder add \
  --region 1 --zone 1 --ip 10.0.0.1 --port 6200 --device sdb --weight 100
swift-ring-builder /etc/swift/object.builder add \
  --region 1 --zone 2 --ip 10.0.0.2 --port 6200 --device sdb --weight 100
swift-ring-builder /etc/swift/object.builder add \
  --region 1 --zone 3 --ip 10.0.0.3 --port 6200 --device sdb --weight 100
```

The `--weight` value is proportional to storage capacity. A 4 TB drive should have twice the weight of a 2 TB drive. The absolute value does not matter, only the ratio between devices.

### Add multiple drives on the same node

```bash
for dev in sdb sdc sdd sde; do
  swift-ring-builder /etc/swift/object.builder add \
    --region 1 --zone 1 --ip 10.0.0.1 --port 6200 \
    --device $dev --weight 100
done
```

### Rebalance the ring

```bash
swift-ring-builder /etc/swift/account.builder   rebalance
swift-ring-builder /etc/swift/container.builder rebalance
swift-ring-builder /etc/swift/object.builder    rebalance
```

The rebalancer assigns partitions to devices proportionally to weight while respecting the replica placement constraints (unique regions, unique zones, unique servers, unique drives). It honours `min-part-hours` and will not move more than one replica of a partition in a single rebalance if that would violate the hour constraint.

After rebalancing, the builder writes the updated `.ring.gz` file. Check the output for balance and dispersion:

```
Reassigned 786432 (100.00%) partitions. Balance is now 0.00.  Dispersion is now 0.00
```

### Validate the ring before distributing

```bash
swift-ring-builder /etc/swift/object.builder
```

Key output lines:
- `balance: X.XX` — should be < 1 for a well-distributed ring
- `dispersion: X.XX` — measures how many partitions have replicas in fewer unique tiers than expected; 0 is ideal
- `Ring file object.ring.gz is up-to-date` — the `.ring.gz` reflects the current builder state

### Add a device to an existing production ring

Do not add many devices at once. Add in batches and rebalance between batches to limit data movement:

```bash
# 1. Add the new device(s)
swift-ring-builder /etc/swift/object.builder add \
  --region 1 --zone 4 --ip 10.0.0.4 --port 6200 --device sdb --weight 100

# 2. Rebalance
swift-ring-builder /etc/swift/object.builder rebalance

# 3. Distribute the new ring.gz to all nodes
scp /etc/swift/object.ring.gz storage1:/etc/swift/
# (repeat for all nodes)

# 4. Wait for replication to settle (monitor replication lag)
# 5. Repeat for next batch of devices
```

### Gradually increase weight (staged weight increase)

When adding a large new device to an existing ring, avoid moving all partitions at once by introducing the device at low weight and gradually increasing it:

```bash
# Add at 10% final weight
swift-ring-builder /etc/swift/object.builder add \
  --region 1 --zone 1 --ip 10.0.0.5 --port 6200 --device sdb --weight 10
swift-ring-builder /etc/swift/object.builder rebalance
# (distribute, wait for replication)

# Increase to 50%
swift-ring-builder /etc/swift/object.builder set_weight --ip 10.0.0.5 --device sdb 50
swift-ring-builder /etc/swift/object.builder rebalance
# (distribute, wait)

# Final weight
swift-ring-builder /etc/swift/object.builder set_weight --ip 10.0.0.5 --device sdb 100
swift-ring-builder /etc/swift/object.builder rebalance
```

### Show partition placement for a specific path

```bash
swift-ring-builder /etc/swift/object.builder search \
  --partition $(swift-ring-builder /etc/swift/object.builder \
    --format json | python3 -c "
import sys, json, hashlib, struct
ring = json.load(sys.stdin)
# ... partition math
")
```

For a quicker lookup, use the `swift` CLI:

```bash
swift-get-nodes /etc/swift/object.ring.gz AUTH_<account-id> my-container my-object.txt
```

Output shows the three (or N) storage nodes that hold replicas of that object, plus handoff nodes.

---

## Understand the Replication System

### Object replicator

`swift-object-replicator` runs continuously (or on a configurable interval) on every storage node. Its job is to ensure every partition on the local node has accurate copies on the correct partner nodes as defined by the ring.

The replication algorithm:

1. Walk all partition directories on the local disk.
2. For each partition, compute the MD5 hash tree of its suffix directories (stored in `hashes.pkl`).
3. Contact each of the other replica nodes for that partition.
4. Compare suffix hashes. If any suffix hash differs, rsync the differing suffix directory to the remote node.
5. If the local node holds a handoff partition (a partition whose primary ring assignment is elsewhere), push it to the primary and delete the local copy once the remote confirms receipt.

Configuration in `object-server.conf` (or `object-replicator.conf`):

```ini
[object-replicator]
vm_test_mode = no
daemonize = on
run_pause = 30          # seconds between replication passes
concurrency = 1         # number of replication jobs in parallel
timeout = 5             # socket timeout for backend connections
node_timeout = 10
http_timeout = 60
rsync_timeout = 900
rsync_bwlimit = 0       # 0 = unlimited; set to limit bandwidth (KB/s)
reclaim_age = 604800    # 7 days; tombstone/.ts files older than this are deleted
```

### Container and account replicators

`swift-container-replicator` and `swift-account-replicator` replicate SQLite databases rather than flat files. The protocol is:

1. The local server computes a hash of its SQLite row data.
2. It sends a REPLICATE request to each partner node.
3. Partners compare hashes and respond with any rows the sender is missing.
4. The sender merges missing rows into its local database.

This merge-based approach means container/account replication is a database synchronisation problem, not a file copy problem.

---

## Understand the Consistency Engine

### hashes.pkl and suffix directories

Swift's consistency model is built around a file-based hash tree. Object files are organised under:

```
/srv/node/<device>/objects/<partition>/
```

Within each partition directory there are **suffix directories** — directories whose name is the last three hex digits of the object hash. Within each suffix is a directory named after the full object hash, and inside that is the timestamped data file.

```
/srv/node/sdb/objects/12345/abc/d4e5f6.../1617000000.00000.data
                       ^^^   ^^^^^^^^^^^^^^^^^^^^^^^^
                    suffix   object hash dir
```

`hashes.pkl` is a Python pickle file stored in each partition directory. It maps each suffix directory name to an MD5 hash of the concatenated filenames (including timestamps) of all objects in that suffix. When any object in a suffix is added, deleted, or modified, the suffix's hash in `hashes.pkl` is invalidated (set to `None`).

During replication, the replicator reads `hashes.pkl`. For each suffix with a `None` hash, it recomputes the hash by listing the suffix directory. It then sends the full suffix-to-hash dict to partner nodes. Partners respond with the list of suffixes where their hash differs. The replicator then rsyncs only those differing suffixes.

This design avoids full scans: only changed suffixes are transmitted, and only changed objects within those suffixes are rsynced.

### Auditor

`swift-object-auditor` reads every `.data` file on disk, computes its MD5 checksum, and compares it against the stored ETag (which is the MD5 of the original upload). If they differ, the auditor quarantines the file by moving it to `/srv/node/<device>/quarantined/objects/<partition>/<suffix>/<hash>/`. A quarantined object causes the replicator to fetch a fresh copy from a healthy replica on its next pass.

The auditor also validates extended attributes (xattrs) contain valid metadata. An object with corrupt xattrs is also quarantined.

Run frequency is controlled by `object_auditor.conf`:

```ini
[object-auditor]
files_per_second = 20    # rate limit to avoid I/O saturation
bytes_per_second = 10000000
log_time = 3600          # log statistics every N seconds
zero_byte_files_per_second = 50
```

### Updater

When an object is uploaded but the container server is unavailable, the object server writes a **async pending** file (in `/srv/node/<device>/async_pending/`) recording the container database update that could not be sent. `swift-object-updater` periodically replays these pending updates, sending them to the container server when it becomes available.

This decouples object writes from container metadata consistency. An object may be stored successfully even if the container listing is temporarily stale.

---

## Configure Storage Policies

Storage policies are defined in `/etc/swift/swift.conf`. Every Swift cluster must have at least one policy (index 0), which serves as the default.

```ini
[swift-hash]
swift_hash_path_prefix = changeme_prefix
swift_hash_path_suffix = changeme_suffix

[storage-policy:0]
name = Policy-0
default = yes
policy_type = replication

[storage-policy:1]
name = gold-replication
policy_type = replication
# Uses object-1.ring.gz

[storage-policy:2]
name = ec-standard
policy_type = erasure_coding
ec_type = liberasurecode_rs_vand
ec_num_data_fragments = 10
ec_num_parity_fragments = 4
ec_object_segment_size = 1048576
# Uses object-2.ring.gz
```

Each storage policy needs its own object ring:

```bash
# Policy 0 (default): object.builder / object.ring.gz
# Policy 1: object-1.builder / object-1.ring.gz
# Policy 2: object-2.builder / object-2.ring.gz

swift-ring-builder /etc/swift/object-1.builder create 18 3 1
swift-ring-builder /etc/swift/object-2.builder create 18 14 1  # 10+4 EC = 14 fragments
```

`swift_hash_path_prefix` and `swift_hash_path_suffix` are secrets that salt the consistent hash. They must be identical on all nodes in the cluster and must never change after deployment. Changing them invalidates all ring mappings.

---

## Understand Erasure Coding

### liberasurecode

Swift's EC support is built on `liberasurecode`, a C library abstraction layer over multiple underlying EC backends:

| Backend | Description |
|---|---|
| `liberasurecode_rs_vand` | Reed-Solomon Vandermonde matrix (pure software, always available) |
| `isa_l_rs_vand` | Intel ISA-L Reed-Solomon (hardware-accelerated on Intel CPUs) |
| `isa_l_rs_cauchy` | Intel ISA-L Cauchy matrix RS |
| `jerasure_rs_vand` | Jerasure library RS Vandermonde |
| `flat_xor_hd` | Simple XOR-based code; low CPU but limited parameters |
| `shss` | NTT Storage's erasure code |

Verify installed backends:

```bash
python3 -c "import liberasurecode; print(liberasurecode.get_backends())"
```

### How EC objects are stored

When a client PUTs an object to an EC container, the proxy server:

1. Buffers an `ec_object_segment_size`-sized segment (default 1 MiB).
2. Encodes the segment using the EC scheme, producing `k + m` fragments.
3. Sends each fragment to a different storage node (one fragment per node).
4. Repeats for each subsequent segment.
5. Writes a **durable file** marker (`.durable`) once all fragments for the full object are confirmed written to a quorum of `k + 1` nodes.

Object files under EC policies have the extension `.#N#data` where `N` is the fragment index (0 through k+m-1):

```
/srv/node/sdb/objects-2/12345/abc/d4e5f6.../<timestamp>#0#data
```

Fragment index `N` is stored as part of the filename and also in the object's xattrs.

### EC reconstruction

On GET, the proxy contacts all `k + m` nodes. If at least `k` fragments are available, the proxy reconstructs the object in memory and streams it to the client. With `liberasurecode_rs_vand 10 4`, the cluster can survive up to 4 simultaneous node/drive failures without data loss.

### EC replicator (reconstructor)

EC policies use `swift-object-reconstructor` instead of `swift-object-replicator`. When a node comes back after a failure, the reconstructor:

1. Identifies partitions where local fragments are missing or out of date.
2. Contacts the other fragment-holding nodes.
3. Fetches `k` fragments and reconstructs the missing fragment locally.
4. Writes the reconstructed fragment to local disk.

```ini
[object-reconstructor]
concurrency = 1
daemonize = on
run_pause = 30
```

---

## Configure the Proxy Pipeline Middleware

Full production pipeline in `/etc/swift/proxy-server.conf`:

```ini
[DEFAULT]
bind_ip = 0.0.0.0
bind_port = 8080
workers = 8
user = swift
swift_dir = /etc/swift
devices = /srv/node
log_facility = LOG_LOCAL1
log_level = INFO

[pipeline:main]
pipeline = catch_errors gatekeeper healthcheck proxy-logging cache listing_formats bulk tempurl ratelimit authtoken keystoneauth copy container_quotas account_quotas slo dlo proxy-logging proxy-server

[app:proxy-server]
use = egg:swift#proxy
allow_account_management = true
account_autocreate = true

[filter:catch_errors]
use = egg:swift#catch_errors

[filter:gatekeeper]
use = egg:swift#gatekeeper

[filter:healthcheck]
use = egg:swift#healthcheck

[filter:proxy-logging]
use = egg:swift#proxy-logging
access_log_headers = false

[filter:cache]
use = egg:swift#memcache
memcache_servers = 10.0.0.1:11211,10.0.0.2:11211,10.0.0.3:11211

[filter:listing_formats]
use = egg:swift#listing_formats

[filter:bulk]
use = egg:swift#bulk
max_deletes_per_request = 10000
max_failed_deletes = 1000

[filter:tempurl]
use = egg:swift#tempurl
methods = GET HEAD PUT POST DELETE
incoming_remove_headers = x-timestamp
incoming_allow_headers =
outgoing_remove_headers = x-object-meta-*
outgoing_allow_headers = x-object-meta-public x-object-meta-content-type

[filter:ratelimit]
use = egg:swift#ratelimit
clock_accuracy = 1000
max_sleep_time_seconds = 60
log_sleep_time_seconds = 0
rate_buffer_seconds = 5
account_ratelimit = 0
container_ratelimit_0 = 0
container_ratelimit_10 = 50

[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
www_authenticate_uri = https://keystone.example.com:5000
auth_url = https://keystone.example.com:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = swift
password = <swift-service-password>
delay_auth_decision = True
cache = swift.cache
include_service_catalog = False

[filter:keystoneauth]
use = egg:swift#keystoneauth
operator_roles = admin,swiftoperator
reseller_prefix = AUTH_
default_domain_id = default

[filter:copy]
use = egg:swift#copy
object_post_as_copy = false

[filter:container_quotas]
use = egg:swift#container_quotas

[filter:account_quotas]
use = egg:swift#account_quotas

[filter:slo]
use = egg:swift#slo
max_manifest_segments = 1000
max_manifest_size = 2097152
min_segment_size = 1048576
max_get_time = 86400

[filter:dlo]
use = egg:swift#dlo
rate_limit_after_segment = 10
rate_limit_segments_per_sec = 1
max_get_time = 86400
```

Key configuration decisions:

- `account_autocreate = true`: proxy automatically creates the Swift account when a Keystone project is first used. Without this, accounts must be pre-created.
- `delay_auth_decision = True` in `authtoken`: passes unrecognised tokens downstream rather than rejecting them; required so that `tempauth` or `keystoneauth` can handle alternative auth mechanisms.
- `operator_roles`: Keystone roles that grant full Swift operator access. Users with the `admin` or `swiftoperator` role on a project get full read/write access to that project's Swift account.
- `reseller_prefix`: prefix prepended to Keystone project IDs to form Swift account names (`AUTH_` is the default, giving `AUTH_<project-id>`).

---

## Configure Encryption at Rest

Swift supports transparent encryption of object data and metadata at rest using two middleware components: **keymaster** and **encryption**.

### How encryption works

- **keymaster** derives a unique encryption key for each object from a root secret and the object path using HMAC-SHA256. No two objects share a key.
- **encryption** intercepts PUT requests from the proxy, encrypts the body and metadata before writing to object servers, and decrypts transparently on GET.
- Encryption is transparent to clients — they see plaintext over HTTPS.
- Keys are never stored on disk alongside data. The key is re-derived from the root secret on each access.

### Configure keymaster

```ini
# In proxy-server.conf pipeline, add keymaster and encryption before proxy-server
[pipeline:main]
pipeline = ... authtoken keystoneauth copy container_quotas account_quotas slo dlo keymaster encryption proxy-logging proxy-server

[filter:keymaster]
use = egg:swift#keymaster
# Path to a file containing the root encryption secret (a base64-encoded 256-bit key)
encryption_root_secret_path = /etc/swift/keymaster.conf

# Or inline (not recommended for production):
# encryption_root_secret = <base64-encoded-256-bit-secret>
```

```ini
# /etc/swift/keymaster.conf
[keymaster]
encryption_root_secret = <base64-encoded-256-bit-key>
```

Generate a root secret:

```bash
python3 -c "import os, base64; print(base64.b64encode(os.urandom(32)).decode())"
```

```ini
[filter:encryption]
use = egg:swift#encryption
disable_encryption = false
```

### Barbican-backed keymaster (KMIP)

For production environments that require external key management, Swift supports Barbican as the key store:

```ini
[filter:keymaster]
use = egg:swift#barbican_keymaster
keystone_endpoint = https://keystone.example.com:5000
barbican_endpoint = https://barbican.example.com:9311
project_id = <swift-service-project-id>
username = swift
password = <swift-service-password>
user_domain_name = Default
project_domain_name = Default
```

### Key rotation

To rotate the root secret without re-encrypting all existing objects, Swift uses a **multi-key** approach. Old keys are retained in a key chain and used to decrypt objects encrypted with them. New objects are encrypted with the current key. This allows gradual migration.

---

## Understand the Account Reaper

When a Swift account is deleted (usually triggered by deleting the corresponding Keystone project and the account reaper daemon detecting the deletion), the **account reaper** (`swift-account-reaper`) asynchronously deletes all data associated with that account.

The reaper:

1. Lists all containers in the deleted account.
2. Lists all objects in each container.
3. Issues DELETE requests for each object (to all replica nodes).
4. Issues DELETE requests for each container.
5. Issues DELETE for the account itself.
6. Continues until the account is completely empty, then removes the account database.

The reaper is designed to be safe in the presence of failures — it is idempotent and can be interrupted and restarted. Deletion is eventually consistent: an account that is logically deleted continues to consume disk space until the reaper completes.

Configuration in `account-server.conf`:

```ini
[account-reaper]
concurrency = 25
interval = 3600           # run every hour
node_timeout = 10
conn_timeout = 0.5
delay_reaping = 0         # seconds to wait after account deletion before starting reap
reap_warn_after = 2592000 # 30 days; log warning if account not fully reaped
```

The reaper is triggered by the `swift-account-reaper` daemon. It should run on at least one account server node (or all of them, since it is safe to run concurrently).

---

## Tune Background Daemon Concurrency

All background daemons support a `concurrency` setting that controls how many simultaneous replication/audit/update jobs run. Higher concurrency increases throughput but also increases load on disk I/O and network.

Production starting points:

| Daemon | Typical concurrency |
|---|---|
| `object-replicator` | 1–4 (limited by disk I/O) |
| `object-auditor` | 1 (sequential I/O preferred) |
| `object-updater` | 4–8 |
| `object-reconstructor` (EC) | 2–4 |
| `container-replicator` | 4–8 |
| `container-updater` | 4 |
| `account-replicator` | 4 |
| `account-reaper` | 25 |

Monitor replication lag with:

```bash
swift-recon --replication --verbose
# Shows per-node replication time, oldest replication completion, and partition dispersion
```

```bash
swift-recon --diskusage --verbose
# Shows per-device disk usage
```

```bash
swift-recon --md5
# Verifies ring file MD5 matches across all nodes
```

```bash
swift-recon --time
# Checks clock skew between nodes (should be < 5 seconds)
```
