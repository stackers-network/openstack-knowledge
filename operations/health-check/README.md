# Health Check — Cross-Service Verification

This skill covers end-to-end health verification for an OpenStack cloud. Use it after deployment, after an upgrade, or as a routine operational check to confirm every service is responding correctly.

## When to Read This Skill

- Verifying a freshly deployed or upgraded cloud
- Investigating intermittent API failures or user-reported errors
- Running a scheduled operational health review
- Diagnosing which layer (infrastructure, service, or endpoint) has failed

---

## Service Status Overview

Run these commands first to get a broad picture before diving into individual services.

```bash
# Compute service health (nova-conductor, nova-scheduler, nova-compute)
openstack compute service list

# Network agent health (L2, L3, DHCP, metadata agents)
openstack network agent list

# Block storage service health (cinder-volume, cinder-scheduler, cinder-backup)
openstack volume service list

# Load balancer provider availability
openstack loadbalancer provider list
```

Expected: every row shows `up` / `enabled`. Any `down` or `XXX` entry is an incident.

---

## Endpoint and Catalog Verification

```bash
# List all registered endpoints
openstack endpoint list

# Show the service catalog for the authenticated user
openstack catalog list
```

For each endpoint listed, verify it resolves and responds:

```bash
# Generic endpoint curl check (replace URL with each endpoint URL)
curl -s --max-time 5 -o /dev/null -w "%{http_code}" https://keystone.example.com:5000/

# Bulk check all endpoints from catalog (requires jq)
openstack catalog list -f json | \
  jq -r '.[].endpoints[].url' | \
  sort -u | \
  while read url; do
    code=$(curl -sk --max-time 5 -o /dev/null -w "%{http_code}" "$url")
    echo "$code  $url"
  done
```

Acceptable HTTP codes: `200`, `300`, `401` (auth required — means the service is listening). `000` means unreachable.

---

## Keystone (Identity)

```bash
# Issue a token — the most basic auth check
openstack token issue

# Verify catalog is populated
openstack catalog list

# Verify admin endpoint works
openstack domain list

# Check Fernet key rotation is not overdue (keys older than rotation interval will fail)
ls -la /etc/keystone/fernet-keys/
ls -la /etc/keystone/credential-keys/
```

Keystone is healthy when `openstack token issue` succeeds and returns a non-empty token ID.

---

## Nova (Compute)

```bash
# List all compute services and their state
openstack compute service list

# List hypervisors and their resource utilization
openstack hypervisor list
openstack hypervisor show <hypervisor-hostname>

# Show aggregated hypervisor stats
openstack hypervisor stats show

# Check for any instances in error state
openstack server list --all-projects --status ERROR

# Test instance launch (requires a flavor, image, and network)
openstack server create \
  --flavor m1.tiny \
  --image cirros \
  --network shared \
  --wait \
  health-check-test

# Verify it reached ACTIVE
openstack server show health-check-test -f value -c status

# Clean up
openstack server delete health-check-test --wait
```

Nova is healthy when all compute services show `up`, hypervisors show available resources, and a test instance reaches `ACTIVE` within a reasonable time (< 2 minutes for a small instance).

### Nova Cell Verification

```bash
# Verify cells are mapped correctly
nova-manage cell_v2 list_cells

# Verify all compute nodes are mapped to a cell
nova-manage cell_v2 list_hosts

# Check for unmapped compute nodes (should return nothing on a healthy cloud)
nova-manage cell_v2 discover_hosts --verbose
```

---

## Neutron (Networking)

```bash
# List all agents and their status
openstack network agent list

# Check which DHCP agents host a specific network
openstack network agent list --network <network-id-or-name>

# Check which L3 agents host a specific router
openstack network agent list --router <router-id-or-name>

# List networks and verify provider networks resolve
openstack network list

# Test connectivity: launch two instances on the same network and ping between them
# (Use the instance launch procedure above, then use console or floating IP)

# Show router details for L3 verification
openstack router list
openstack router show <router-name>

# Verify Neutron security group rules are intact
openstack security group list
openstack security group rule list default
```

