# Designate Architecture

## Overview

Designate is composed of six service processes that cover the API surface, business logic, DNS backend operations, periodic maintenance, internal DNS, and external event consumption. All services communicate through oslo.messaging (RabbitMQ) and share a common database.

```
                         ┌──────────────────────────────────────────────┐
                         │             Designate Services                │
                         │                                              │
  REST clients ─────────►│  designate-api                               │
                         │       │ RPC                                  │
                         │       ▼                                      │
                         │  designate-central  (business logic, DB)    │
                         │       │ RPC                                  │
                         │       ▼                                      │
                         │  designate-worker   (DNS backend ops)       │
                         │       │                                      │
                         │  designate-producer (periodic tasks)        │
                         │       │ internal zone transfers              │
                         │  designate-mdns     (mini-DNS / AXFR)       │
                         │                                              │
                         │  designate-sink     (Neutron notifications) │
                         └──────────────────────────────────────────────┘
                                         │ AXFR / IXFR
                              ┌──────────┴──────────┐
                              │    DNS Backends      │
                              │  BIND9 / PowerDNS   │
                              │  Infoblox / Akamai  │
                              └─────────────────────┘
```

## Service Processes

### designate-api

The WSGI front-end for the Designate v2 REST API. Responsibilities:

- Authenticates and authorizes requests using Keystone tokens and oslo.policy
- Validates request bodies (zone name format, record data format, TTL ranges)
- Persists approved resource records to the database via designate-central (RPC call)
- Returns synchronous responses for read operations; zone mutations return `202 Accepted` with a status field that moves from `PENDING` to `ACTIVE` or `ERROR` asynchronously
- Handles API versioning at `/v2/`

Multiple designate-api workers can run behind a load balancer. All workers are stateless.

### designate-central

The business logic core of Designate. Responsibilities:

- Receives RPC calls from designate-api
- Enforces business rules: zone ownership, record set validation, quota checking, duplicate detection
- Writes zone and record set state to the database
- Manages serial number increments (UNIX timestamp or incremental, depending on config)
- Dispatches backend operation tasks to designate-worker via RPC
- Handles zone transfers, zone imports, and secondary zone polling

designate-central is the authoritative owner of the Designate database. Only central writes to zone and record set tables.

### designate-worker

Executes operations against DNS backend servers. Responsibilities:

- Receives tasks from designate-central via RabbitMQ (create zone, update recordset, delete zone, etc.)
- Calls the pool manager to select which pool(s) handle the zone
- Calls the backend driver for the selected pool to apply changes (via RNDC for BIND9, via API for PowerDNS, etc.)
- Verifies that the backend reflects the intended state (sends a DNS query to confirm record propagation)
- Reports success or failure back to designate-central, which updates `status` accordingly
- Retries failed operations with exponential back-off up to `max_retries`

### designate-producer

Runs periodic tasks against the Designate database. Responsibilities:

- **Zone purging**: deletes zones that are in `DELETED` state and older than `zone_purge_after` seconds
- **Secondary zone refresh**: triggers polls to the primary nameserver for secondary zones to check for updated serials
- **Delayed notify**: re-sends DNS NOTIFY messages to backends that missed them (e.g., after a backend restart)
- **Worker periodic recovery**: re-queues tasks that stalled in `PENDING` state

designate-producer runs as a single instance or as an active-active pair with leader election via tooz.

### designate-mdns (Mini-DNS)

Implements a minimal DNS server that acts as the authoritative source for AXFR zone transfers to DNS backends. Responsibilities:

- Responds to DNS `NOTIFY` messages from Designate and sends `NOTIFY` to backends
- Serves authoritative `AXFR` (full zone transfer) and `IXFR` (incremental) responses to backend nameservers
- Reads zone data directly from the Designate database (not from BIND9 or PowerDNS)
- Allows DNS backends to pull zone data from Designate on-demand, rather than Designate pushing individual records
- Listens on a configurable UDP/TCP port (default 5354) on the management network

The mini-DNS approach means Designate can support any DNS backend that understands standard AXFR zone transfers, regardless of backend-specific APIs.

### designate-sink

Consumes event notifications from other OpenStack services and creates or deletes DNS records automatically. Responsibilities:

- Subscribes to oslo.messaging notification topics (e.g., `notifications.info`)
- Handles Neutron events:
  - `floatingip.create.end` → create a PTR record for the floating IP (if `auto_reverse` is enabled)
  - `floatingip.delete.end` → delete the PTR record
  - `floatingip.update.end` → update PTR record when association changes
  - `port.create.end` → create an A record for the instance port (if `format_pattern` is configured)
  - `port.delete.end` → delete the corresponding A record
- Handlers are configurable; operators can write custom sink handlers for other notification types

## Zone Model

### Primary Zones

A primary zone is one that Designate owns authoritatively. Record sets are managed through the Designate API and propagated to DNS backends.

