# Load Balancing — Octavia

Octavia is the OpenStack Load Balancing service. It provisions and manages load balancers as a service (LBaaS), distributing incoming traffic across a pool of back-end instances. Each load balancer runs as a dedicated virtual machine (the *amphora*) on compute nodes, with HAProxy handling the actual traffic forwarding inside that VM. Octavia replaced the deprecated Neutron LBaaS v2 extension and is the reference LBaaS implementation for all current OpenStack releases.

## When to Read This Skill

- Deploying Octavia and configuring its management network and amphora image
- Creating load balancers, listeners, pools, and members via CLI or API
- Configuring health monitors, L7 policies, and TLS termination
- Troubleshooting amphora failures, health heartbeat timeouts, or failover
- Switching from the amphora provider to the OVN provider driver
- Understanding amphora lifecycle management and the spare pool
- Configuring Octavia flavors and anti-affinity placement
- Tuning health-manager, housekeeping, and worker processes

## Sub-Files

| File | What It Covers |
|---|---|
| [architecture.md](architecture.md) | octavia-api, octavia-worker, health-manager, housekeeping; amphora design (HAProxy VM, management network, heartbeats); provider framework (amphora, OVN); flavors and anti-affinity |
| [operations.md](operations.md) | CLI and API: load balancers, listeners, pools, members, health monitors, L7 policies, stats, failover, quota |
| [internals.md](internals.md) | Amphora lifecycle (create, configure, heartbeat, failover, delete); Taskflow-based operations; provider driver framework; TLS certificate management; Octavia flavors deep-dive |

## Quick Reference

```bash
# Create a load balancer on a private subnet
openstack loadbalancer create --name web-lb --vip-subnet-id private-subnet

# Wait for ACTIVE/ONLINE status
openstack loadbalancer show web-lb

# Add an HTTP listener
openstack loadbalancer listener create \
  --name http-listener \
  --protocol HTTP \
  --protocol-port 80 \
  web-lb

# Create a round-robin pool
openstack loadbalancer pool create \
  --name web-pool \
  --protocol HTTP \
  --lb-algorithm ROUND_ROBIN \
  --listener http-listener

# Add back-end members
openstack loadbalancer member create \
  --name web-1 \
  --address 192.168.1.10 \
  --protocol-port 8080 \
  web-pool

# Add an HTTP health monitor
openstack loadbalancer healthmonitor create \
  --name hm-http \
  --type HTTP \
  --delay 5 \
  --timeout 3 \
  --max-retries 3 \
  web-pool

# Check stats
openstack loadbalancer stats show web-lb
```

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| Keystone | Authentication and service catalog | All Octavia API calls use Keystone tokens |
| Nova | Spawns amphora VMs | Octavia uses a dedicated Nova flavor and image |
| Neutron | VIP ports, management network, security groups | Octavia creates Neutron ports for VIPs and amphorae |
| Glance | Amphora image storage | A pre-built amphora image must be uploaded before deployment |
| MariaDB / PostgreSQL | Octavia database | Stores LB, listener, pool, member, and amphora state |
| RabbitMQ | RPC between octavia-api, worker, health-manager | oslo.messaging |
| Barbican | TLS certificates for HTTPS termination | Optional; required when using `TERMINATED_HTTPS` |
| `python-openstackclient` | Unified CLI (`openstack loadbalancer …`) | Wraps the Octavia v2 REST API |

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

Octavia release notes: https://docs.openstack.org/releasenotes/octavia/
