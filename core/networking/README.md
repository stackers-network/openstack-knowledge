# Networking — Neutron

Neutron is the OpenStack Networking service. It provides network connectivity as a service between interface devices managed by other OpenStack services (primarily Nova). Neutron exposes a REST API to create and manage networks, subnets, routers, floating IPs, security groups, and ports, and delegates actual packet forwarding to pluggable backend drivers — most commonly ML2/OVS or ML2/OVN.

## When to Read This Skill

- Deploying Neutron for the first time or migrating from nova-network
- Creating tenant networks, provider networks, routers, or floating IPs
- Configuring ML2 plugin, OVS agent, or OVN integration
- Troubleshooting connectivity failures between instances
- Implementing security groups, QoS policies, or trunk ports
- Enabling distributed virtual routing (DVR) or HA routers
- Migrating from ML2/OVS to ML2/OVN
- Configuring BGP dynamic routing, VPNaaS, or DNS integration
- Understanding port binding, DHCP, or metadata service flow
- Extending Neutron with service plugins or custom ML2 drivers

## Sub-Files

| File | What It Covers |
|---|---|
| [architecture.md](architecture.md) | Neutron server, ML2 plugin, type/mechanism drivers, agents (L2, L3, DHCP, metadata), network model, extension framework, Nova integration |
| [operations.md](operations.md) | CLI and API: networks, subnets, routers, floating IPs, security groups, ports, agents, RBAC, config file reference |
| [internals.md](internals.md) | ML2 internals, port binding flow, L2 population, DHCP agent/dnsmasq, metadata proxy chain, L3 namespaces, HA/DVR routers, security group implementation, RPC, DB models |
| [ovs.md](ovs.md) | Open vSwitch deep-dive: br-int, br-tun, br-ex, flow tables, OVS agent, security group firewall driver, DPDK, diagnostic commands |
| [ovn.md](ovn.md) | OVN deep-dive: architecture, northbound/southbound DBs, distributed routing, native DHCP/security groups, metadata agent, migration from OVS |
| [advanced.md](advanced.md) | Trunking, QoS, network segments, BGP dynamic routing, VPNaaS, port forwarding, DNS integration |

## Quick Reference

```bash
# Create a private tenant network (VXLAN)
openstack network create --provider-network-type vxlan private-net

# Create a provider (external) network
openstack network create \
  --provider-physical-network physnet1 \
  --provider-network-type flat \
  --external \
  provider-net

# Create subnets
openstack subnet create \
  --network private-net \
  --subnet-range 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --dns-nameserver 8.8.8.8 \
  private-subnet

openstack subnet create \
  --network provider-net \
  --subnet-range 203.0.113.0/24 \
  --gateway 203.0.113.1 \
  --no-dhcp \
  provider-subnet

# Create and wire a router
openstack router create main-router
openstack router add subnet main-router private-subnet
openstack router set --external-gateway provider-net main-router

# Allocate and assign a floating IP
openstack floating ip create provider-net
openstack server add floating ip my-instance 203.0.113.15

# Create a security group with common rules
openstack security group create web-sg --description "Web tier"
openstack security group rule create web-sg --protocol tcp --dst-port 80 --remote-ip 0.0.0.0/0
openstack security group rule create web-sg --protocol tcp --dst-port 443 --remote-ip 0.0.0.0/0
openstack security group rule create web-sg --protocol icmp

# Check agent health
openstack network agent list
```

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| Keystone | Authentication and service catalog | Every Neutron API call is validated against Keystone tokens |
| MariaDB 10.6+ or PostgreSQL 14+ | Neutron database (networks, subnets, ports, routers) | MariaDB most common in production |
| RabbitMQ 3.12+ | AMQP message bus between neutron-server and agents | oslo.messaging; critical for agent RPC |
| Open vSwitch 3.1+ or OVN 23.x+ | Dataplane: packet forwarding, tunneling, security | OVN is the recommended backend for 2025.x+ |
| Nova | Port binding during instance boot; metadata API | Neutron calls Nova on port binding events |
| `python-openstackclient` | Unified CLI | Wraps the Neutron v2 REST API |
| `python-neutronclient` | Lower-level Python bindings | Used by automation and other services |
| dnsmasq | DHCP and DNS for tenant networks | Managed by the DHCP agent (not needed with OVN native DHCP) |
| keepalived + conntrackd | HA routers (VRRP failover) | Required only when using L3 agent HA mode |
| BIRD or GoBGP | BGP dynamic routing | Required only when using neutron-dynamic-routing |

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

Neutron release notes: https://docs.openstack.org/releasenotes/neutron/
