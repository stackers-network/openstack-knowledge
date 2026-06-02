# Octavia Architecture

## Overview

Octavia implements load balancing by provisioning dedicated virtual machines called *amphorae*. Each amphora runs a hardened Linux image with HAProxy managing the actual traffic. Octavia itself orchestrates the lifecycle of these VMs through four main service processes and a pluggable provider driver framework.

```
                  ┌─────────────────────────────────────────────┐
                  │              Octavia Services                │
                  │                                             │
  REST API ──────►│  octavia-api                                │
                  │       │ oslo.messaging (RabbitMQ)           │
                  │       ▼                                      │
                  │  octavia-worker  ──── Nova/Neutron/Glance   │
                  │                                             │
                  │  health-manager  ◄─── UDP heartbeats        │
                  │       │ (update DB)                         │
                  │  housekeeping    ──── spare pool, cleanup   │
                  └─────────────────────────────────────────────┘
                              │ management network
                    ┌─────────┴──────────┐
                    │      Amphora VM    │
                    │   (HAProxy + agent)│
                    └────────────────────┘
                         │ VIP network
                    Client traffic in/out
```

## Service Processes

### octavia-api

The WSGI front-end for the Octavia v2 REST API. Responsibilities:

- Validates and authenticates incoming API requests (Keystone tokens)
- Enforces policy (oslo.policy)
- Persists resource state to the Octavia database
- Publishes Taskflow jobs to octavia-worker via RabbitMQ cast
- Returns `202 Accepted` for asynchronous operations; the caller must poll until `provisioning_status` reaches `ACTIVE` or `ERROR`

The API is versioned at `/v2/lbaas/`. The OpenStack SDK and `python-openstackclient` wrap this endpoint.

### octavia-worker

Executes the actual lifecycle operations on amphorae and HAProxy. Responsibilities:

- Subscribes to the RabbitMQ task queue
- Runs Taskflow linear flows for each operation (create LB, create listener, failover, etc.)
- Calls the Nova API to boot/delete amphora VMs
- Calls the Neutron API to allocate VIP ports and management ports
- Connects to the amphora REST agent over the management network (HTTPS) to configure HAProxy
- Updates `provisioning_status` in the database as tasks complete or fail

Multiple octavia-worker processes can run for horizontal scaling. Each picks up jobs from the shared queue.

### health-manager

Monitors the liveness of all running amphorae. Responsibilities:

- Listens on a UDP port on the management network for heartbeat packets from amphorae
- Each amphora sends a UDP heartbeat every `heartbeat_interval` seconds (default 10 s) containing its current status and statistics
- health-manager updates the database with per-listener and per-member operational status
- If no heartbeat is received within `heartbeat_timeout` seconds (default 60 s), health-manager marks the amphora as `ERROR` and triggers an automatic failover
- health-manager is stateless; multiple instances can run behind a load balancer for HA, but each amphora's heartbeats are sticky to one instance (consistent hashing on amphora UUID)

### housekeeping

Performs periodic maintenance tasks. Responsibilities:

- **Spare pool management**: pre-creates amphora VMs in the `READY` state so that new load balancer requests can be served quickly without waiting for Nova to boot a fresh VM
- **Expired amphora cleanup**: deletes amphorae that are in `DELETED` state and older than `delete_amphora_later_sec` seconds
- **Certificate rotation**: rotates the client/server TLS certificates used between octavia-worker and the amphora REST agent before they expire
- **Expired flow cleanup**: removes stale Taskflow database entries

## Amphora Design

Each amphora is a Nova VM running a purpose-built image built with `diskimage-builder` using the `amphora-image` element. Inside the VM:

```
Amphora VM
├── amphora-agent         (Python REST API on port 9443 over TLS)
│     ├── Accepts HTTPS calls from octavia-worker
│     ├── Writes HAProxy config files
│     ├── Manages HAProxy process lifecycle
│     └── Sends UDP heartbeat to health-manager
├── HAProxy               (actual packet forwarder)
│     ├── One HAProxy process per load balancer (listener namespace)
│     └── Config written to /var/lib/octavia/<lb-uuid>/haproxy.cfg
└── keepalived            (VRRP for active-standby HA pairs)
```

### Network Interfaces

Each amphora has two network interfaces:

| Interface | Purpose | Connected To |
|---|---|---|
| `eth0` (management) | Communication with octavia-worker and health-manager; amphora REST agent and UDP heartbeat | Octavia management network (operator-managed, not tenant) |
| `eth1` (VIP / tenant) | Carries load-balanced traffic; HAProxy binds listeners here | Neutron VIP port on the tenant subnet |

For active-standby HA amphorae a third interface (`eth2`) may carry the VRRP peer link.

### Management Network

The management network is a dedicated Neutron network created by the operator during Octavia deployment. It is not exposed to tenants. The octavia-worker and health-manager connect to amphorae exclusively via this network using:

- **HTTPS** (TCP 9443): octavia-worker → amphora REST agent (mutual TLS; certs rotated by housekeeping)
- **UDP heartbeat** (configurable port, default 5555): amphora → health-manager

The management network should use a separate physical interface or a dedicated VLAN to ensure that health-check traffic is isolated from tenant data.

### Active-Standby HA

When `--vip-subnet-id` is combined with an HA topology (the default `SINGLE` topology can be overridden to `ACTIVE_STANDBY`):

