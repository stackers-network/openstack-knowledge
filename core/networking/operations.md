# Neutron Operations

## Networks

### Create a Tenant Network (VXLAN Overlay)

```bash
openstack network create \
  --provider-network-type vxlan \
  private-net
```

Neutron allocates the next available VNI from the `vni_ranges` pool automatically.

### Create a Provider Network (Flat — External)

```bash
openstack network create \
  --provider-physical-network physnet1 \
  --provider-network-type flat \
  --external \
  provider-net
```

`physnet1` must appear in the OVS agent's `bridge_mappings` (e.g., `physnet1:br-ex`).

### Create a Provider Network (VLAN)

```bash
openstack network create \
  --provider-physical-network physnet1 \
  --provider-network-type vlan \
  --provider-segment 200 \
  --share \
  vlan200-net
```

### List, Show, Update, Delete Networks

```bash
openstack network list
openstack network show private-net
openstack network set --description "Application tier" private-net
openstack network delete private-net
```

### Share a Network via RBAC

```bash
# Share private-net with project "acme"
openstack network rbac create \
  --type network \
  --action access_as_shared \
  --target-project acme \
  private-net
```

## Subnets

### Create a Subnet on a Tenant Network

```bash
openstack subnet create \
  --network private-net \
  --subnet-range 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --dns-nameserver 8.8.8.8 \
  --dns-nameserver 8.8.4.4 \
  private-subnet
```

### Create a Subnet on a Provider Network (No DHCP)

```bash
openstack subnet create \
  --network provider-net \
  --subnet-range 203.0.113.0/24 \
  --gateway 203.0.113.1 \
  --no-dhcp \
  provider-subnet
```

### Create a Subnet with Allocation Pools and Host Routes

```bash
openstack subnet create \
  --network private-net \
  --subnet-range 10.0.0.0/24 \
  --gateway 10.0.0.1 \
  --allocation-pool start=10.0.0.100,end=10.0.0.200 \
  --host-route destination=10.1.0.0/24,gateway=10.0.0.254 \
  --dns-nameserver 1.1.1.1 \
  app-subnet
```

### Create an IPv6 Subnet (SLAAC)

```bash
openstack subnet create \
  --network private-net \
  --subnet-range fd00:192:168:1::/64 \
  --ip-version 6 \
  --ipv6-ra-mode slaac \
  --ipv6-address-mode slaac \
  private-subnet-v6
```

### List, Show, Update, Delete Subnets

```bash
openstack subnet list --network private-net
openstack subnet show private-subnet
openstack subnet set --dns-nameserver 9.9.9.9 private-subnet
openstack subnet delete private-subnet
```

## Routers

### Create a Router and Wire It

```bash
openstack router create main-router
openstack router add subnet main-router private-subnet
openstack router set --external-gateway provider-net main-router
```

### Create an HA Router

```bash
openstack router create --ha ha-router
openstack router add subnet ha-router app-subnet
openstack router set --external-gateway provider-net ha-router
```

### Create a Distributed Router (DVR)

```bash
openstack router create --distributed distributed-router
```

DVR requires `neutron.conf` to have `router_distributed = True` or setting it per-router as above.

### Manage Router Interfaces

```bash
# Add a subnet interface
openstack router add subnet main-router another-subnet

# Add a specific port as router interface
openstack router add port main-router <port-uuid>

# Remove a subnet interface
openstack router remove subnet main-router another-subnet
```

### Show Router Details

```bash
openstack router show main-router
openstack router list
```

### Inspect a Router's Network Namespace (on the network/compute node)

```bash
# Find the namespace
ip netns list | grep qrouter

# Inspect routing table inside the namespace
ip netns exec qrouter-<uuid> ip route

# Test connectivity from within the namespace
ip netns exec qrouter-<uuid> ping 8.8.8.8
```

## Floating IPs

### Allocate a Floating IP

```bash
openstack floating ip create provider-net
```

Output includes the allocated IP, e.g. `203.0.113.15`.

### Associate a Floating IP with an Instance

