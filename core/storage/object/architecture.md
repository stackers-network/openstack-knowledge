# Swift Architecture

## Understand the Three-Level Namespace

Every piece of data in Swift lives at exactly one address: `account/container/object`.

- **Account** — the top-level namespace, mapped one-to-one to a Keystone project. The account name is `AUTH_<project-id>` by default. An account has its own SQLite database that lists all its containers.
- **Container** — a flat bucket inside an account. A container has its own SQLite database that lists all its objects. There is no container-within-container nesting; the hierarchy is simulated with object name prefixes and the `delimiter` query parameter.
- **Object** — the actual data blob, up to 5 GiB (single PUT) or unlimited with large object manifests. Objects have user-defined metadata stored as HTTP headers (`X-Object-Meta-*`).

All three layers are managed by separate server processes. They share no code path once a request leaves the proxy.

---

## Understand the Proxy Server

The proxy server (`swift-proxy-server`) is the sole entry point for all client requests. Every PUT, GET, DELETE, HEAD, and POST passes through exactly one proxy process. The proxy is stateless and horizontally scalable; any number of proxy nodes can run behind a load balancer.

### Request routing

When a request arrives, the proxy:

1. Validates the request URL, extracts account, container, and object names.
2. Runs the WSGI middleware pipeline (auth, rate limiting, caching, feature middleware — see below).
3. Hashes the object path (or account/container path) against the appropriate ring to determine which storage nodes hold the three (or N) replicas.
4. Issues concurrent backend requests to those storage nodes.
5. Returns the first successful response to the client (for GETs and HEADs) or waits for a quorum of successes (for PUTs and DELETEs).

A **quorum** for a write is `floor(replicas / 2) + 1`. With three replicas, two successful backend writes constitute a successful PUT. The proxy returns `201 Created` to the client even if the third replica is momentarily unavailable; the **replicator** catches up later.

### Auth middleware

Swift uses a WSGI middleware chain. Authentication can be handled by:

- **`keystoneauth`** — standard production middleware; validates tokens against Keystone, maps project-scoped tokens to Swift accounts.
- **`tempauth`** — built-in single-process auth for testing and all-in-one deployments; configured entirely in `proxy-server.conf`.
- **`swauth`** — legacy third-party auth system; rarely used in new deployments.

The auth middleware runs early in the pipeline. After authentication, the middleware sets `REMOTE_USER` and `HTTP_X_AUTH_TOKEN` so downstream middleware knows the identity.

### Rate limiting

The `ratelimit` middleware enforces per-account and per-container request rate limits. Limits are configured as requests per second (RPS) in `proxy-server.conf`:

```ini
[filter:ratelimit]
use = egg:swift#ratelimit
clock_accuracy = 1000
max_sleep_time_seconds = 60
log_sleep_time_seconds = 0
rate_buffer_seconds = 5
account_ratelimit = 0
# Per-container: container_ratelimit_<size_bucket> = <rps>
container_ratelimit_0 = 0
container_ratelimit_10 = 50
container_ratelimit_50 = 75
```

Requests that exceed the limit receive `498 Rate Limited`.

### Concurrency model

Each proxy worker is a separate OS process (controlled by `workers` in `[DEFAULT]`). Inside each worker, Swift uses **eventlet** (green threads) for non-blocking I/O. A single worker process handles hundreds of concurrent connections without blocking on network I/O.

---

## Understand the Account, Container, and Object Servers

### Account server

`swift-account-server` manages account-level metadata and the list of containers in each account. Each account has one SQLite database file stored on a data disk. The account server handles:

- `GET /v1/<account>` — list containers (reads from SQLite)
- `PUT /v1/<account>` — create account (create SQLite DB)
- `DELETE /v1/<account>` — mark account as deleted
- `HEAD /v1/<account>` — return account metadata
- `POST /v1/<account>` — update account metadata

The account server runs on the same nodes as object servers, typically on port **6202**.

### Container server

`swift-container-server` manages container-level metadata and the list of objects in each container. Each container has one SQLite database file. The container server handles:

- `GET /v1/<account>/<container>` — list objects
- `PUT /v1/<account>/<container>` — create container
- `DELETE /v1/<account>/<container>` — delete container (must be empty)
- `HEAD /v1/<account>/<container>` — return container metadata
- `POST /v1/<account>/<container>` — update container metadata (ACLs, versioning, etc.)

Container servers run on the same nodes as object servers, typically on port **6201**.

### Object server

`swift-object-server` manages the actual data blobs. Each object is stored as a file on a local XFS or ext4 filesystem. Objects are stored in a well-known path derived from the ring partition and object hash:

