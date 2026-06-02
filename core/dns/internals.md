# Designate Internals

## Backend Driver Framework

Designate uses a pluggable backend driver model to support multiple DNS server implementations. The driver interface (`designate.backend.base.Backend`) defines the operations that each driver must implement:

```python
class Backend:
    def create_zone(self, context, zone): ...
    def update_zone(self, context, zone): ...
    def delete_zone(self, context, zone): ...
```

The driver is invoked by designate-worker when it receives a task from designate-central via RabbitMQ.

### Backend Types

| Backend | Driver Module | Mechanism |
|---|---|---|
| BIND9 (agent) | `designate.backend.agent_backend.impl_bind9` | Connects to a `designate-agent` sidecar on the BIND9 host via RPC; the agent runs `rndc` commands locally |
| BIND9 (direct) | `designate.backend.impl_bind9` | designate-worker sends RNDC commands directly to BIND9 (requires network access to RNDC port 953) |
| PowerDNS | `designate.backend.impl_pdns4` | Calls the PowerDNS authoritative API (HTTP) to create/delete zones and records |
| Infoblox | `designate.backend.impl_infoblox` | Calls the Infoblox NIOS REST API (WAPI) |
| Akamai | `designate.backend.impl_akamai` | Calls the Akamai Fast DNS REST API |
| NOOP | `designate.backend.impl_noop` | Used for testing; does nothing |

### BIND9 Agent Backend (Most Common)

In most deployments, Designate does not manage BIND9 directly. Instead, a `designate-agent` process runs on each BIND9 nameserver host. The flow is:

```
designate-worker
    │ RPC (oslo.messaging)
    ▼
designate-agent (on BIND9 host)
    │ subprocess
    ▼
rndc addzone / modzone / delzone  →  BIND9
```

The `designate-agent` translates Designate's zone creation/deletion tasks into `rndc` commands:

- `rndc addzone example.com. '{type slave; masters { <mdns-ip> port 5354; }; };'`
- `rndc delzone example.com.`
- `rndc modzone example.com. ...`

When BIND9 receives the `addzone` command, it contacts `designate-mdns` to pull the zone data via AXFR.

### BIND9 Direct Backend (Simpler, Less Scalable)

For single-node setups, designate-worker can manage BIND9 directly:

```ini
# /etc/designate/designate.conf
[service:worker]
enabled_backends = bind9

[backend:bind9]
type = bind9

[backend:bind9:bind9]
rndc_host = 127.0.0.1
rndc_port = 953
rndc_config_file = /etc/bind/rndc.conf
rndc_key_file = /etc/bind/rndc.key
```

### PowerDNS Backend

PowerDNS is managed via its authoritative API. Designate calls:

- `POST /api/v1/servers/localhost/zones` to create a zone (as a slave that transfers from mdns)
- `DELETE /api/v1/servers/localhost/zones/<zone_id>` to delete a zone
- PowerDNS then fetches zone data from `designate-mdns` via AXFR

```ini
[backend:pdns4]
type = pdns4

[backend:pdns4:pdns4]
host = 127.0.0.1
port = 8081
api_endpoint = http://127.0.0.1:8081
api_token = <pdns_api_key>
```

### Infoblox Backend

Infoblox integration uses the NIOS WAPI. Designate creates a delegated zone object in Infoblox that points to Designate's mdns as the authoritative source:

```ini
[backend:infoblox]
type = infoblox

[backend:infoblox:infoblox]
wapi_url = https://infoblox.example.com/wapi/v2.1
username = admin
password = <password>
ns_group = designate-ns-group
```

## Pool Configuration

Pools are defined in `pools.yaml` and loaded into the database with `designate-manage pool update`. A complete pool configuration:

```yaml
---
- name: default
  description: Default BIND9 pool
  attributes: {}

  ns_records:
    - hostname: ns1.example.com.
      priority: 1
    - hostname: ns2.example.com.
      priority: 2

  nameservers:
    - host: 198.51.100.1
      port: 53
    - host: 198.51.100.2
      port: 53

  targets:
    - type: bind9
      description: Primary BIND9 server
      masters:
        - host: 192.168.1.10        # designate-mdns IP
          port: 5354
      options:
        host: 198.51.100.1
        port: 953
        rndc_key_file: /etc/designate/rndc.key
        rndc_config_file: /etc/designate/rndc.conf
        clean_zonefile: True

    - type: bind9
      description: Secondary BIND9 server
      masters:
        - host: 192.168.1.10
          port: 5354
      options:
        host: 198.51.100.2
        port: 953
        rndc_key_file: /etc/designate/rndc.key
        rndc_config_file: /etc/designate/rndc.conf

  also_notifies:
    - host: 198.51.100.50
      port: 53
```