### ML2/OVS Agent Check

```bash
# On the network node or compute node, verify OVS agent is running
systemctl status neutron-openvswitch-agent

# Check OVS bridge configuration
ovs-vsctl show

# Verify br-int, br-tun, br-ex exist as expected
ovs-vsctl list-br
```

### ML2/OVN Agent Check

```bash
# Check ovn-controller status on each node
systemctl status ovn-controller

# Verify OVN southbound and northbound databases are reachable
ovn-nbctl show
ovn-sbctl show

# Check logical switch and router population
ovn-nbctl ls-list
ovn-nbctl lr-list
```

---

## Cinder (Block Storage)

```bash
# List all volume services
openstack volume service list

# List volumes — check for any stuck in error or creating state
openstack volume list --all-projects --status error
openstack volume list --all-projects --status creating

# Test volume lifecycle
openstack volume create --size 1 health-check-vol
openstack volume show health-check-vol -f value -c status
# Wait for 'available'

# Test attaching to an existing instance
openstack server add volume <instance-id> health-check-vol

# Detach and delete
openstack server remove volume <instance-id> health-check-vol
openstack volume delete health-check-vol
```

Cinder is healthy when all `cinder-volume` and `cinder-scheduler` services are `up`, and a test volume reaches `available` status.

---

## Glance (Image)

```bash
# List images — verifies the API is responding
openstack image list

# Show details of a specific image
openstack image show <image-name-or-id>

# Download a small test image to verify backend read access
openstack image save --file /tmp/glance-test-download.img <image-id>
ls -lh /tmp/glance-test-download.img

# Upload a test image to verify backend write access
echo "test" > /tmp/glance-test-upload.img
openstack image create \
  --disk-format raw \
  --container-format bare \
  --file /tmp/glance-test-upload.img \
  health-check-test-image

openstack image show health-check-test-image -f value -c status
# Should be 'active'

openstack image delete health-check-test-image
rm /tmp/glance-test-download.img /tmp/glance-test-upload.img
```

---

## Placement

```bash
# List resource providers (one per compute node, plus shared providers)
openstack resource provider list

# Show inventory for a specific resource provider
openstack resource provider inventory list <rp-uuid>

# Show current allocation totals
openstack resource provider usage show <rp-uuid>

# Verify allocation candidates exist for a basic resource request
openstack allocation candidate list --resource VCPU=1,MEMORY_MB=512,DISK_GB=10
```

Placement is healthy when all compute nodes appear as resource providers and allocation candidates can be listed.

---

## Database Connectivity

Run on each controller node to verify service databases are reachable and contain expected tables.

```bash
# Test each service database connection
mysql -u keystone -p keystone -e "SHOW TABLES;" | wc -l
mysql -u nova -p nova -e "SHOW TABLES;" | wc -l
mysql -u nova_api -p nova_api -e "SHOW TABLES;" | wc -l
mysql -u nova_cell0 -p nova_cell0 -e "SHOW TABLES;" | wc -l
mysql -u neutron -p neutron -e "SHOW TABLES;" | wc -l
mysql -u cinder -p cinder -e "SHOW TABLES;" | wc -l
mysql -u glance -p glance -e "SHOW TABLES;" | wc -l

# Check Galera cluster health (if using MariaDB Galera)
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep%';" | grep -E 'wsrep_cluster_size|wsrep_local_state_comment|wsrep_ready'
# Expect: wsrep_cluster_size = 3 (or your node count), wsrep_local_state_comment = Synced, wsrep_ready = ON

# Check replication lag (should be 0 or near 0)
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_local_recv_queue';"
```

---

## Message Queue (RabbitMQ)

```bash
# Overall RabbitMQ node health
rabbitmqctl status

# List all queues and message counts (non-zero ready messages may indicate stuck consumers)
rabbitmqctl list_queues name messages_ready messages_unacknowledged

# List active connections (should include connections from each service)
rabbitmqctl list_connections user peer_address state

# List consumers per queue
rabbitmqctl list_consumers

# Check cluster node membership
rabbitmqctl cluster_status

# Check for alarms (memory or disk pressure)
rabbitmqctl list_alarms
```