```
/srv/node/<device>/objects/<partition>/<suffix>/<object-hash>/<timestamp>.data
```

The `.data` extension holds the object body and its metadata in extended file attributes (xattrs). A `.meta` file stores POST-only metadata updates. A `.ts` (tombstone) file records deletions so that replication can propagate deletes to lagging replicas.

Object servers run on port **6200** by default.

### Auditor, replicator, and updater

Each server type has companion background daemons:

| Daemon | Purpose |
|---|---|
| `swift-object-auditor` | Reads every object file and verifies its md5 checksum; quarantines corrupted objects |
| `swift-object-replicator` | Pushes missing or out-of-date objects to partner nodes; uses rsync or the Swift replication protocol |
| `swift-object-updater` | Replays failed container database updates (e.g., object counts) that could not be sent when the container server was unavailable |
| `swift-container-auditor` | Verifies container database integrity |
| `swift-container-replicator` | Replicates container SQLite databases to partner nodes via HTTP merge |
| `swift-container-updater` | Updates account databases with container statistics |
| `swift-account-auditor` | Verifies account database integrity |
| `swift-account-replicator` | Replicates account SQLite databases to partner nodes |
| `swift-account-reaper` | Deletes all data belonging to deleted accounts |

---

## Understand the Ring Architecture

The **ring** is the core data structure that maps every account, container, and object to the set of storage devices that hold it. There are three separate rings: one for accounts, one for containers, and one for objects (or one per storage policy if multiple policies are configured).

### Consistent hashing

Swift uses a modified consistent hashing algorithm. Rather than hashing directly to devices, Swift introduces an intermediate layer: **partitions**.

The object path (or account/container path) is MD5-hashed, and the top N bits of the MD5 digest select a **partition**. Every partition is pre-assigned to a fixed set of devices by the ring. When a request arrives at the proxy, it hashes the path, looks up the partition, and finds the device list in the ring — O(1) lookup, no coordination required.

### Partition power

The number of partitions is `2^partition_power`. A partition power of **18** creates 262,144 partitions. This is enough granularity for millions of devices and billions of objects. Partition power is set at ring creation and cannot be changed without rebuilding the ring.

Guidelines:
- Small cluster (< 5 nodes, < 50 drives): partition power 10–12
- Medium cluster (10–50 nodes): partition power 14–16
- Large cluster (50+ nodes, 100+ drives): partition power 18–20
- Very large (1,000+ drives): partition power 22+

### Replicas

Each partition is assigned to exactly `replica_count` devices. With the default of 3, every partition maps to 3 devices. The proxy reads from or writes to all three concurrently.

### Zones and regions

Swift uses **zones** and **regions** to model failure domains:

- **Zone** — a failure domain within a region (e.g., a rack, a power domain, a network switch). Swift ensures each replica of a partition lands in a distinct zone. If a zone goes offline, only one replica is affected.
- **Region** — a geographic site or data center. Swift can span multiple regions. Each region contains one or more zones. Replicas are distributed across regions first, then zones.

The ring builder enforces replica placement: before placing a replica on a device in the same zone as an existing replica for the same partition, it exhausts all other zones. Same for regions.

### Devices and weight

Each device in the ring has a **weight** (a floating-point number proportional to its storage capacity). A 4 TB drive gets twice the weight of a 2 TB drive. The ring builder distributes partitions proportionally to weight, so larger drives hold more partitions.

A device entry contains:
- `region` — integer region ID
- `zone` — integer zone ID
- `ip` — IP address of the storage server
- `port` — port number (6202 for account, 6201 for container, 6200 for object)
- `device` — device name as mounted under `/srv/node/` (e.g., `sdb`, `sdc`)
- `weight` — proportional capacity
- `meta` — free-form annotation (rack number, drive model, etc.)

### Ring file format

The ring is serialized as a **gzipped Python pickle** (`.ring.gz`). The proxy and storage servers load it at startup and can reload it on SIGHUP or at a configurable interval. The ring file is typically a few hundred kilobytes even for large clusters because it stores only partition-to-device mappings.

The builder file (`.builder`) is a separate, larger file used only by `swift-ring-builder` to reconstruct and rebalance the ring. Builder files must be kept safe — losing a builder file makes future ring changes much harder.

### Handoff partitions

When a device is unavailable, the ring designates **handoff** devices — additional devices beyond the primary replica set that should temporarily store a partition's data. When the primary device comes back online, the replicator moves the data from handoff to primary and removes the handoff copy.

---

## Understand the Eventually Consistent Model

Swift is **AP** (Available, Partition-tolerant) in CAP terms. It prioritises availability and partition tolerance over strong consistency.