Load after editing:

```bash
designate-manage pool update --file /etc/designate/pools.yaml
designate-manage pool list
```

### Multi-Pool Setup

Operators can define multiple pools for different purposes:

```yaml
- name: internal
  description: Internal zones served by local BIND9
  ...

- name: external
  description: External zones served by PowerDNS + Akamai
  ...
```

Pool selection for a zone is controlled by a scheduler configured in `designate.conf`:

```ini
[scheduler:pool_manager_scheduler]
filters = random, in_doubt_default_pool
```

Zones can also be explicitly pinned to a pool at creation time:

```bash
openstack zone create \
  --email admin@example.com \
  --attributes pool_id:<external-pool-uuid> \
  external.example.com.
```

## Serial Number Management

The SOA serial number is a 32-bit unsigned integer that must increase with every zone change. Designate manages this automatically using the configured `serial_generator`.

### UNIX Timestamp Strategy (Default)

```ini
[service:central]
serial_generator = timestamp
```

- Serial = current UNIX timestamp at time of change
- Always increases
- Gap between serials reflects the time between changes
- Can produce large jumps if the zone is infrequently modified

### Datestamp Strategy

```ini
[service:central]
serial_generator = datestamp
```

- Serial format: `YYYYMMDDnn` (date + 2-digit sequence)
- Familiar to DNS operators used to manual BIND9 serial management
- Supports up to 99 changes per day

### Increment Strategy

```ini
[service:central]
serial_generator = increment
```

- Serial starts at 1 and increments by 1 with each change
- Smallest gaps but doesn't encode timestamp information

## SOA Record Auto-Generation

Designate auto-generates the SOA record for every primary zone and updates it whenever zone configuration changes. The SOA record is constructed from:

```
example.com. 300 IN SOA ns1.example.com. admin.example.com. (
    2024030101  ; serial
    3600        ; refresh (seconds before secondary checks for updates)
    600         ; retry (seconds before secondary retries after failed refresh)
    86400       ; expire (seconds secondary may continue serving if primary unreachable)
    300         ; minimum TTL (negative caching TTL)
)
```

These values come from pool configuration (`ns_records` for the primary NS) and `designate.conf`:

```ini
[service:central]
default_ttl = 300
default_soa_refresh = 3600
default_soa_retry = 600
default_soa_expire = 86400
default_soa_minimum = 300
```

The SOA record is read-only via the Designate API — it cannot be created or deleted by API users. Attempting to create a recordset with type `SOA` returns a 400 error.

## Mini-DNS (designate-mdns) — AXFR Zone Transfer

`designate-mdns` is a purpose-built DNS server that serves zone data directly from the Designate database. It is not a general-purpose DNS server — it only handles:

1. **NOTIFY** (outbound): sends DNS NOTIFY messages to backend nameservers when a zone changes
2. **AXFR/IXFR** (inbound requests): responds to zone transfer requests from backend nameservers

### AXFR Flow

```
1. designate-central receives API call to update record set
2. designate-central increments serial, writes new recordset to DB
3. designate-central dispatches UPDATE_ZONE task to designate-worker
4. designate-worker calls backend driver (e.g., BIND9 agent)
5. Backend driver sends RNDC or API call to nameserver
6. Nameserver receives NOTIFY from designate-mdns (step 4a, parallel)
7. Nameserver sends AXFR request to designate-mdns
8. designate-mdns reads zone from DB, generates RFC 1035 wire-format AXFR response
9. Nameserver stores the zone and begins serving it
```

designate-mdns config:

```ini
[service:mdns]
host = 192.168.1.10      # Management network IP
port = 5354
tcp_backlog = 100
tcp_recv_timeout = 0.5
```

TSIG key configuration for authenticating AXFR:

```ini
[backend:bind9:tsig_key:default]
name = designate-tsig
algorithm = hmac-sha256
secret = <base64-encoded-key>
```

### NOTIFY Flow

