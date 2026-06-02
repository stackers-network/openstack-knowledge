# OVN Deep-Dive

## Overview

OVN (Open Virtual Network) is a network virtualization solution built on top of OVS. In OpenStack, OVN is used via the ML2/OVN mechanism driver and replaces the need for separate L3 agent, DHCP agent, and OVS agent processes. OVN is the **recommended backend** for new OpenStack deployments as of 2023.1+.

Key advantages over ML2/OVS:

- Distributed routing on every hypervisor (no centralized L3 agent bottleneck)
- Native DHCP (no dnsmasq processes, no DHCP agent)
- Native ACL-based security groups (no iptables, no conntrack in userspace)
- Fewer moving parts: no RabbitMQ dependency for dataplane operations
- Better scalability: thousands of logical ports without per-port RPC storms

## OVN Architecture

```
┌──────────────────────────────────────────────────────┐
│  Neutron Server                                       │
│  (neutron-server with ML2/OVN mechanism driver)       │
│         │ writes to                                   │
│         ▼                                             │
│  OVN Northbound DB (ovsdb-server)                     │
│  ├── Logical_Switch (one per Neutron network)         │
│  ├── Logical_Switch_Port (one per Neutron port)       │
│  ├── Logical_Router (one per Neutron router)          │
│  ├── Logical_Router_Port                              │
│  ├── Load_Balancer                                    │
│  ├── ACL (security group rules)                       │
│  ├── Address_Set (security group membership)          │
│  ├── DHCP_Options (native DHCP config)                │
│  └── QoS, NAT, Port_Group, ...                       │
│         │ processed by                                │
│         ▼                                             │
│  ovn-northd (translation daemon)                      │
│         │ writes to                                   │
│         ▼                                             │
│  OVN Southbound DB (ovsdb-server)                     │
│  ├── Datapath_Binding (maps to logical network/router)│
│  ├── Port_Binding (maps logical port to chassis)      │
│  ├── Logical_Flow (abstract flow instructions)        │
│  ├── MAC_Binding (ARP/ND cache)                       │
│  ├── Chassis (registered hypervisors/gateways)        │
│  └── Gateway_Chassis                                 │
└──────────────────────────────────────────────────────┘
           │ consumed by
           ▼
  ovn-controller  (runs on every hypervisor/gateway)
  ├── Reads Logical_Flows and Port_Bindings from SB DB
  ├── Programs local OVS flow tables
  ├── Creates/destroys Geneve tunnels to other chassis
  └── Reports chassis state back to SB DB
```

### ovn-northd

`ovn-northd` is the translation daemon. It watches the Northbound DB for changes and translates logical network topology into logical flows in the Southbound DB. It does not touch hypervisors directly — `ovn-controller` on each host consumes the logical flows and translates them to physical OVS flows.

`ovn-northd` is typically run in HA mode using the built-in OVSDB clustering (Raft-based, 3-node cluster) or behind a load balancer.

### ovn-controller

`ovn-controller` runs on every hypervisor and gateway node. It:

1. Registers the local chassis in the Southbound DB `Chassis` table
2. Watches `Port_Binding` and `Logical_Flow` tables for changes
3. Translates logical flows into OVS flows and programs them via `ovs-vsctl` / OVSDB
4. Claims port bindings for ports on its local hypervisor
5. Creates Geneve tunnel ports to other chassis automatically

No RabbitMQ is involved — all communication is via OVSDB protocol.

## Integration with Neutron — ML2/OVN Mechanism Driver

The ML2/OVN mechanism driver translates Neutron API operations directly into OVN Northbound DB entries:

| Neutron API Call | OVN NB Change |
|---|---|
| `POST /v2.0/networks` | Creates `Logical_Switch` |
| `POST /v2.0/subnets` | Adds `DHCP_Options` to the switch |
| `POST /v2.0/ports` | Creates `Logical_Switch_Port` |
| `POST /v2.0/routers` | Creates `Logical_Router` |
| `PUT /v2.0/routers/{id}/add_router_interface` | Creates `Logical_Router_Port` + patch port pair |
| `POST /v2.0/floatingips` | Adds `NAT` rule on the router |
| `POST /v2.0/security-groups` | Creates `Port_Group` |
| `POST /v2.0/security-group-rules` | Adds `ACL` entries |

Port binding in OVN: when a port is bound to a chassis, `ovn-controller` on that chassis sets `Port_Binding.chassis` and programs flows locally. No RPC to neutron-server is needed for the dataplane to become active.

Configuration in `ml2_conf.ini`:

