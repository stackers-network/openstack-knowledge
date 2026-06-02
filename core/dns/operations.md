# Designate Operations

## Prerequisites

Confirm the Designate API endpoint is available:

```bash
openstack catalog show dns
openstack zone list
```

All zone names and record set names must be fully qualified (trailing dot). The `python-openstackclient` package with `python-designateclient` plugin provides the `openstack zone` and `openstack recordset` subcommands.

## Zones

### Create a Primary Zone

```bash
# Minimal zone creation
openstack zone create --email admin@example.com example.com.

# With explicit TTL and description
openstack zone create \
  --email admin@example.com \
  --ttl 300 \
  --description "Production web zone" \
  example.com.

# Assign to a specific pool (admin)
openstack zone create \
  --email admin@example.com \
  --attributes pool_id:<pool-uuid> \
  example.com.
```

### Create a Secondary Zone

```bash
# Follow an external primary nameserver
openstack zone create \
  --email secondary@example.com \
  --type secondary \
  --masters 198.51.100.1 \
  external.example.com.

# Multiple masters
openstack zone create \
  --email secondary@example.com \
  --type secondary \
  --masters 198.51.100.1,198.51.100.2 \
  external.example.com.
```

### List and Show

```bash
openstack zone list
openstack zone list --type PRIMARY
openstack zone list --name example.com.

openstack zone show example.com.
```

Key fields in `zone show` output:

| Field | Meaning |
|---|---|
| `serial` | Current SOA serial number |
| `status` | `ACTIVE`, `PENDING`, or `ERROR` |
| `action` | Current action in progress (`CREATE`, `UPDATE`, `DELETE`, `NONE`) |
| `pool_id` | Which DNS pool serves this zone |
| `masters` | (Secondary zones only) upstream primary nameservers |

### Update a Zone

```bash
# Change the default TTL
openstack zone set --ttl 600 example.com.

# Change the contact email
openstack zone set --email ops@example.com example.com.

# Change the description
openstack zone set --description "Updated production zone" example.com.
```

### Delete a Zone

```bash
openstack zone delete example.com.
```

Deletion is asynchronous. The zone enters `PENDING_DELETE` status while Designate removes it from all backends.

### Abandon a Zone (Admin)

Removes the zone from Designate's database without touching the DNS backends (useful when migrating zones out of Designate):

```bash
openstack zone abandon example.com.
```

## Record Sets

All record set operations use the pattern: `openstack recordset <action> <zone> <name>`

### A Records (IPv4 addresses)

```bash
# Single A record
openstack recordset create \
  --type A \
  --record 203.0.113.10 \
  example.com. www

# Multiple A records (DNS round-robin)
openstack recordset create \
  --type A \
  --record 203.0.113.10 \
  --record 203.0.113.11 \
  example.com. www

# With explicit TTL
openstack recordset create \
  --type A \
  --record 203.0.113.10 \
  --ttl 60 \
  example.com. www
```

### AAAA Records (IPv6 addresses)

```bash
openstack recordset create \
  --type AAAA \
  --record "2001:db8::1" \
  example.com. ipv6

openstack recordset create \
  --type AAAA \
  --record "2001:db8::1" \
  --record "2001:db8::2" \
  example.com. www
```

### MX Records (Mail exchangers)

The record value format is `<priority> <target>` with a trailing dot on the target:

```bash
# Primary mail server
openstack recordset create \
  --type MX \
  --record "10 mail.example.com." \
  example.com. ""

# Primary and backup mail servers
openstack recordset create \
  --type MX \
  --record "10 mail.example.com." \
  --record "20 mail2.example.com." \
  example.com. ""
```

The empty string `""` as the name means the record is at the zone apex (`@`).

### CNAME Records

```bash
# www is a CNAME to the apex
openstack recordset create \
  --type CNAME \
  --record example.com. \
  example.com. www

# Subdomain alias
openstack recordset create \
  --type CNAME \
  --record backend.internal.example.com. \
  example.com. api
```

