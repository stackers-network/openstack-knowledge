# Networking Decision Guide

## Comparison

| Feature | OVN | Open vSwitch (OVS) | LinuxBridge |
|---|---|---|---|
| **Recommended for** | New deployments | Existing OVS users | Simple/small environments |
| **Complexity** | Medium | Medium-High | Low |
| **Performance** | High (kernel datapath) | High (kernel datapath) | Moderate |
| **Distributed routing (DVR)** | Native (built-in) | Supported (extra config) | Not supported |
| **Hardware offload** | Supported | Supported | Not supported |
| **L3 HA** | Native | Requires VRRP config | Requires VRRP config |
| **Security groups** | OVN ACLs (fast) | OVS conntrack | iptables (slower at scale) |
| **DPDK support** | Yes | Yes | No |
| **Maintenance burden** | Low (single control plane) | Higher (separate L3/DHCP agents) | Low |
| **Upstream focus** | Primary — active development | Maintenance mode | Maintenance mode |
| **IPv6** | Full support | Full support | Full support |

## Recommendation by Use Case

### Production / New deployment → **OVN**

OVN is the recommended networking backend for all new OpenStack deployments as of 2025.1. It provides:
- Distributed routing without extra configuration
- Native L3 HA without VRRP
- Single control plane (no separate L3, DHCP, or metadata agents)
- Active upstream development and community focus

In `globals.yml`:
```yaml
neutron_plugin_agent: "ovn"
```

### Existing OVS deployment → **Keep OVS**

If you already run Open vSwitch, there's no urgent reason to migrate. OVS remains fully supported. Consider migrating to OVN during a major version upgrade if you want to reduce operational complexity.

In `globals.yml`:
```yaml
neutron_plugin_agent: "openvswitch"
enable_neutron_dvr: "yes"  # if distributed routing is needed
```

### Dev/test or very simple environments → **LinuxBridge**

LinuxBridge is the simplest option. No OVS/OVN infrastructure needed. Good for single-node or small lab deployments where advanced networking features aren't required.

In `globals.yml`:
```yaml
neutron_plugin_agent: "linuxbridge"
```

## Migration Paths

| From | To | Path |
|---|---|---|
| LinuxBridge → OVN | Requires full neutron migration. Not a simple config change — plan a maintenance window and follow the upstream migration guide. |
| OVS → OVN | Supported migration path documented upstream. Best done during a major version upgrade. Requires neutron DB migration. |
| OVN → OVS | Not recommended. Possible but requires manual neutron reconfiguration and DB changes. |

**Key rule:** Pick your networking backend at initial deployment. Migrations are possible but non-trivial and require downtime.
