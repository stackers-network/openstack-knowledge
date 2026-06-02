# DNS — Designate

Designate is the OpenStack DNS as a Service (DNSaaS) component. It exposes a REST API for creating and managing DNS zones and record sets, and propagates changes to one or more back-end DNS servers (BIND9, PowerDNS, Infoblox, Akamai, and others) through a pool-based driver framework. Designate integrates with Neutron to automatically create PTR records for floating IPs and forward DNS records for instance ports.

## When to Read This Skill

- Deploying Designate and configuring the first DNS pool and backend driver
- Creating zones and record sets via CLI or API
- Configuring reverse DNS (PTR records) for floating IPs and fixed IPs
- Setting up Neutron integration for automatic DNS record creation
- Implementing zone transfers between projects or to external nameservers
- Troubleshooting record propagation to BIND9 or PowerDNS backends
- Configuring TSIG keys for zone transfer security
- Understanding the mini-DNS (mdns) internal zone transfer mechanism
- Managing DNS pools and backend selection

## Sub-Files

| File | What It Covers |
|---|---|
| [architecture.md](architecture.md) | designate-api, designate-central, designate-worker, designate-producer, designate-mdns, designate-sink; zone model (primary/secondary); Neutron integration; pool architecture |
| [operations.md](operations.md) | CLI and API: zones, record sets (A, AAAA, MX, CNAME, TXT, SRV, PTR), zone transfers, zone import/export, TSIG keys, quotas |
| [internals.md](internals.md) | Backend driver framework (BIND9 agent, PowerDNS, Infoblox, Akamai); pool configuration; serial number management; SOA records; sink notification handling; mini-DNS AXFR |

## Quick Reference

```bash
# Create a zone
openstack zone create --email admin@example.com example.com.

# Add an A record
openstack recordset create --type A --record 203.0.113.10 example.com. www

# Add an MX record
openstack recordset create \
  --type MX \
  --record "10 mail.example.com." \
  example.com. ""

# Add an AAAA record
openstack recordset create \
  --type AAAA \
  --record "2001:db8::1" \
  example.com. ipv6

# List record sets in a zone
openstack recordset list example.com.

# Show zone status and serial
openstack zone show example.com.
```

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| Keystone | Authentication and service catalog | All Designate API calls require Keystone tokens |
| MariaDB / PostgreSQL | Designate database | Stores zones, recordsets, pools, backend state |
| RabbitMQ | RPC between Designate services | oslo.messaging; used for worker tasks and sink events |
| BIND9 / PowerDNS | Actual DNS serving | At least one backend required; configurable per pool |
| Neutron | Floating IP and port DNS integration | Optional; enables automatic record creation |
| `python-openstackclient` | Unified CLI (`openstack zone …`, `openstack recordset …`) | Wraps the Designate v2 REST API |
| `python-designateclient` | Python bindings | Used by automation and other services |

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

Designate release notes: https://docs.openstack.org/releasenotes/designate/