```bash
openstack server add floating ip my-instance 203.0.113.15
```

### Associate a Floating IP with a Specific Port

```bash
openstack floating ip set --port <port-uuid> 203.0.113.15
```

### Disassociate and Release

```bash
openstack server remove floating ip my-instance 203.0.113.15
openstack floating ip delete 203.0.113.15
```

### List Floating IPs

```bash
openstack floating ip list
openstack floating ip list --status ACTIVE
openstack floating ip list --project acme
```

## Security Groups

### Create a Security Group

```bash
openstack security group create web-sg \
  --description "Security group for web tier"
```

### Add Ingress Rules

```bash
# Allow HTTP from anywhere
openstack security group rule create web-sg \
  --protocol tcp \
  --dst-port 80 \
  --remote-ip 0.0.0.0/0 \
  --ingress

# Allow HTTPS from anywhere
openstack security group rule create web-sg \
  --protocol tcp \
  --dst-port 443 \
  --remote-ip 0.0.0.0/0 \
  --ingress

# Allow ICMP (ping) from anywhere
openstack security group rule create web-sg \
  --protocol icmp \
  --remote-ip 0.0.0.0/0 \
  --ingress

# Allow SSH from a management CIDR
openstack security group rule create web-sg \
  --protocol tcp \
  --dst-port 22 \
  --remote-ip 10.10.0.0/24 \
  --ingress
```

### Allow Traffic from Another Security Group

```bash
# Allow PostgreSQL access from instances in the app-sg group
openstack security group rule create db-sg \
  --protocol tcp \
  --dst-port 5432 \
  --remote-group app-sg \
  --ingress
```

### Allow All UDP (e.g., for DNS)

```bash
openstack security group rule create web-sg \
  --protocol udp \
  --dst-port 53 \
  --remote-ip 0.0.0.0/0 \
  --ingress
```

### List, Show, Delete Rules

```bash
openstack security group rule list web-sg
openstack security group rule show <rule-uuid>
openstack security group rule delete <rule-uuid>
openstack security group delete web-sg
```

### Apply a Security Group to an Instance Port

```bash
openstack server add security group my-instance web-sg
openstack server remove security group my-instance default
```

## Ports

### Create a Port

```bash
openstack port create \
  --network private-net \
  --fixed-ip subnet=private-subnet,ip-address=192.168.1.50 \
  --security-group web-sg \
  app-port
```

### Create a Port with a Specific MAC

```bash
openstack port create \
  --network private-net \
  --mac-address fa:16:3e:ab:cd:ef \
  reserved-port
```

### Create a Port with Allowed Address Pairs (e.g., VIP)

```bash
openstack port create \
  --network private-net \
  --allowed-address ip-address=192.168.1.200 \
  vip-port
```

### Disable Port Security on a Port

```bash
openstack port set --no-security-group --disable-port-security router-port
```

### List, Show, Update, Delete Ports

```bash
openstack port list --network private-net
openstack port list --server my-instance
openstack port show app-port
openstack port set --description "Application server NIC" app-port
openstack port delete app-port
```

### Show Port Binding Details

```bash
openstack port show app-port -c binding_host_id -c binding_vif_type -c binding_vnic_type -c status
```

## Agent Operations

### List All Agents and Their State

```bash
openstack network agent list
openstack network agent list --agent-type "OVS agent"
openstack network agent list --host compute01.example.com
```

### Show Agent Details

```bash
openstack network agent show <agent-uuid>
```

### Check Which Networks a DHCP Agent Serves

```bash
openstack network agent list --network private-net
```

### Schedule a Network to a Specific DHCP Agent

```bash
openstack network add agent <dhcp-agent-uuid> private-net
openstack network remove agent <dhcp-agent-uuid> private-net
```

### Schedule a Router to a Specific L3 Agent

```bash
openstack network agent add router <l3-agent-uuid> main-router
openstack network agent remove router <l3-agent-uuid> main-router
```

## RBAC Policies