Key properties:

- **Writes** return success after a quorum of replicas acknowledge. A minority of replicas may be momentarily stale.
- **Reads** return the response from the first replica to respond. If replicas disagree, the proxy uses `X-Timestamp` metadata on each response to identify the most recent version and serve it (this is called **read repair** in some systems; Swift calls it the object server's **reconciliation** behaviour).
- **Deletes** are recorded as tombstone files (`.ts`), not immediate physical deletion, so that replication can propagate the delete to all replicas even if some were offline at delete time. Tombstones are eventually removed by the auditor after a configurable grace period (`reclaim_age`, default 604800 seconds / 7 days).
- **Eventual consistency** means that after a write to quorum, a client reading from a lagging replica may briefly see stale data. In practice the replicator's run interval (default: 30 seconds) bounds the staleness window.

---

## Understand Storage Policies

A **storage policy** is a named configuration that defines how objects are stored. Every container is assigned to exactly one policy at creation time; the policy cannot be changed afterward.

Two policy types exist:

### Replication policy (default)

Objects are stored as N complete copies, one per replica. This is the default and simplest model. It uses the standard object ring. Configuration in `/etc/swift/swift.conf`:

```ini
[storage-policy:0]
name = Policy-0
default = yes
policy_type = replication
```

### Erasure coding policy

Objects are split into data fragments and parity fragments using a configurable EC scheme. The fragments are distributed across storage nodes. Common schemes:

| Scheme | Data fragments | Parity fragments | Minimum nodes needed |
|---|---|---|---|
| `liberasurecode_rs_vand 10 4` | 10 | 4 | 14 |
| `liberasurecode_rs_vand 6 3` | 6 | 3 | 9 |
| `isa_l_rs_vand 10 4` | 10 | 4 | 14 |

The EC policy stores only `k/(k+m)` of the data size per node (e.g., 10/14 ≈ 71% vs. 100% for 3-replica), while tolerating up to `m` simultaneous node/drive failures.

```ini
[storage-policy:1]
name = ec-policy
policy_type = erasure_coding
ec_type = liberasurecode_rs_vand
ec_num_data_fragments = 10
ec_num_parity_fragments = 4
ec_object_segment_size = 1048576
```

Each storage policy has its own object ring. The ring file for policy 0 is `object.ring.gz`; for policy N (N > 0) it is `object-N.ring.gz`.

---

## Understand the WSGI Pipeline

Swift's proxy server is a WSGI application assembled from a chain of middleware. The pipeline is defined in `proxy-server.conf` under `[pipeline:main]`. A typical production pipeline:

```ini
[pipeline:main]
pipeline = catch_errors gatekeeper healthcheck proxy-logging cache listing_formats bulk tempurl ratelimit authtoken keystoneauth copy container_quotas account_quotas slo dlo proxy-logging proxy-server
```

Each entry is a filter (middleware) except the final `proxy-server`, which is the application itself. Filters are processed left to right. The most important filters:

| Filter | Purpose |
|---|---|
| `catch_errors` | Catches unhandled exceptions and returns 500; prevents stack traces leaking to clients |
| `gatekeeper` | Strips disallowed headers from incoming requests (prevents header injection) |
| `healthcheck` | Returns 200 OK for `/healthcheck` (load balancer probe endpoint) |
| `proxy-logging` | Logs access in combined log format (appears twice to log both before and after auth) |
| `cache` | Caches account and container metadata in Memcached |
| `listing_formats` | Enables JSON/XML/plain format negotiation for container listings |
| `bulk` | Handles bulk delete and tar archive extract requests |
| `tempurl` | Validates and serves temporary URL requests |
| `ratelimit` | Enforces per-account, per-container request rate limits |
| `authtoken` | Validates Keystone tokens (keystonemiddleware); caches validation results |
| `keystoneauth` | Maps validated Keystone identity to Swift ACLs and roles |
| `copy` | Implements server-side object copy (COPY method and PUT with `X-Copy-From`) |
| `container_quotas` | Enforces per-container byte and object count quotas |
| `account_quotas` | Enforces per-account byte quotas |
| `slo` | Static Large Object: handles manifest PUTs and transparent multi-segment GETs |
| `dlo` | Dynamic Large Object: assembles segments by prefix at GET time |
| `proxy-server` | Core proxy: ring lookup, backend I/O, response assembly |

The ordering is significant. `authtoken`/`keystoneauth` must come before any middleware that requires an authenticated identity. `slo`/`dlo` must come before `proxy-server`. `tempurl` must come before `authtoken` so that temp URL requests can bypass normal token auth.
