# Neutron Architecture

## Neutron Server

The `neutron-server` process is a WSGI application that exposes the Neutron v2 REST API. It handles no actual packet forwarding — its job is to accept API requests, validate them, persist state to the database, and dispatch configuration to agents via RPC (AMQP/RabbitMQ) or directly to backend SDN controllers.

Internal structure of neutron-server:

```
neutron-server
├── API layer (Pecan/WSGI, versioned at /v2.0)
│   ├── Resource controllers (networks, subnets, ports, routers, …)
│   └── Policy enforcement (oslo.policy, RBAC)
├── Plugin framework
│   ├── Core plugin  →  ML2Plugin (most deployments)
│   └── Service plugins (L3, QoS, Trunk, Segments, BGP, …)
├── DB layer (oslo.db → SQLAlchemy → MariaDB/PostgreSQL)
├── RPC layer (oslo.messaging → RabbitMQ)
│   ├── Cast/call to agents (OVS agent, DHCP agent, L3 agent, metadata agent)
│   └── Callbacks from agents (port status updates, binding confirmations)
└── Extension manager (loads API extensions from entry points)
```

neutron-server runs as multiple workers (set `api_workers` in `neutron.conf`). All workers share the same DB connection pool and the same RabbitMQ connection.

## ML2 Plugin

The **Modular Layer 2 (ML2)** plugin is the standard core plugin. It replaces the older monolithic plugins (OVS plugin, Linux Bridge plugin). ML2 decomposes network behaviour into two independent driver hierarchies:

### Type Drivers

Type drivers manage the allocation and lifecycle of network segment identifiers (VLAN IDs, VXLAN VNIs, etc.).

| Type | Segment ID | Use Case |
|---|---|---|
| `flat` | none (untagged) | Provider networks on a dedicated physical NIC |
| `vlan` | 802.1Q VLAN ID (1–4094) | Provider networks; isolation via VLAN tagging |
| `vxlan` | VNI (1–16 777 215) | Tenant networks; overlay tunnels between compute nodes |
| `gre` | tunnel key | Tenant overlay; less common than VXLAN |
| `geneve` | VNI | Used by OVN; carries metadata in header extensions |
| `local` | none | Single-node testing; no inter-node connectivity |

Type drivers are configured in `ml2_conf.ini`:

```ini
[ml2]
type_drivers = flat,vlan,vxlan,geneve
tenant_network_types = vxlan,geneve
```

### Mechanism Drivers

Mechanism drivers implement the actual wiring of ports to the underlying network infrastructure. Multiple mechanism drivers can be active simultaneously.

| Driver | Backend | Agent Required |
|---|---|---|
| `openvswitch` | Open vSwitch | `neutron-openvswitch-agent` on each compute/network node |
| `ovn` | OVN (ovsdb) | `ovn-controller` on each node; no separate Neutron agent |
| `linuxbridge` | Linux bridge + VXLAN/VLAN | `neutron-linuxbridge-agent` on each node |
| `l2population` | Flood/learn optimization | Works alongside OVS or Linux Bridge to pre-populate FDB |
| `macvtap` | SR-IOV macvtap | For VMs with direct hardware attachment |

Configuration:

```ini
[ml2]
mechanism_drivers = ovn,l2population
# or for OVS deployments:
# mechanism_drivers = openvswitch,l2population
```

### Extension Manager

ML2's extension manager loads port binding extensions, port security, VLAN transparency, and others as ML2 extension drivers:

```ini
[ml2]
extension_drivers = port_security,qos,dns
```

## Agents

In ML2/OVS deployments, several agents run on each host and communicate with neutron-server over RabbitMQ.

### L2 Agent (OVS Agent)

`neutron-openvswitch-agent` runs on every compute node and network node. Responsibilities:

- Programs OVS flow rules on `br-int` and `br-tun` in response to RPC from neutron-server
- Manages tunnel endpoints (VTEP) for VXLAN/GRE
- Handles port binding confirmation
- Reports agent state (heartbeat) to neutron-server every `report_interval` seconds (default 30 s)
- Implements security group rules as OVS flows (OVS firewall driver) or iptables (hybrid driver)

In ML2/OVN deployments the OVS agent is replaced by `ovn-controller` (see [ovn.md](ovn.md)).

### L3 Agent

`neutron-l3-agent` runs on dedicated network nodes (or on compute nodes in DVR mode). Responsibilities:

- Creates Linux network namespaces (`qrouter-<uuid>`) for each virtual router
- Configures IP addresses, routing tables, iptables NAT rules inside the namespace
- Implements SNAT for floating IPs and for instances without floating IPs (centralized SNAT)
- In HA mode: runs keepalived/VRRP inside the namespace for failover between L3 agent nodes
- In DVR mode: each compute node hosts a partial router namespace for east-west traffic; only SNAT remains centralized