- Two amphora VMs are provisioned
- keepalived runs VRRP inside both amphorae
- The active amphora holds the VIP; the standby monitors it
- If the active amphora fails, the standby takes the VIP within the VRRP `advert_int` interval (default 1 s)
- health-manager also triggers a full amphora failover if heartbeat is lost

### Active-Active HA

When topology is `ACTIVE_ACTIVE` (requires OVN-Octavia or newer octavia-driver-lib support):

- Multiple active amphorae share traffic via the VIP
- Requires a provider driver that supports it (e.g., the OVN provider)
- The amphora provider implements active-active via conntrack sync and ECMP

## Provider Driver Framework

Octavia decouples the API surface from the back-end implementation through a provider driver interface (`octavia_lib.api.drivers`). Operators can register multiple providers and users can choose which one to use when creating a load balancer.

### Amphora Provider (default)

- Ships with Octavia
- Provisions a full HAProxy VM per load balancer
- Supports all LBaaS features: HTTP/HTTPS/TCP/UDP listeners, L7 policies, health monitors, TLS termination, connection limits
- Best for production workloads requiring full feature parity

### OVN Provider

- Implemented in the `ovn-octavia-provider` project
- Delegates load balancing to OVN's native load balancer (kernel-space conntrack DNAT)
- No amphora VMs are created; lower overhead
- Limitations: no L7 policies, no TLS termination, no UDP health monitors, limited algorithm support
- Best for lightweight internal TCP/HTTP load balancing in deployments already using ML2/OVN

### Registering Providers

```ini
# /etc/octavia/octavia.conf
[api_settings]
enabled_provider_drivers = amphora:The Octavia Amphora driver.,ovn:OVN provider driver.
default_provider_driver = amphora
```

List available providers:

```bash
openstack loadbalancer provider list
```

Specify at creation time:

```bash
openstack loadbalancer create \
  --name ovn-lb \
  --vip-subnet-id private-subnet \
  --provider ovn
```

## Spare Pool

The spare pool keeps pre-booted amphora VMs warm so that new load balancer creation is fast (seconds rather than minutes). housekeeping continuously replenishes it.

```ini
[house_keeping]
spare_amphora_pool_size = 2          # number of warm spare amphorae to maintain
```

When a load balancer is created, octavia-worker first tries to claim a spare amphora. If no spare is available, it falls back to booting a new Nova VM. After an amphora is deleted, housekeeping eventually creates a replacement spare.

## Anti-Affinity and Placement

For active-standby HA load balancers, Octavia uses Nova server groups with `anti-affinity` to place the two amphora VMs on different compute nodes. This is controlled by:

```ini
[nova]
enable_anti_affinity = True
anti_affinity_policy = anti-affinity   # or soft-anti-affinity
```

When `enable_anti_affinity = True`, Octavia creates a Nova server group for each HA pair and passes `scheduler_hints` when booting each amphora so the scheduler places them on distinct hypervisors.

## Flavors and Flavor Profiles

Octavia flavors allow operators to offer different load balancer tiers (e.g., small/medium/large) with different amphora Nova flavors, topologies, or provider drivers.

```
Flavor Profile
 └── defines: compute_flavor (Nova flavor ID), amp_topology (SINGLE/ACTIVE_STANDBY),
              loadbalancer_topology, provider_name

Flavor
 └── references: a FlavorProfile
 └── exposed to: tenants (via loadbalancer create --flavor)
```

```bash
# Create a flavor profile using a larger Nova flavor
openstack loadbalancer flavorprofile create \
  --name large-profile \
  --provider amphora \
  --flavor-data '{"compute_flavor": "m1.large", "loadbalancer_topology": "ACTIVE_STANDBY"}'

# Create a user-facing flavor
openstack loadbalancer flavor create \
  --name large \
  --flavorprofile large-profile \
  --description "HA load balancer on m1.large"

# Use the flavor
openstack loadbalancer create \
  --name ha-lb \
  --vip-subnet-id private-subnet \
  --flavor large
```

## Database Schema Overview

Octavia maintains its own database (separate from Neutron). Key tables:

| Table | Contents |
|---|---|
| `load_balancer` | VIP address, subnet, project, provisioning_status, operating_status |
| `listener` | Protocol, port, connection_limit, default_pool, TLS ref |
| `pool` | Algorithm, protocol, health_monitor ref |
| `member` | Address, protocol_port, weight, backup, operating_status |
| `health_monitor` | Type, delay, timeout, max_retries, HTTP path/codes |
| `amphora` | Nova instance ID, management IP, status, role (MASTER/BACKUP/STANDALONE) |
| `l7policy` / `l7rule` | L7 redirect and content-routing rules |

## Operating Status vs Provisioning Status

Octavia resources carry two independent status fields:

| Field | Meaning | Values |
|---|---|---|
| `provisioning_status` | Control-plane operation state | `ACTIVE`, `PENDING_CREATE`, `PENDING_UPDATE`, `PENDING_DELETE`, `DELETED`, `ERROR` |
| `operating_status` | Data-plane health | `ONLINE`, `DRAINING`, `OFFLINE`, `DEGRADED`, `ERROR`, `NO_MONITOR` |

A load balancer is healthy when `provisioning_status=ACTIVE` and `operating_status=ONLINE`. A member shows `ONLINE` only when the health monitor confirms it is passing checks.
