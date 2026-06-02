# Open vSwitch (OVS) Deep-Dive

## Overview

Open vSwitch is a production-quality, multilayer virtual switch used as the dataplane in ML2/OVS Neutron deployments. In an OpenStack compute or network node, OVS is managed by the `neutron-openvswitch-agent`, which translates Neutron port/network state into OVS flow rules.

OVS consists of:

- `ovs-vswitchd`: the switch daemon; programs the kernel datapath
- `ovsdb-server`: stores configuration (bridges, ports, flow tables) in the OVS database
- Kernel datapath module (`openvswitch.ko`): high-performance packet forwarding in kernel space
- Userspace datapath: used with DPDK for even higher performance

## Bridge Architecture

A typical compute node runs three OVS bridges:

```
VM (qemu tap)
  │ tapXXXXXX
  ▼
br-int  (integration bridge)
  │  patch-tun or patch-int
  ▼
br-tun  (tunnel bridge)
  │  vxlan-<remote-vtep>  (one per remote compute node)
  ▼
Overlay network (UDP encapsulation)

br-int
  │  patch-ext (or phy-br-ex veth pair)
  ▼
br-ex  (provider/external bridge)
  │  eth1 (physical uplink)
  ▼
Physical network
```

### br-int — Integration Bridge

`br-int` is the central switching fabric for a compute node. Every VM port, router port, DHCP port, and metadata port plugs into `br-int`. It operates at the local VLAN level — each Neutron network is assigned a local VLAN ID that is only significant within this node.

Key OVS port types on br-int:

| Port | Type | Purpose |
|---|---|---|
| `tapXXXXXX` | internal/tap | VM network interface |
| `qr-XXXXXXXX` | internal | Router interface (L3 agent) |
| `qdhcp-XXXXXXXX` | internal | DHCP tap |
| `int-br-ex` | patch | Patch to br-ex (for provider networks) |
| `patch-tun` | patch | Patch to br-tun (for overlay networks) |

Flow tables on br-int (simplified):

```
Table 0  (Ingress classification)
  priority=65535, in_port=<local>, actions=NORMAL          # from local ports
  priority=1, actions=drop

Table 23 (Ingress from tunnel bridge)
  dl_vlan=<tun-vlan>, actions=mod_vlan_vid:<local-vlan>,NORMAL

Table 24 (Ingress from external bridge — for provider networks)
  in_port=int-br-ex, dl_vlan=<physnet-vlan>, actions=mod_vlan_vid:<local-vlan>,NORMAL

Table 60 (L2 forwarding)
  priority=2, dl_vlan=<local-vlan>, dl_dst=<mac>, actions=output:<port>
  priority=1, dl_vlan=<local-vlan>, actions=FLOOD
```

### br-tun — Tunnel Bridge

`br-tun` handles VXLAN (or GRE) encapsulation and decapsulation between compute nodes. Each remote compute node has a dedicated tunnel port:

```
vxlan-0a000001  →  10.0.0.1 (compute01 VTEP)
vxlan-0a000002  →  10.0.0.2 (compute02 VTEP)
```

Flow tables on br-tun:

```
Table 0  (Ingress classification)
  priority=1, in_port=patch-int, actions=goto_table:1     # from br-int
  priority=1, in_port=<tun-port>, actions=goto_table:4    # from remote

Table 1  (Tunnel encap — from br-int outbound)
  dl_dst=ff:ff:ff:ff:ff:ff, actions=goto_table:2          # broadcast → flood
  dl_dst=fa:16:3e:xx:xx:xx, actions=goto_table:20         # unicast → lookup

Table 2  (Flood — with l2population disabled)
  dl_vlan=<local-vlan>, actions=strip_vlan,set_tunnel:<vni>,
           output:vxlan-0a000001,output:vxlan-0a000002

Table 20 (Unicast — with l2population enabled)
  priority=2, dl_vlan=<local-vlan>, dl_dst=fa:16:3e:ab:cd:ef,
    actions=strip_vlan,set_tunnel:<vni>,output:vxlan-0a000002

Table 4  (Ingress from tunnel — decap)
  tun_id=<vni>, actions=mod_vlan_vid:<local-vlan>,goto_table:10

Table 10 (Ingress security / learn)
  actions=learn(table=20,...),output:patch-int
```

### br-ex — Provider/External Bridge

`br-ex` connects OVS to physical network interfaces for provider networks and external gateway traffic. It has a physical uplink port (e.g., `eth1`) and a patch port to `br-int`.