```bash
# Allow another project to use a network
openstack network rbac create \
  --type network \
  --action access_as_shared \
  --target-project <project-uuid> \
  private-net

# Allow another project to use a network as an external network
openstack network rbac create \
  --type network \
  --action access_as_external \
  --target-project <project-uuid> \
  provider-net

openstack network rbac list
openstack network rbac show <rbac-uuid>
openstack network rbac delete <rbac-uuid>
```

## Configuration Reference

### `/etc/neutron/neutron.conf`

```ini
[DEFAULT]
core_plugin = ml2
service_plugins = router,qos,trunk,segments,port_forwarding
auth_strategy = keystone
transport_url = rabbit://openstack:rabbitpass@controller01:5672/
api_workers = 4
rpc_workers = 4
router_distributed = False
allow_overlapping_ips = True
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
dns_domain = openstack.internal.

[database]
connection = mysql+pymysql://neutron:neutrondbpass@controller01/neutron?charset=utf8mb4

[keystone_authtoken]
www_authenticate_uri = http://controller01:5000
auth_url = http://controller01:5000
memcached_servers = controller01:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = neutronservicepass

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp

[oslo_messaging_rabbit]
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
heartbeat_timeout_threshold = 60
```

### `/etc/neutron/plugins/ml2/ml2_conf.ini`

```ini
[ml2]
type_drivers = flat,vlan,vxlan,geneve
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security,qos,dns

[ml2_type_flat]
flat_networks = physnet1

[ml2_type_vlan]
network_vlan_ranges = physnet1:100:299,physnet2:300:499

[ml2_type_vxlan]
vni_ranges = 1:65535
vxlan_group = 239.1.1.1

[ml2_type_gre]
tunnel_id_ranges = 1:1000

[ml2_type_geneve]
vni_ranges = 1:65535
max_header_size = 38

[securitygroup]
enable_security_group = True
firewall_driver = openvswitch
# For iptables hybrid mode:
# firewall_driver = iptables_hybrid
```

### `/etc/neutron/plugins/ml2/openvswitch_agent.ini`

```ini
[ovs]
local_ip = 10.0.0.11
bridge_mappings = physnet1:br-ex
tunnel_types = vxlan

[agent]
tunnel_types = vxlan
l2_population = True
arp_responder = True
prevent_arp_spoofing = True

[securitygroup]
enable_security_group = True
firewall_driver = openvswitch
```

### `/etc/neutron/plugins/ml2/ml2_conf.ini` — OVN section

```ini
[ml2]
type_drivers = local,flat,vlan,geneve
tenant_network_types = geneve
mechanism_drivers = ovn
extension_drivers = port_security,qos,dns

[ml2_type_geneve]
vni_ranges = 1:65535
max_header_size = 38

[ml2_type_flat]
flat_networks = physnet1

[ml2_type_vlan]
network_vlan_ranges = physnet1:100:299

[ovn]
ovn_nb_connection = tcp:192.168.0.10:6641
ovn_sb_connection = tcp:192.168.0.10:6642
ovn_l3_scheduler = leastloaded
ovn_metadata_enabled = True
enable_distributed_floating_ip = True

[securitygroup]
enable_security_group = True
```

### `/etc/neutron/l3_agent.ini`

```ini
[DEFAULT]
interface_driver = openvswitch
external_network_bridge =
# Leave external_network_bridge empty when using bridge_mappings in OVS agent

[l3_agent]
agent_mode = legacy
# agent_mode = dvr           # on compute nodes for DVR
# agent_mode = dvr_snat      # on the network node for DVR+SNAT
ha_confs_dir = /var/lib/neutron/ha_confs
```

### `/etc/neutron/dhcp_agent.ini`

```ini
[DEFAULT]
interface_driver = openvswitch
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
force_metadata = True

[agent]
root_helper = sudo neutron-rootwrap /etc/neutron/rootwrap.conf
```

### `/etc/neutron/metadata_agent.ini`

```ini
[DEFAULT]
nova_metadata_host = controller01
nova_metadata_port = 8775
metadata_proxy_shared_secret = metadata-proxy-secret-change-me
metadata_workers = 2
```