```
Zone (example.com.)
 ├── type: PRIMARY
 ├── email: admin@example.com
 ├── ttl: 300 (default TTL for records without explicit TTL)
 ├── serial: 2024030101 (monotonically increasing)
 ├── status: ACTIVE | PENDING | ERROR
 └── Record Sets
       ├── SOA  (auto-generated by Designate)
       ├── NS   (auto-generated, reflects pool nameservers)
       ├── A    www → 203.0.113.10
       ├── MX   "" → "10 mail.example.com."
       └── TXT  "" → "v=spf1 include:_spf.example.com ~all"
```

### Secondary Zones

A secondary zone is one where Designate follows a primary nameserver outside Designate. Designate polls the external primary for serial changes and pulls updated zone data via AXFR.

```bash
openstack zone create \
  --email secondary@example.com \
  --type secondary \
  --masters 198.51.100.1 \
  external-zone.example.com.
```

designate-producer's secondary zone refresh task checks the external primary's SOA serial periodically. If the serial has advanced, designate-central triggers an AXFR from the external primary to pull the new zone data.

## Record Set Model

A record set groups all records of the same type at the same name within a zone:

```
RecordSet
 ├── zone_id → Zone
 ├── name: "www.example.com."
 ├── type: A
 ├── ttl: 300
 ├── records: ["203.0.113.10", "203.0.113.11"]   (multiple records = DNS round-robin)
 └── status: ACTIVE | PENDING | ERROR
```

Each record in the `records` list is served as a separate RR in DNS responses. Designate validates record data format per RFC before accepting it.

## DNS Pool Architecture

A pool is a named group of DNS backend servers that jointly serve a set of zones. Pools allow operators to:

- Route different zones to different backends (e.g., internal zones to BIND9, external zones to PowerDNS)
- Scale DNS serving by adding nameservers to a pool
- Offer different SLA tiers (e.g., a `premium` pool with Akamai edge DNS)

```
Pool
 ├── id / name: "default" (UUID)
 ├── description
 ├── nameservers: [ns1.example.com., ns2.example.com.]  (publicly advertised NS records)
 ├── targets:                                            (backends designate-worker pushes to)
 │     ├── Target: bind9-primary
 │     │     ├── type: bind9
 │     │     └── options: host, port, rndc_key, rndc_config_file
 │     └── Target: bind9-secondary
 │           ├── type: bind9
 │           └── options: host, port, also_notifies
 └── also_notifies: [198.51.100.50]   (extra NOTIFY recipients)
```

Pool configuration is managed in `pools.yaml` and loaded with:

```bash
designate-manage pool update --file /etc/designate/pools.yaml
```

### Zone-to-Pool Assignment

By default, all zones are assigned to the `default` pool. Operators can assign zones to specific pools using zone attributes or pool selectors (configured in `designate.conf`).

## Neutron Integration

Designate integrates with Neutron through two mechanisms:

### 1. Port/Instance DNS via Neutron (forward DNS)

When Neutron's `dns_domain` extension is enabled, each Neutron network can have a `dns_domain` attribute. When an instance port is created on a network with `dns_domain` set, Neutron calls Designate to create an A record for that port using:

```
<port_dns_name>.<network_dns_domain>
```

Enable in `neutron.conf`:

```ini
[DEFAULT]
dns_domain = openstack.example.com.

[designate]
url = http://designate-api:9001/v2
admin_auth_url = http://keystone:5000/v3
admin_username = neutron
admin_password = <password>
admin_tenant_name = service
```

### 2. Floating IP PTR via designate-sink

When a floating IP is created, designate-sink receives the Neutron notification and:

1. Determines the reverse zone for the floating IP's address (e.g., `203.0.113.10` → `113.0.203.in-addr.arpa.`)
2. Creates a PTR record: `10.113.0.203.in-addr.arpa. PTR web-1.example.com.`

The sink handler uses a configurable `format_pattern` to generate the PTR record target name:

```ini
[handler:neutron_floatingip]
zone_id = <reverse-zone-uuid>
format_pattern = %(octet0)s-%(octet1)s-%(octet2)s-%(octet3)s.example.com.
```

## Serial Number Management

Designate manages SOA serial numbers automatically. Two strategies are configurable:

| Strategy | Behaviour | Config value |
|---|---|---|
| `increment` | Serial starts at 1 and increases by 1 with each change | `serial_generator = increment` |
| `datestamp` | Serial is formatted as `YYYYMMDDnn` (date + daily counter) | `serial_generator = datestamp` |
| `timestamp` (default) | Serial is the UNIX timestamp at time of last change | `serial_generator = timestamp` |

The UNIX timestamp strategy ensures that the serial always increases even when zones are imported from external sources, but may skip multiple increments on rapid changes.

## API Versioning and Formats

The Designate v2 API uses:

- Trailing dot on zone names: `example.com.` (FQDN format required)
- Trailing dot on record data for host names: `mail.example.com.`
- Zone names and record names are case-insensitive but stored lowercase
- Records within a record set are a list: `"records": ["203.0.113.10"]`