For provider networks, OVS agent configures `bridge_mappings = physnet1:br-ex` and programs flows to translate between the physical VLAN and the local internal VLAN.

```bash
# Show br-ex configuration
ovs-vsctl show
ovs-ofctl dump-flows br-ex
```

## OVS Agent Operation

The `neutron-openvswitch-agent` runs on every compute/network node. Its main loop:

1. **Setup**: creates bridges, wires them together with patch ports, programs base flows
2. **Polling**: uses `ovsdb-monitor` (or periodic polling) to detect new/removed ports on `br-int`
3. **Port processing**: for each new port, looks up its Neutron port record via RPC `get_devices_details_list`
4. **Flow programming**: writes OVS flows for the port's local VLAN, security group rules, DHCP/ARP rules
5. **RPC handling**: listens for `port_update`, `network_update`, `security_group_rules_for_devices` from neutron-server
6. **Heartbeat**: calls `report_state` every 30 s

Key configuration options in `openvswitch_agent.ini`:

```ini
[ovs]
local_ip = 10.0.0.11          # VTEP IP for VXLAN tunnels
bridge_mappings = physnet1:br-ex
tunnel_types = vxlan
ovsdb_connection = tcp:127.0.0.1:6640  # OVSDB server

[agent]
l2_population = True          # Enable l2population mechanism driver optimization
arp_responder = True          # Answer ARP locally without flooding
prevent_arp_spoofing = True   # Drop ARP with spoofed source IPs
polling_interval = 2          # Seconds between port scan cycles
minimize_polling = True       # Use OVSDB monitor instead of polling

[securitygroup]
firewall_driver = openvswitch  # OVS-native conntrack firewall (recommended)
# firewall_driver = iptables_hybrid  # Legacy iptables-based approach
```

## Security Group Firewall Drivers

### OVS Firewall Driver (Recommended)

Security group rules are translated into OVS flows with conntrack (`ct`) actions. No Linux bridges or iptables. Works entirely in the kernel OVS datapath.

Flow table layout for OVS firewall:

| Table | Purpose |
|---|---|
| 71 | Ingress drop (default deny ingress to unknown ports) |
| 72 | Accept established inbound traffic (ct_state=+est) |
| 73 | Ingress rules (match security group allow rules) |
| 81 | Egress drop (default deny egress from unknown ports) |
| 82 | Accept established outbound traffic |
| 83 | Egress rules |

Example flows for a port with `tcp:80` ingress rule:

```
table=73, priority=77, ct_state=+new, tcp, nw_dst=192.168.1.50, tp_dst=80
  actions=ct(commit,zone=<port-zone>,table=0)

table=72, priority=77, ct_state=+est-rel-rpl,
  actions=output:<port>

table=71, priority=10, actions=drop
```

### iptables Hybrid Driver (Legacy)

Each VM port has:
- A Linux bridge `qbrXXXXXX` between the tap device and `br-int`
- iptables FORWARD chain rules applied to the bridge
- conntrack for stateful matching

This approach is less efficient because every packet must traverse iptables in addition to OVS. Use the OVS firewall driver for new deployments.

## DPDK Support

OVS-DPDK bypasses the kernel entirely using DPDK poll-mode drivers (PMDs) for line-rate performance.

Requirements:
- CPUs with hugepages enabled (`default_hugepagesz=1G hugepagesz=1G hugepages=8` in kernel cmdline)
- NUMA-aware memory allocation
- DPDK-compatible NIC (Intel i40e, mlx5, etc.)
- `ovs-vswitchd` built with DPDK support (`ovs-vswitchd --version` shows DPDK version)

Configure hugepages for OVS-DPDK:

```bash
# /etc/default/grub
GRUB_CMDLINE_LINUX="default_hugepagesz=1G hugepagesz=1G hugepages=8 iommu=pt intel_iommu=on"

# /etc/dpdk.conf (or equivalent)
EAL_ARGS="-l 2,3,4,5 --socket-mem 2048,2048"
```

Configure OVS for DPDK:

```bash
ovs-vsctl set Open_vSwitch . other_config:dpdk-init=true
ovs-vsctl set Open_vSwitch . other_config:dpdk-socket-mem="2048,2048"
ovs-vsctl set Open_vSwitch . other_config:pmd-cpu-mask=0x1c  # PMD threads on cores 2,3,4

# Create DPDK bridge
ovs-vsctl add-br br-int
ovs-vsctl set Bridge br-int datapath_type=netdev

# Add DPDK physical port
ovs-vsctl add-port br-int dpdk0 -- set Interface dpdk0 type=dpdk options:dpdk-devargs=0000:01:00.0

# Add vhost-user port for VMs
ovs-vsctl add-port br-int vhost-user0 \
  -- set Interface vhost-user0 type=dpdkvhostuserclient \
     options:vhost-server-path=/var/run/openvswitch/vhost-user0
```

DPDK bond for link aggregation:

```bash
ovs-vsctl add-bond br-ex dpdkbond dpdk0 dpdk1 \
  bond_mode=active-backup \
  -- set Interface dpdk0 type=dpdk options:dpdk-devargs=0000:01:00.0 \
  -- set Interface dpdk1 type=dpdk options:dpdk-devargs=0000:01:00.1
```

Nova virt driver configuration for vhost-user ports:

```ini
# nova.conf
[DEFAULT]
cpu_shared_set = 0,1
cpu_dedicated_set = 2,3,4,5

[libvirt]
virt_type = kvm
```

## Diagnostic Commands

### Show Bridge and Port Configuration

```bash
# Show all bridges, ports, and interfaces
ovs-vsctl show

# List all bridges
ovs-vsctl list-br

# List ports on a specific bridge
ovs-vsctl list-ports br-int

# Show detailed interface record
ovs-vsctl list Interface tapabcdef12
```

### Inspect Flow Tables

```bash
# Dump all flows on br-int (human-readable)
ovs-ofctl dump-flows br-int

# Dump flows on br-tun sorted by table and priority
ovs-ofctl dump-flows br-tun --sort

# Dump flows matching a specific MAC
ovs-ofctl dump-flows br-int "dl_dst=fa:16:3e:ab:cd:ef"

# Dump flows for a specific table
ovs-ofctl dump-flows br-int "table=0"

# Dump flows with byte/packet counters
ovs-ofctl dump-flows -O OpenFlow13 br-int
```

### Inspect the Forwarding Database

```bash
# Show MAC → port mappings learned by OVS
ovs-appctl fdb/show br-int

# Show VXLAN tunnel endpoints
ovs-appctl tnl/neigh/show

# Show all tunnel ports and their remote IPs
ovs-appctl dpif/show
```

### Conntrack Inspection

```bash
# Show conntrack table entries
ovs-appctl dpctl/dump-conntrack

# Show conntrack entries for a specific zone
ovs-appctl dpctl/dump-conntrack "zone=<port-zone>"
```

### Packet Tracing

```bash
# Trace a packet through OVS (simulated)
ovs-appctl ofproto/trace br-int \
  "in_port=<port-number>,dl_src=fa:16:3e:11:22:33,dl_dst=fa:16:3e:44:55:66,dl_type=0x0800,nw_src=192.168.1.10,nw_dst=192.168.1.20"
```

### OVSDB Queries

```bash
# Show all OVS configuration (raw)
ovsdb-client dump

# Monitor OVSDB changes in real time
ovsdb-client monitor Open_vSwitch

# Show Port table
ovs-vsctl list Port

# Show Interface table with statistics
ovs-vsctl list Interface
```

### Performance Inspection

```bash
# Show PMD thread CPU usage (DPDK)
ovs-appctl dpif-netdev/pmd-stats-show

# Show PMD thread RX/TX queues
ovs-appctl dpif-netdev/pmd-rxq-show

# Show datapath statistics
ovs-dpctl show
ovs-dpctl dump-flows
```

## Common Issues and Fixes

| Symptom | Likely Cause | Fix |
|---|---|---|
| Port stuck in `BUILD` status | OVS agent not running or not receiving RPC | Check `neutron-openvswitch-agent` logs; verify RabbitMQ connectivity |
| VM cannot reach another VM on same compute node | br-int flows missing for local VLAN | Restart OVS agent; check `ovs-ofctl dump-flows br-int table=0` |
| VM cannot reach VM on another compute node | Tunnel not established or VNI mismatch | Check `ovs-vsctl show` for tunnel ports; verify `local_ip` matches |
| Security group rules not applied | OVS firewall driver misconfigured or iptables hybrid used | Check `firewall_driver` in agent config; inspect flows in tables 71-83 |
| ARP flooding causing performance issues | l2population or arp_responder disabled | Enable both in `openvswitch_agent.ini`; ensure l2population in mechanism_drivers |