A CNAME cannot coexist with other record types at the same name. CNAMEs at the zone apex are not valid (use A/AAAA instead).

### TXT Records

```bash
# SPF record at zone apex
openstack recordset create \
  --type TXT \
  --record '"v=spf1 include:_spf.google.com ~all"' \
  example.com. ""

# DMARC record
openstack recordset create \
  --type TXT \
  --record '"v=DMARC1; p=reject; rua=mailto:dmarc@example.com"' \
  example.com. _dmarc

# Domain verification
openstack recordset create \
  --type TXT \
  --record '"google-site-verification=abc123xyz"' \
  example.com. ""
```

TXT record data must be enclosed in double quotes within the record value string.

### SRV Records

Format: `<priority> <weight> <port> <target>`

```bash
# SIP over TLS
openstack recordset create \
  --type SRV \
  --record "10 20 5061 sip.example.com." \
  example.com. _sips._tcp

# XMPP client service
openstack recordset create \
  --type SRV \
  --record "5 0 5222 xmpp.example.com." \
  example.com. _xmpp-client._tcp
```

### NS Records

NS records at the zone apex are auto-generated by Designate from the pool configuration. To add delegating NS records for a subdomain:

```bash
# Delegate sub.example.com to separate nameservers
openstack recordset create \
  --type NS \
  --record "ns1.sub.example.com." \
  --record "ns2.sub.example.com." \
  example.com. sub
```

### CAA Records

```bash
# Allow only Let's Encrypt to issue certificates
openstack recordset create \
  --type CAA \
  --record "0 issue letsencrypt.org" \
  example.com. ""
```

### List, Show, Update, Delete

```bash
# List all record sets in a zone
openstack recordset list example.com.

# Filter by type
openstack recordset list --type A example.com.

# Show a specific record set
openstack recordset show example.com. www

# Update (replace) records in a record set
openstack recordset set \
  --record 203.0.113.20 \
  example.com. www

# Add additional records without replacing (API: GET then PUT)
# The CLI --record flag replaces all records; use the API for atomic addition

# Delete a record set
openstack recordset delete example.com. www
```

## PTR Records and Reverse DNS

### Create a Reverse Zone

Reverse zones follow the in-addr.arpa naming convention:

```bash
# Reverse zone for 203.0.113.0/24
openstack zone create \
  --email admin@example.com \
  113.0.203.in-addr.arpa.

# Reverse zone for 2001:db8::/32 (IPv6)
openstack zone create \
  --email admin@example.com \
  8.b.d.0.1.0.0.2.ip6.arpa.
```

### Create PTR Records Manually

```bash
# PTR for 203.0.113.10 → web.example.com.
openstack recordset create \
  --type PTR \
  --record web.example.com. \
  113.0.203.in-addr.arpa. 10

# PTR for 203.0.113.25
openstack recordset create \
  --type PTR \
  --record mail.example.com. \
  113.0.203.in-addr.arpa. 25
```

### PTR Records for Floating IPs (via Neutron integration)

When `designate-sink` is configured with the Neutron floating IP handler, PTR records are created automatically when floating IPs are allocated. Verify the sink is running:

```bash
# Check sink is consuming notifications (on the designate-sink host)
journalctl -u designate-sink --since "5 minutes ago"
```

To manually create a PTR for a floating IP using the Designate API:

```bash
# Using the designateclient directly
openstack ptr record set \
  --description "Web server floating IP" \
  RegionOne:203.0.113.10 \
  web.example.com.

openstack ptr record list
openstack ptr record show RegionOne:203.0.113.10
openstack ptr record unset RegionOne:203.0.113.10
```

## Zone Transfers

Zone transfers allow a zone to be transferred from one OpenStack project to another.

### Request a Transfer

The zone owner creates a transfer request:

```bash
# Create a transfer request (optionally restrict to a target project)
openstack zone transfer request create example.com.

# Restrict to a specific target project
openstack zone transfer request create \
  --target-project-id <project-uuid> \
  example.com.

# List pending transfer requests
openstack zone transfer request list
openstack zone transfer request show <request-id>
```

The response includes a `key` that must be shared with the recipient out-of-band.

### Accept a Transfer

The recipient project accepts the transfer:

```bash
# Accept using the transfer request ID and key
openstack zone transfer accept request \
  --transfer-id <request-id> \
  --key <transfer-key>

# List accepted transfers
openstack zone transfer accept list
```

After acceptance, the zone and all its record sets belong to the recipient project. The original project loses access.

### Delete a Transfer Request

```bash
openstack zone transfer request delete <request-id>
```

## Zone Import and Export

### Export a Zone

Export a zone as a standard RFC 1035 zone file:

```bash
# Request an export (asynchronous)
openstack zone export create example.com.

# List exports
openstack zone export list

# Download the zone file once status is COMPLETE
openstack zone export showfile <export-id>
openstack zone export showfile <export-id> > example.com.zone
```

### Import a Zone File

```bash
# Import a standard RFC 1035 zone file
openstack zone import create example.com.zone

# List imports
openstack zone import list

# Show import status
openstack zone import show <import-id>
```

## TSIG Keys

TSIG (Transaction SIGnature) keys authenticate DNS zone transfers between Designate's mini-DNS and backend nameservers.

### Create TSIG Keys

```bash
# Create a TSIG key (algorithm defaults to hmac-sha256)
openstack zone tsig key create \
  --name tsig-key-1 \
  --algorithm hmac-sha256 \
  --secret "$(openssl rand -base64 32)"

# Supported algorithms: hmac-md5, hmac-sha1, hmac-sha224, hmac-sha256, hmac-sha384, hmac-sha512

# List TSIG keys
openstack zone tsig key list
openstack zone tsig key show tsig-key-1

# Delete a TSIG key
openstack zone tsig key delete tsig-key-1
```

TSIG keys must also be configured on the receiving nameserver (BIND9 `tsig-keygen` or PowerDNS TSIG API) with matching algorithm and secret.

## Quotas

```bash
# Show quotas for the current project
openstack dns quota list

# Show quotas for a specific project (admin)
openstack dns quota list --project <project-id>

# Set quotas (admin)
openstack dns quota set \
  --zones 20 \
  --zone-records 500 \
  --zone-recordsets 300 \
  --recordset-records 20 \
  <project-id>

# Reset to defaults
openstack dns quota reset <project-id>
```

## Checking Zone Propagation

After creating or updating records, verify propagation to the DNS backends:

```bash
# Show zone status (should be ACTIVE when propagated)
openstack zone show example.com.

# Query the backend nameserver directly
dig @ns1.example.com example.com. SOA
dig @ns1.example.com www.example.com. A
dig @ns1.example.com 10.113.0.203.in-addr.arpa. PTR

# Check serial number matches what Designate reports
DESIGNATE_SERIAL=$(openstack zone show example.com. -f value -c serial)
DNS_SERIAL=$(dig @ns1.example.com example.com. SOA +short | awk '{print $3}')
echo "Designate: $DESIGNATE_SERIAL  DNS: $DNS_SERIAL"
```

## Common Troubleshooting

```bash
# Check zone is not stuck in PENDING
openstack zone list --status PENDING

# Check for ERROR zones
openstack zone list --status ERROR

# Check designate-worker logs
journalctl -u designate-worker --since "15 minutes ago"

# Check designate-central logs
journalctl -u designate-central --since "15 minutes ago"

# Verify mini-DNS is responding (on the designate-mdns host)
dig @127.0.0.1 -p 5354 example.com. AXFR

# Force a zone resync to backends (admin)
openstack zone touch example.com.
```