When a zone is updated, designate-mdns sends a DNS NOTIFY to each nameserver in the pool's `nameservers` list and `also_notifies` list. The NOTIFY carries the new serial number. Standard-compliant nameservers respond with `NOTIMPL` (they will pull via AXFR) or `NOERROR` (they checked the serial and decided whether to pull).

## designate-sink — Notification Handling

designate-sink subscribes to oslo.messaging notification events published by other OpenStack services. This enables automatic DNS record lifecycle management.

### Notification Flow

```
Nova/Neutron ──► oslo.messaging (notifications.info topic)
                        │
                        ▼
             designate-sink (consumes)
                        │
                        ▼
           evaluate handler rules
                        │
           ┌────────────┴────────────┐
           ▼                         ▼
   create recordset           delete recordset
   (via designate-central)    (via designate-central)
```

### Neutron Floating IP Handler

```ini
[handler:neutron_floatingip]
event_types = floatingip.create.end, floatingip.update.end, floatingip.delete.end
control_exchange = neutron
zone_id = <reverse-zone-uuid>
format_pattern = %(octet0)s-%(octet1)s-%(octet2)s-%(octet3)s.cloud.example.com.
```

When `floatingip.create.end` fires with IP `203.0.113.42`:

1. Sink handler parses `octet0=203`, `octet1=0`, `octet2=113`, `octet3=42`
2. Generates PTR name `42.113.0.203.in-addr.arpa.`
3. Generates PTR target `203-0-113-42.cloud.example.com.`
4. Creates PTR record in the reverse zone via designate-central

When `floatingip.delete.end` fires:
- Sink finds and deletes the corresponding PTR record

### Nova Instance Handler

```ini
[handler:nova_fixed]
event_types = compute.instance.create.end, compute.instance.delete.end
control_exchange = nova
zone_id = <forward-zone-uuid>
format_pattern = %(hostname)s.nova.cloud.example.com.
```

When an instance is created with hostname `web-01`, the sink creates:
- `web-01.nova.cloud.example.com. A <instance-fixed-ip>`

## Database Schema Overview

Key Designate tables:

| Table | Contents |
|---|---|
| `zones` | Zone name, email, TTL, serial, status, pool_id, project_id |
| `recordsets` | Zone ID, name, type, TTL, status |
| `records` | RecordSet ID, data (the actual record value), hash |
| `pool_attributes` | Key/value attributes for pool scheduler |
| `pool_nameservers` | Publicly advertised NS records per pool |
| `pool_targets` | Backend target configs (type, host, options) |
| `tsigkeys` | TSIG key name, algorithm, secret, scope |
| `zone_transfer_requests` | Pending zone transfers (source zone, key, target project) |
| `zone_transfer_accepts` | Completed zone transfer records |

## Periodic Tasks (designate-producer)

designate-producer runs the following periodic tasks:

| Task | Interval | Purpose |
|---|---|---|
| `zone_purge` | Configurable (default 3600 s) | Hard-delete zones in `DELETED` state older than `zone_purge_after` |
| `secondary_zone_refresh` | Configurable (default 300 s) | Check serial of secondary zones against upstream primaries; trigger AXFR if stale |
| `delayed_notify` | 5 s | Re-send NOTIFY to backends that haven't confirmed receipt |
| `worker_recovery` | 120 s | Re-queue tasks stuck in `PENDING` state longer than `worker_recovery_timeout` |

```ini
[service:producer]
enabled_tasks = zone_purge, secondary_zone_refresh, delayed_notify, worker_recovery
threads = 10
```

## Configuration Reference

Key `designate.conf` sections:

```ini
[service:api]
listen = 0.0.0.0:9001
api_base_uri = http://designate-api:9001/
auth_strategy = keystone
enable_api_v2 = True

[service:central]
backend_driver = agent           # or pdns4, infoblox, etc.
scheduler_filters = random, in_doubt_default_pool
serial_generator = timestamp
default_ttl = 300

[service:worker]
enabled_backends = bind9
notify = True
workers = 10

[service:mdns]
host = 192.168.1.10
port = 5354

[service:sink]
enabled_notification_handlers = neutron_floatingip

[handler:neutron_floatingip]
zone_id = <uuid>
format_pattern = %(octet0)s-%(octet1)s-%(octet2)s-%(octet3)s.example.com.
```