```ini
[ml2]
mechanism_drivers = ovn
type_drivers = local,flat,vlan,geneve
tenant_network_types = geneve
extension_drivers = port_security,qos,dns

[ml2_type_geneve]
vni_ranges = 1:65535
max_header_size = 38

[ovn]
ovn_nb_connection = tcp:192.168.0.10:6641,tcp:192.168.0.11:6641,tcp:192.168.0.12:6641
ovn_sb_connection = tcp:192.168.0.10:6642,tcp:192.168.0.11:6642,tcp:192.168.0.12:6642
ovn_l3_scheduler = leastloaded
ovn_metadata_enabled = True
enable_distributed_floating_ip = True
ovn_emit_need_to_frag = False
dns_servers = 8.8.8.8,8.8.4.4
```

## Distributed Routing

OVN implements routing in the logical flow pipeline. Every chassis that hosts a VM behind a router has the router's logical flows compiled and programmed locally. East-west traffic between VMs on different subnets never leaves the hypervisor:

```
VM1 (192.168.1.10) on compute01
  → OVS flows on compute01 match IP dst 10.0.0.20
  → Logical router flow: decrement TTL, rewrite dst MAC, forward to compute02
  → Geneve tunnel to compute02
  → VM2 (10.0.0.20) on compute02
```

No packets traverse a dedicated network node for east-west traffic.

For north-south with floating IPs, `enable_distributed_floating_ip = True` means DNAT/SNAT is also performed on the compute node. The compute node must have an external network bridge (`br-ex`) with a physical uplink.

For centralized SNAT (instances without floating IPs), traffic goes to a **gateway chassis** (see below).

## Native DHCP

OVN implements DHCP entirely in the OVS flow pipeline. When `DHCP_Options` are configured for a subnet:

1. The `ovn-controller` on the hypervisor programs a flow that intercepts DHCP DISCOVER/REQUEST packets from VMs
2. The flow constructs a DHCP OFFER/ACK response inline and sends it back to the VM
3. No separate dnsmasq process, no DHCP namespace, no DHCP agent

The `DHCP_Options` record contains all fields needed to construct the response:

```
_uuid               : xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
cidr                : "192.168.1.0/24"
external_ids        : {neutron:subnet_id="<subnet-uuid>"}
options             : {classless_static_route="...", dns_server="{8.8.8.8,8.8.4.4}",
                       lease_time="43200", mtu="1442", router="192.168.1.1",
                       server_id="192.168.1.1", server_mac="fa:16:3e:xx:xx:xx"}
```

## Native Security Groups (ACLs)

Security group rules are implemented as OVN `ACL` entries attached to `Port_Group` objects. Each security group maps to a Port_Group; each security group rule maps to an ACL.

```
Port_Group: pg-<security-group-uuid>
  ports: [<lsp-uuid1>, <lsp-uuid2>, ...]   # all ports in this SG

ACL:
  direction: from-lport  (egress from VM) | to-lport (ingress to VM)
  priority: 1002
  match: "inport == @pg-<sg-uuid> && tcp && tcp.dst == 80"
  action: allow-related
  log: false
```

OVN compiles ACLs into logical flows using conntrack state tracking. The entire security group evaluation is done in the OVS kernel datapath — no iptables chains, no userspace involvement per packet.

Address sets allow referencing all IPs in a security group efficiently:

```
Address_Set: as-<security-group-uuid>
  addresses: ["192.168.1.10", "192.168.1.20", ...]
```

Rule "allow from security group app-sg" compiles to:
```
match: "ip4.src == $as-<app-sg-uuid> && tcp.dst == 5432"
```

## OVN Metadata Agent

Since OVN replaces the DHCP agent and its network namespaces, a dedicated `neutron-ovn-metadata-agent` provides the metadata proxy service. It runs on every compute node (not just network nodes).

The agent:
1. Watches OVN Southbound DB for port bindings on the local chassis
2. Creates a haproxy instance per network to proxy `169.254.169.254:80` requests
3. Uses OVN `localport` type ports for metadata tap devices (unique per chassis, not routed)
4. Adds the required HTTP headers and forwards to Nova metadata API

Configuration in `ovn_metadata_agent.ini`:

```ini
[DEFAULT]
nova_metadata_host = controller01
nova_metadata_port = 8775
metadata_proxy_shared_secret = metadata-proxy-secret-change-me
metadata_workers = 2

[ovs]
ovsdb_connection = unix:/var/run/openvswitch/db.sock

[ovn]
ovn_sb_connection = tcp:192.168.0.10:6642
```

## Gateway Nodes and Chassis Scheduling

Gateway chassis handle:
- Centralized SNAT for instances without floating IPs
- Ingress/egress for provider networks when compute nodes do not have direct uplinks

A node is designated as a gateway chassis by adding it to the `ovn-cms-options`:

```bash
ovs-vsctl set Open_vSwitch . external-ids:ovn-cms-options=enable-chassis-as-gw
```

The ML2/OVN driver uses the `ovn_l3_scheduler` (default `leastloaded`) to assign routers to gateway chassis. Multiple gateway chassis can be configured for HA — OVN uses BFD to detect failures and reroutes via the next available gateway chassis.

Inspect gateway chassis assignments:

```bash
ovn-sbctl list Gateway_Chassis
ovn-nbctl list Logical_Router_Port  # shows gateway_chassis field for external LRPs
```

## Migration from ML2/OVS to ML2/OVN

The official migration path uses the `neutron-ovn-migration` tooling. High-level steps:

1. **Prepare OVN infrastructure**: deploy OVN NB/SB databases (clustered or standalone), install `ovn-northd` and `ovn-controller` on all nodes
2. **Install ML2/OVN driver**: add the `python3-networking-ovn` package (part of neutron since Stein)
3. **Run pre-migration checks**: verify all agents are healthy, no ports stuck in non-ACTIVE state
4. **Run the migration script**:
   ```bash
   neutron-ovn-migration --neutron-conf /etc/neutron/neutron.conf \
     --ovn-conf /etc/neutron/plugins/ml2/ml2_conf.ini \
     --log-file /var/log/neutron/migration.log
   ```
   The script:
   - Populates the OVN NB DB from Neutron DB state
   - Updates `ml2_conf.ini` to use `mechanism_drivers = ovn`
   - Restarts neutron-server and switches the dataplane
5. **Verify**: check `ovn-nbctl show`, confirm all ports are bound, test connectivity
6. **Decommission**: stop and disable `neutron-openvswitch-agent`, `neutron-l3-agent`, `neutron-dhcp-agent` on all nodes

During migration, existing VMs are briefly disrupted (a few seconds per host as `ovn-controller` takes over from the OVS agent). Plan for a maintenance window.

## Diagnostic Commands

### Northbound DB — Logical Topology

```bash
# Show complete logical topology
ovn-nbctl show

# List all logical switches (Neutron networks)
ovn-nbctl list Logical_Switch

# List all logical switch ports (Neutron ports)
ovn-nbctl list Logical_Switch_Port

# List all logical routers
ovn-nbctl list Logical_Router

# List all logical router ports
ovn-nbctl list Logical_Router_Port

# List all NAT rules (floating IPs and SNAT)
ovn-nbctl list NAT

# List all DHCP options
ovn-nbctl list DHCP_Options

# List all ACLs for a port group
ovn-nbctl list ACL

# List all load balancers
ovn-nbctl list Load_Balancer
```

### Southbound DB — Physical Bindings

```bash
# Show physical topology (chassis and port bindings)
ovn-sbctl show

# List all registered chassis (hypervisors/gateways)
ovn-sbctl list Chassis

# List port bindings and which chassis hosts them
ovn-sbctl list Port_Binding

# List all logical flows (can be very verbose)
ovn-sbctl list Logical_Flow

# List MAC bindings (ARP/ND cache)
ovn-sbctl list MAC_Binding

# List BFD entries (used for gateway HA)
ovn-sbctl list BFD
```

### Trace Packet Path Through OVN

`ovn-trace` simulates how a packet traverses the OVN logical pipeline:

```bash
# Trace a TCP packet from VM to external IP (through router)
ovn-trace \
  --summary \
  <logical-switch-uuid> \
  'inport == "<lsp-uuid>", \
   eth.src = fa:16:3e:11:22:33, \
   eth.dst = fa:16:3e:aa:bb:cc, \
   ip4.src = 192.168.1.10, \
   ip4.dst = 8.8.8.8, \
   ip.ttl = 64, \
   tcp.src = 12345, \
   tcp.dst = 443'

# Trace with full logical flow output
ovn-trace --detailed <datapath-uuid> '<match-expression>'
```

### OVN Controller Status

```bash
# Show ovn-controller connection state and chassis UUID
ovs-vsctl get Open_vSwitch . external-ids

# Show what ovn-controller has programmed on a host
ovs-ofctl dump-flows br-int | grep -v OFPST

# Check ovn-controller logs
journalctl -u ovn-controller -f
```

### Common OVN Debug Queries

```bash
# Find which chassis a specific Neutron port is bound to
ovn-sbctl find Port_Binding external_ids:neutron:port_id=<port-uuid>

# Find all ports on a specific logical switch
ovn-nbctl lsp-list <logical-switch-name>

# Show router NAT rules
ovn-nbctl lr-nat-list <logical-router-name>

# Show DHCP options for a subnet
ovn-nbctl dhcp-options-list

# Show logical router routes
ovn-nbctl lr-route-list <logical-router-name>
```
