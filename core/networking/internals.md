# Neutron Internals

## ML2 Plugin Architecture

ML2 uses three internal managers, each responsible for a different axis of the plugin framework:

### TypeManager

Manages network segment types and their ID allocations. When a network is created:

1. `TypeManager.create_network_segments()` is called
2. It iterates the requested (or default) `tenant_network_types`
3. The appropriate type driver (e.g., `VxlanTypeDriver`) allocates the next available VNI from the DB table `ml2_vxlan_allocations`
4. The segment record is written to `networksegments`

Type drivers maintain allocation tables:

| Type Driver | Allocation Table | ID Column |
|---|---|---|
| vlan | `ml2_vlan_allocations` | `vlan_id` |
| vxlan | `ml2_vxlan_allocations` | `vxlan_vni` |
| gre | `ml2_gre_allocations` | `gre_id` |
| geneve | `ml2_geneve_allocations` | `geneve_vni` |

### MechanismManager

Orchestrates mechanism drivers for every CRUD operation on networks, subnets, and ports. Calls are issued in two phases:

- **Precommit** (`create_network_precommit`, `create_port_precommit`, etc.) — runs inside the DB transaction. Must be fast. Used to validate and reserve resources.
- **Postcommit** (`create_network_postcommit`, `create_port_postcommit`, etc.) — runs after the DB transaction commits. Used to push config to agents/SDN controllers.

If a postcommit call fails, the DB change has already been committed. Mechanism drivers must handle this gracefully (idempotent operations, retry via RPC).

### ExtensionManager

Loads ML2 extension drivers (e.g., `port_security`, `qos`, `dns`) and hooks into port/network create/update operations to handle extension attributes.

## Port Binding Flow

Port binding is how Neutron selects the mechanism driver and programs the dataplane for a specific port on a specific host.

```
Nova calls PUT /v2.0/ports/<uuid>
  body: { "port": { "binding:host_id": "compute01.example.com" } }

  ML2Plugin.update_port()
    └── _bind_port_if_needed()
          └── _bind_port()
                ├── Calls each mechanism driver's bind_port(context) in order
                │   (driver list determined by mechanism_drivers in ml2_conf.ini)
                ├── Each driver checks:
                │   - Is the host known to this driver? (agent is registered?)
                │   - Does the agent support the network type?
                │   - Can the driver program this vif_type on this host?
                ├── Winning driver sets:
                │   - binding:vif_type  (e.g., "ovs", "vhostuser", "hw_veb")
                │   - binding:vif_details (e.g., OVS bridge name, vhostuser socket path)
                │   - binding:profile
                └── If no driver wins → port stays in PENDING state; Nova retries
```

When binding succeeds, the OVS agent (or ovn-controller) receives an RPC `port_update` notification and programs the local OVS bridge. The agent then calls `update_device_up` back to neutron-server, which sets `status = ACTIVE` and fires a Nova `network-changed` event.

## L2 Population

L2 population is a mechanism driver extension (`l2population`) that eliminates multicast/broadcast flooding for VXLAN tunnel traffic.

Without L2 population: when a VM sends its first ARP, the OVS agent floods it over all tunnels to all compute nodes.

With L2 population:

1. When a port binds on `compute02`, neutron-server calls `update_fdb_entries` on all other agents
2. Each agent programs a unicast OVS flow: for MAC `fa:16:3e:xx:xx:xx` send directly to VTEP `10.0.0.12` (compute02's tunnel IP)
3. ARP requests are answered locally by the OVS agent's ARP responder (when `arp_responder = True`)

This reduces east-west ARP flooding from O(N) tunnels to 0 broadcast tunnels after initial convergence.

## DHCP Agent and dnsmasq

The DHCP agent uses dnsmasq as the DHCP and DNS backend. For each DHCP-enabled network:

1. A Linux network namespace `qdhcp-<network-uuid>` is created
2. A tap interface `tapXXXXXX` is created in that namespace and plugged into `br-int` as a Neutron port (device owner `network:dhcp`)
3. dnsmasq is launched with:
   - `--conf-file` pointing to a generated config
   - `--hostsfile` with `<MAC>,<IP>,<hostname>` entries for every port on the network
   - `--optsfile` with DHCP options (router, DNS servers, domain name, static routes)
   - `--interface` pointing to the tap device inside the namespace

When a port is created or updated, the DHCP agent calls dnsmasq's `SIGHUP` to reload the hosts file without restarting.

### DHCP HA

Multiple DHCP agents can be scheduled to the same network. All agents serve DHCP simultaneously (active-active). dnsmasq on each agent has the same pool; duplicate offers are harmless because clients use the first valid OFFER they receive. The DB table `networkdhcpagentbindings` tracks which agents serve which networks.

```bash
# Check DHCP agent scheduling for a network
openstack network agent list --network private-net
```

## Metadata Agent — Proxy Chain

Instances retrieve metadata from `http://169.254.169.254/openstack/` or `http://169.254.169.254/latest/`. The proxy chain is:

```
Instance (no route to host)
  │  sends HTTP to 169.254.169.254:80
  ▼
iptables DNAT rule (inside qrouter-<uuid> or qdhcp-<uuid> namespace)
  │  redirects to 169.254.169.254:9697 (haproxy or socat in namespace)
  ▼
neutron-metadata-agent (listens on Unix socket or TCP)
  │  adds headers:
  │    X-Forwarded-For: <instance-fixed-ip>
  │    X-Neutron-Network-Id: <network-uuid>
  │    X-Neutron-Router-Id: <router-uuid>  (when coming via router namespace)
  ▼
Nova metadata API (nova-api process, port 8775)
  │  looks up instance by IP + network/router
  ▼
Returns instance metadata (hostname, ssh keys, user-data, etc.)
```

The `metadata_proxy_shared_secret` in `metadata_agent.ini` and Nova's `neutron.metadata_proxy_shared_secret` must match. This secret is used as an HMAC to prevent spoofed metadata requests.

## L3 Agent — Router Namespaces

For each router assigned to an L3 agent, a Linux network namespace `qrouter-<uuid>` is created containing:

- One `qg-XXXXXXXX` interface: connected to `br-ex` (or the external bridge), carries the router's gateway IP and the floating IPs
- One `qr-XXXXXXXX` interface per attached subnet: connected to `br-int`, carries the subnet's gateway IP
- `iptables` NAT rules:
  - SNAT rule: masquerade all traffic from tenant CIDRs leaving via `qg-XXXXXXXX`
  - DNAT/SNAT rules: for each floating IP mapping `203.0.113.15 → 192.168.1.50`
- Static routes from `router routes` and `subnet host_routes`

```bash
# List all router namespaces on a network node
ip netns list | grep qrouter

# Show interfaces and IPs inside a router namespace
ip netns exec qrouter-abc123 ip addr

# Show NAT rules for floating IPs
ip netns exec qrouter-abc123 iptables -t nat -L -n -v
```

### HA Routers (VRRP / keepalived)

When a router is created with `--ha`, neutron-server schedules it to `ha_network_agents` L3 agents (default 2). For each agent:

1. A `qrouter-<uuid>` namespace is created as above
2. A special HA network (type VXLAN, not visible to tenants) connects the agents
3. A `ha-XXXXXXXX` interface in the namespace connects to this HA network
4. keepalived runs inside the namespace using VRRP over the HA network
5. The MASTER agent holds the `qg-` IP and floating IPs; the BACKUP agent has them configured but not active

Failover: if the MASTER agent loses its keepalived heartbeat, the BACKUP promotes itself, claims the VIPs, and issues a gratuitous ARP. Failover time is typically 2–10 seconds.

conntrackd can be used alongside keepalived to replicate connection tracking state, enabling stateful NAT failover.

### Distributed Virtual Router (DVR)

DVR moves east-west and floating IP routing to compute nodes, eliminating the central network node as a bottleneck.

- On each compute node: a `qrouter-<uuid>` namespace handles floating IP DNAT/SNAT locally
- On the network node (dvr_snat mode): a `snat-<uuid>` namespace handles centralized SNAT for instances without floating IPs
- The OVS agent on each compute node creates a `qg`-style interface connected to `br-ex` (compute nodes need physical uplinks to the external network)

Agent modes:

| Node | `agent_mode` |
|---|---|
| Network node | `dvr_snat` |
| Compute node | `dvr` |
| Legacy (no DVR) | `legacy` |

## Security Group Implementation

### iptables Hybrid Driver

Each VM tap interface (`tapXXXXXX`) is placed in a Linux bridge (`qbrXXXXXX`). The bridge has two ports: the tap device (to the VM) and a veth pair peer (`qvbXXXXXX`) that connects to `br-int`.

iptables rules are applied to the `qbrXXXXXX` bridge using the FORWARD chain. conntrack is used for stateful tracking:

```
-A neutron-openvswi-i<tap> -m state --state RELATED,ESTABLISHED -j RETURN
-A neutron-openvswi-i<tap> -m state --state INVALID -j DROP
-A neutron-openvswi-i<tap> -p tcp --dport 80 -j RETURN
-A neutron-openvswi-i<tap> -j neutron-openvswi-sg-fallback
```

### OVS Firewall Driver (Recommended)

When `firewall_driver = openvswitch`, security group rules are implemented entirely in OVS flow rules on `br-int`. No Linux bridges or iptables chains are needed, which improves performance significantly.

Ports are placed in a "dead VLAN" until security group rules are programmed. Conntrack integration (`ct` action in OVS flows) provides stateful tracking:

```
# Allow established traffic back in
table=82, priority=65535, ct_state=+est-rel-rpl, actions=NORMAL

# Drop invalid tracked packets
table=82, priority=65535, ct_state=+inv, actions=DROP

# Apply security group ingress rules
table=82, priority=70, tcp,tp_dst=80, actions=ct(commit,table=0)
```

## RPC Between Server and Agents

Neutron uses oslo.messaging with RabbitMQ. Two communication patterns:

### Cast (fire and forget)

Used for agent notifications. Server casts `port_update`, `network_update`, `security_group_rules_for_devices` to an agent's topic queue. Agents process asynchronously.

```
neutron-server
  └── cast → topic: neutron.openvswitch_agent
              routing_key: compute01.example.com
```

### Call (synchronous RPC)

Used sparingly for operations where the server needs a response (e.g., `get_devices_details_list`). Agents also call back to the server (`update_device_up`, `update_device_down`).

### Agent Heartbeat

Agents report state every `report_interval` seconds (default 30 s) via `report_state()`. If an agent misses `agent_down_time` seconds (default 75 s), it is marked `DOWN` in the DB and shown as `XXX` in `openstack network agent list`.

## Database Models

Key tables in the Neutron DB (SQLAlchemy models in `neutron/db/models/`):

| Table | Key Columns | Notes |
|---|---|---|
| `networks` | `id`, `name`, `tenant_id`, `status`, `admin_state_up`, `shared` | Core L2 domain |
| `subnets` | `id`, `name`, `network_id`, `cidr`, `gateway_ip`, `ip_version`, `enable_dhcp` | IP address space |
| `ipallocationpools` | `subnet_id`, `first_ip`, `last_ip` | DHCP allocation ranges |
| `ports` | `id`, `name`, `network_id`, `mac_address`, `device_id`, `device_owner`, `status` | L2 attachment points |
| `ipallocations` | `port_id`, `subnet_id`, `ip_address` | Fixed IP assignments |
| `routers` | `id`, `name`, `tenant_id`, `status`, `admin_state_up`, `gw_port_id`, `distributed`, `ha` | L3 routers |
| `routerports` | `router_id`, `port_id`, `port_type` | Router ↔ port mappings |
| `floatingips` | `id`, `tenant_id`, `floating_network_id`, `floating_ip_address`, `fixed_port_id`, `fixed_ip_address`, `router_id` | NAT mapping |
| `securitygroups` | `id`, `name`, `tenant_id` | Security group containers |
| `securitygrouprules` | `id`, `security_group_id`, `direction`, `protocol`, `port_range_min`, `port_range_max`, `remote_ip_prefix`, `remote_group_id` | Firewall rules |
| `networksegments` | `id`, `network_id`, `network_type`, `physical_network`, `segmentation_id` | Segment type and ID |
| `ml2_port_bindings` | `port_id`, `host`, `vif_type`, `vif_details`, `driver`, `segment` | Binding state per port |
| `agents` | `id`, `binary`, `host`, `topic`, `admin_state_up`, `alive`, `heartbeat_timestamp` | Registered agents |

### Useful Debug Queries

```sql
-- Find all ports on a network that are not ACTIVE
SELECT p.name, p.status, b.vif_type, b.host
FROM ports p
LEFT JOIN ml2_port_bindings b ON b.port_id = p.id
WHERE p.network_id = '<network-uuid>'
  AND p.status != 'ACTIVE';

-- Find orphaned floating IPs (associated but instance is gone)
SELECT f.floating_ip_address, f.fixed_ip_address, f.fixed_port_id
FROM floatingips f
LEFT JOIN ports p ON p.id = f.fixed_port_id
WHERE f.fixed_port_id IS NOT NULL
  AND p.id IS NULL;

-- Show VXLAN VNI allocations
SELECT vxlan_vni, allocated FROM ml2_vxlan_allocations WHERE allocated = 1;
```
