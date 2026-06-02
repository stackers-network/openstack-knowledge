# Choose a Neutron ML2 Mechanism Driver

Neutron uses the ML2 plugin with a mechanism driver that handles L2 connectivity. The three main choices are ML2/OVS (Open vSwitch), ML2/OVN (Open Virtual Network), and ML2/Linux Bridge.

## Feature Comparison

| Feature | ML2/OVS | ML2/OVN | ML2/Linux Bridge |
|---|---|---|---|
| Performance | High | High | Medium |
| Distributed routing (DVR) | Yes (complex, DVR mode) | Yes (native, default) | No |
| Distributed DHCP | No (centralized DHCP agent) | Yes (native, no agent needed) | No |
| Security groups | iptables/conntrack or OVS firewall driver | Native OVN ACLs (no iptables) | iptables |
| DPDK support | Yes (OVS-DPDK) | Yes (OVN-DPDK) | No |
| SR-IOV support | Yes (with SR-IOV agent) | Yes (with OVN hardware offload) | No |
| Hardware offload | Yes (selected SmartNICs) | Yes (selected SmartNICs) | No |
| BGP dynamic routing | Yes (with neutron-dynamic-routing) | Yes (native BGP in OVN) | Limited |
| Trunk ports (802.1q) | Yes | Yes | Yes |
| QoS | Yes | Yes | Yes |
| IPv6 | Yes | Yes | Yes |
| Agent count | Many (OVS + L3 + DHCP + metadata per node) | Few (ovn-northd, ovn-controller per node) | Many (Linux Bridge + L3 + DHCP + metadata) |
| Centralized L3 agent | Yes (unless DVR) | No (distributed by default) | Yes |
| Operational complexity | Medium | Medium-High | Low |
| Community direction | Maintenance mode | Active development (default since Yoga) | Legacy, minimal new features |
| Stability | Very mature | Mature (production since Train) | Very mature |
| Debugging tools | ovs-vsctl, ovs-ofctl, ovs-appctl | ovn-nbctl, ovn-sbctl, ovs-vsctl | brctl, ip, iptables |

## Recommendation by Use Case

### Development and Learning
**Use ML2/Linux Bridge.** Simplest to understand, no OVS daemon required, easy to trace with standard Linux tools. Not suitable for production.

### Small to Medium Production Cloud (< 50 compute nodes)
**Use ML2/OVN.** Default since Yoga (2022.1). Distributed routing and DHCP are built in — no need to scale out L3 and DHCP agents. Fewer moving parts than OVS with DVR.

### Large Production Cloud (> 50 compute nodes)
**Use ML2/OVN.** OVN scales better than OVS+DVR because the control plane (ovn-northd) is centralized but the data plane is fully distributed. The OVN SB (Southbound) database synchronizes per-chassis state efficiently.

### Existing OVS Deployment
**Stay on ML2/OVS** unless you have a clear migration window. OVS is in maintenance mode but fully functional. Plan migration to OVN on the next major upgrade.

### DPDK Workloads (NFV, high throughput)
**Use ML2/OVN with OVN-DPDK** for new deployments. OVS-DPDK is also supported but OVN is the strategic direction. DPDK requires dedicated CPU cores (PMD threads) and huge pages.

### Hardware Offload (SmartNIC)
**Use ML2/OVN** with OVN hardware offload. Better integration with NVIDIA/Mellanox BlueField and similar NICs.

## Migration: OVS to OVN

Migrating an existing ML2/OVS cloud to ML2/OVN requires a maintenance window. There is no hot migration path.

### Pre-migration Checks

```bash
# Verify OVN packages are available
apt-cache show ovn-central ovn-host 2>/dev/null || dnf info ovn 2>/dev/null

# Check current ML2 config
grep "mechanism_drivers\|type_drivers" /etc/neutron/plugins/ml2/ml2_conf.ini

# List all agents that will be replaced
openstack network agent list --agent-type "Open vSwitch agent"
openstack network agent list --agent-type "L3 agent"
openstack network agent list --agent-type "DHCP agent"
```

### Migration Steps (Outline)

1. Deploy OVN (ovn-central on network/controller nodes, ovn-host on all nodes).
2. Run the Neutron OVN migration script to translate existing Neutron state to OVN NB/SB databases: `neutron-ovn-migration-mtu`.
3. Update `mechanism_drivers = ovn` in `ml2_conf.ini`.
4. Restart neutron-server.
5. Remove OVS agents and L3/DHCP agents from each compute node.
6. Start `ovn-controller` on each compute node.
7. Verify port bindings: `openstack port list | grep binding_failed`.
8. Validate connectivity from instances.

Refer to the official Neutron OVN migration documentation for the full procedure for your release.

### Post-migration Verification

```bash
# All ports should be bound
openstack port list -f json | jq '[.[] | select(."Binding VIF Type" == "binding_failed")] | length'
# Should return 0

# OVN controller status on each compute node
sudo ovn-appctl -t ovn-controller connection-status

# Check logical flows
sudo ovn-sbctl lflow-list | wc -l

# Verify no legacy agents remain
openstack network agent list --agent-type "Open vSwitch agent"
openstack network agent list --agent-type "DHCP agent"
openstack network agent list --agent-type "L3 agent"
```

## Configuration Quick Reference

### ML2/OVN (new deployment)

```ini
# /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
mechanism_drivers = ovn
type_drivers = local,flat,vlan,geneve
tenant_network_types = geneve

[ml2_type_geneve]
vni_ranges = 1:65536
max_header_size = 38

[ovn]
ovn_nb_connection = tcp:127.0.0.1:6641
ovn_sb_connection = tcp:127.0.0.1:6642
ovn_l3_scheduler = leastloaded
```

### ML2/OVS (existing deployment)

```ini
# /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
mechanism_drivers = openvswitch
type_drivers = local,flat,vlan,vxlan,gre
tenant_network_types = vxlan

[ml2_type_vxlan]
vni_ranges = 1:1000

[ovs]
bridge_mappings = provider:br-ex
local_ip = <TUNNEL_ENDPOINT_IP>

[securitygroup]
firewall_driver = openvswitch
```

### ML2/Linux Bridge

```ini
# /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
mechanism_drivers = linuxbridge
type_drivers = local,flat,vlan,vxlan
tenant_network_types = vxlan

[ml2_type_vxlan]
vni_ranges = 1:1000

[linux_bridge]
physical_interface_mappings = provider:eth1

[securitygroup]
firewall_driver = iptables
```