A healthy RabbitMQ cluster has: all nodes `running`, zero alarms, and queues with `messages_ready` counts at or near 0 (no accumulation). Persistent non-zero ready counts in nova, neutron, or cinder queues indicate consumer failures.

---

## Quick Health Script

Run this script to perform a sequential check of all services and print a summary. It exits with code `1` if any check fails.

```bash
#!/bin/bash
# openstack-health-check.sh
# Run as a user with admin credentials sourced (source openrc)

set -euo pipefail

PASS=0
FAIL=0
WARN=0

check() {
  local name="$1"
  local cmd="$2"
  if eval "$cmd" &>/dev/null; then
    echo "[PASS] $name"
    ((PASS++))
  else
    echo "[FAIL] $name"
    ((FAIL++))
  fi
}

warn_if_any() {
  local name="$1"
  local cmd="$2"
  local count
  count=$(eval "$cmd" 2>/dev/null | wc -l)
  if [ "$count" -gt 0 ]; then
    echo "[WARN] $name — $count item(s) found"
    ((WARN++))
  else
    echo "[PASS] $name"
    ((PASS++))
  fi
}

echo "=============================="
echo " OpenStack Health Check"
echo " $(date -u '+%Y-%m-%d %H:%M:%S UTC')"
echo "=============================="

echo ""
echo "--- Auth & Catalog ---"
check "Keystone token issue"          "openstack token issue"
check "Endpoint list"                 "openstack endpoint list"
check "Catalog list"                  "openstack catalog list"

echo ""
echo "--- Compute ---"
check "Compute service list"          "openstack compute service list"
check "Hypervisor list"               "openstack hypervisor list"
warn_if_any "Servers in ERROR state"  "openstack server list --all-projects --status ERROR -f value -c ID"

echo ""
echo "--- Networking ---"
check "Network agent list"            "openstack network agent list"
check "Network list"                  "openstack network list"
check "Router list"                   "openstack router list"

echo ""
echo "--- Block Storage ---"
check "Volume service list"           "openstack volume service list"
warn_if_any "Volumes in error state"  "openstack volume list --all-projects --status error -f value -c ID"

echo ""
echo "--- Image ---"
check "Image list"                    "openstack image list"

echo ""
echo "--- Placement ---"
check "Resource provider list"        "openstack resource provider list"

echo ""
echo "--- Load Balancer ---"
check "LB provider list"              "openstack loadbalancer provider list"

echo ""
echo "=============================="
echo " Results: ${PASS} passed, ${WARN} warnings, ${FAIL} failed"
echo "=============================="

if [ "$FAIL" -gt 0 ]; then
  exit 1
fi
```

Make the script executable and run it:

```bash
chmod +x openstack-health-check.sh
source /etc/openstack/admin-openrc
./openstack-health-check.sh
```

---

## Interpreting Results

| Symptom | Likely Cause | Next Step |
|---|---|---|
| `openstack token issue` fails | Keystone down, DB unreachable, or wrong credentials | Check Keystone service logs, DB connection |
| Compute service `down` | nova-compute or nova-conductor process stopped | `systemctl status nova-compute`, check RabbitMQ connectivity |
| Network agent `down` | neutron-openvswitch-agent or ovn-controller stopped | `systemctl status neutron-*`, check connectivity to Neutron server |
| Volume service `down` | cinder-volume process stopped or backend unavailable | `systemctl status cinder-volume`, check backend (Ceph, LVM, etc.) |
| Large `messages_ready` in RabbitMQ | Service consumers are down or overwhelmed | Identify the queue name (maps to a service), check that service |
| Endpoint returns `000` | DNS failure, firewall, or service not listening | `curl -v` to the endpoint, check HAProxy/nginx and service process |
| Glance image stays `saving` | Glance worker stuck or backend write failure | Check glance-api logs, backend storage (Ceph, file, Swift) health |
| `No valid host` on server create | Placement has no candidates or Nova scheduler filtered all | Check `openstack allocation candidate list`, check host aggregates |