### DHCP Agent

`neutron-dhcp-agent` runs on network nodes. Responsibilities:

- Spawns one `dnsmasq` process per network that has DHCP enabled
- Each dnsmasq process runs in a Linux network namespace (`qdhcp-<uuid>`)
- Passes host/DHCP entries to dnsmasq via a `hosts` file and `opts` file
- Supports HA: multiple DHCP agents can be scheduled to the same network; all serve DHCP simultaneously (active-active)

### Metadata Agent

`neutron-metadata-agent` runs on network nodes (and on compute nodes in DVR mode). Responsibilities:

- Provides a proxy for the Nova metadata API (`http://169.254.169.254`)
- Instances send HTTP to 169.254.169.254; the request enters the router or DHCP namespace, is forwarded to the metadata agent, which adds `X-Forwarded-For` and `X-Neutron-Network-Id` headers and proxies to `http://<nova-api>:8775/openstack/`
- In OVN deployments a dedicated `neutron-ovn-metadata-agent` handles this using OVN port binding extensions (see [ovn.md](ovn.md))

## Network Types

### Provider Networks

Provider networks map directly to physical network infrastructure the operator controls. They are typically used for:

- External/floating IP networks (mapped to an uplink VLAN or untagged interface)
- Networks shared to tenants that need direct physical access

Key attributes:

```
--provider-physical-network  physnet1       # maps to bridge_mappings in agent config
--provider-network-type      flat|vlan|...
--provider-segment           200            # VLAN ID (for vlan type)
```

Provider networks can be `--external` (used as router gateways / floating IP pools) or shared with `--share`.

### Tenant / Project Networks

Tenant networks are created by end users. They are isolated by VXLAN VNI, GRE key, or GENEVE VNI and live entirely in the overlay. Traffic leaves the overlay only when:

1. It passes through a virtual router with an external gateway (SNAT/floating IP)
2. It is destined for another tenant network behind the same router (east-west)

## Core Resource Model

```
Network  (L2 domain)
 └── Subnet  (IP range, gateway, DHCP pool, DNS, routes)
      └── Port  (attachment point; has MAC, one or more fixed IPs)
           └── Bound to: instance, router interface, DHCP tap, load balancer, etc.

Router  (L3 gateway)
 ├── Router interfaces (connect subnets to the router)
 └── External gateway (connects router to provider network)

Floating IP  (1:1 NAT mapping)
 ├── External IP (from provider subnet)
 └── Fixed IP (on a port attached to an instance)

Security Group  (stateful firewall)
 └── Security Group Rules (ingress/egress, protocol, port range, CIDR or remote SG)
```

### Port States and Binding

A port moves through these states:

| Status | Meaning |
|---|---|
| `DOWN` | Port created but not bound to a host |
| `BUILD` | Nova has requested binding; agent is programming the dataplane |
| `ACTIVE` | Port is fully bound and dataplane is configured |
| `ERROR` | Binding failed |

The `binding:vif_type` indicates what interface type Nova should plumb (e.g., `ovs`, `vhostuser`, `hw_veb` for SR-IOV).

## Extension Framework

Neutron's API is extensible via:

1. **API extensions** — add new resources or extend existing ones (e.g., `router`, `security-group`, `floating-ips`, `port-security`, `dns-integration`)
2. **Service plugins** — loaded at startup, implement entire feature areas with their own DB tables and agents (e.g., `router` / L3, `qos`, `trunk`, `segments`, `bgp`, `vpnaas`, `lbaas`)

Extensions are discovered from entry points. List loaded extensions:

```bash
openstack extension list --network
```

## Nova Integration — Port Binding During Instance Boot

When Nova boots an instance that requests a Neutron port:

1. Nova calls `POST /v2.0/ports` (or uses a pre-created port) with `binding:host_id` set to the compute node hostname
2. Neutron ML2 iterates mechanism drivers calling `bind_port()` in order
3. The winning mechanism driver sets `binding:vif_type` and `binding:vif_details`
4. Nova reads `vif_type` to determine how to attach the guest NIC (tap, vhostuser, etc.)
5. The L2 agent on that compute node receives an RPC `port_update` and programs OVS flows
6. The agent calls back with `update_device_up` to set port status to `ACTIVE`
7. Nova is notified via the `network-changed` server event and the instance network is ready

In OVN deployments steps 5-6 are handled by `ovn-controller` writing directly to the OVN southbound DB; no Neutron RPC round-trip to an agent is needed.
