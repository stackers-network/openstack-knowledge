# Advanced Neutron Features

## Trunking (VLAN-Aware VMs)

Trunk ports allow a single VM NIC to carry multiple Neutron networks, each tagged with a different VLAN ID inside the VM. This is essential for NFV/VNF deployments, SR-IOV with VLAN tagging, and nested virtualization.

### Concepts

| Term | Meaning |
|---|---|
| Trunk | The logical container; represents the VM's trunk port |
| Parent port | The Neutron port that attaches the trunk to the VM's NIC |
| Sub-port | A Neutron port on a different network, carried as a VLAN-tagged sub-interface inside the VM |
| Segmentation type | Currently only `vlan` is supported for sub-ports |
| Segmentation ID | The 802.1Q VLAN ID the VM sees for that sub-port |

### Create a Trunk

```bash
# 1. Create the parent port (untagged — represents the native/management VLAN)
openstack port create \
  --network management-net \
  trunk-parent-port

# 2. Create sub-ports on other networks
openstack port create --network app-net app-subport
openstack port create --network storage-net storage-subport

# 3. Create the trunk with sub-ports
openstack network trunk create \
  --parent-port trunk-parent-port \
  --subport port=app-subport,segmentation-type=vlan,segmentation-id=100 \
  --subport port=storage-subport,segmentation-type=vlan,segmentation-id=200 \
  my-trunk

# 4. Boot a VM on the parent port
openstack server create \
  --image ubuntu-22.04 \
  --flavor m1.medium \
  --port trunk-parent-port \
  my-vnf
```

### Add and Remove Sub-ports

```bash
openstack network trunk add subport my-trunk \
  --subport port=<new-port-uuid>,segmentation-type=vlan,segmentation-id=300

openstack network trunk remove subport my-trunk \
  --subport port=<port-uuid>
```

### List and Show Trunks

```bash
openstack network trunk list
openstack network trunk show my-trunk
```

### Inside the VM

The VM sees its parent interface (e.g., `eth0`) as the native VLAN, and creates VLAN sub-interfaces for each sub-port:

```bash
# Inside the VM
ip link add link eth0 name eth0.100 type vlan id 100
ip link add link eth0 name eth0.200 type vlan id 200
ip addr add 10.0.100.10/24 dev eth0.100
ip link set eth0.100 up
```

## QoS Policies

Neutron QoS allows bandwidth limiting, minimum bandwidth guarantees, and DSCP marking on ports and networks.

### Create a QoS Policy

```bash
openstack network qos policy create bw-limit-policy \
  --description "Limit to 100 Mbps egress, 50 Mbps ingress"
```

### Add Bandwidth Limit Rules

```bash
# Egress limit (from VM perspective): max 100 Mbps, burst 200 Mbps
openstack network qos rule create bw-limit-policy \
  --type bandwidth-limit \
  --max-kbps 102400 \
  --max-burst-kbits 204800 \
  --egress

# Ingress limit (traffic coming into the VM): max 50 Mbps
openstack network qos rule create bw-limit-policy \
  --type bandwidth-limit \
  --max-kbps 51200 \
  --max-burst-kbits 102400 \
  --ingress
```

### Add Minimum Bandwidth Rule (SR-IOV / hardware scheduling)

```bash
openstack network qos policy create min-bw-policy

openstack network qos rule create min-bw-policy \
  --type minimum-bandwidth \
  --min-kbps 10240 \
  --egress
```

Minimum bandwidth rules require a Nova/Placement integration to work end-to-end. The scheduler ensures the compute node's NIC can satisfy the minimum.

### Add DSCP Marking Rule

```bash
openstack network qos policy create dscp-policy

openstack network qos rule create dscp-policy \
  --type dscp-marking \
  --dscp-mark 14
```

DSCP value 14 = AF13 (Assured Forwarding class 1, drop precedence 3). Standard values: 0 (BE), 8 (CS1), 10 (AF11), 46 (EF).

### Apply a QoS Policy to a Port

```bash
openstack port set --qos-policy bw-limit-policy my-port
```

### Apply a QoS Policy to a Network (Affects All Ports)

```bash
openstack network set --qos-policy bw-limit-policy private-net
```

### List QoS Policies and Rules

```bash
openstack network qos policy list
openstack network qos policy show bw-limit-policy
openstack network qos rule list bw-limit-policy
```

### QoS Driver Configuration

Enable QoS in `neutron.conf`:

```ini
[DEFAULT]
service_plugins = router,qos,trunk,segments
```

Enable QoS extension in `ml2_conf.ini`:

```ini
[ml2]
extension_drivers = port_security,qos
```

The OVS agent implements QoS using `tc` (Linux traffic control) for ingress policing and OVS meter actions or `tc` HTB queues for egress. The OVN backend implements QoS natively via `QoS` table entries in the OVN NB DB.

## Network Segments

Network segments allow a single Neutron network to span multiple physical segments (e.g., different VLAN ranges on different physical networks). This is useful for stretching a network across availability zones.

### Create a Multi-Segment Network

```bash
# Create the base network
openstack network create multi-segment-net

# Add segments manually
openstack network segment create \
  --network multi-segment-net \
  --network-type vlan \
  --physical-network physnet1 \
  --segment 201 \
  segment-physnet1

openstack network segment create \
  --network multi-segment-net \
  --network-type vlan \
  --physical-network physnet2 \
  --segment 301 \
  segment-physnet2
```

### Create Subnets on Specific Segments

```bash
openstack subnet create \
  --network multi-segment-net \
  --network-segment segment-physnet1 \
  --subnet-range 10.10.1.0/24 \
  --gateway 10.10.1.1 \
  subnet-physnet1

openstack subnet create \
  --network multi-segment-net \
  --network-segment segment-physnet2 \
  --subnet-range 10.10.2.0/24 \
  --gateway 10.10.2.1 \
  subnet-physnet2
```

Port binding will use the segment that corresponds to the compute node's available physical networks. Nova integrates with the segments API to schedule VMs to compute nodes that can reach the required segment.

### List Segments

```bash
openstack network segment list
openstack network segment list --network multi-segment-net
openstack network segment show segment-physnet1
```

## BGP Dynamic Routing

`neutron-dynamic-routing` enables Neutron to advertise floating IP and tenant network prefixes to upstream BGP routers. This eliminates the need for static routes in the physical network for floating IPs.

### Install and Enable

```bash
pip install neutron-dynamic-routing
```

In `neutron.conf`:

```ini
[DEFAULT]
service_plugins = router,qos,trunk,bgp
```

### Create a BGP Speaker

```bash
openstack bgp speaker create \
  --ip-version 4 \
  --local-as 65000 \
  main-bgp-speaker
```

### Add Peers

```bash
openstack bgp peer create \
  --peer-ip 203.0.113.1 \
  --remote-as 65001 \
  --auth-type md5 \
  --password bgp-peer-secret \
  upstream-router-peer

openstack bgp speaker add peer main-bgp-speaker upstream-router-peer
```

### Associate a Network

The BGP speaker advertises floating IPs allocated from provider networks that are associated with it:

```bash
openstack bgp speaker add network main-bgp-speaker provider-net
```

To advertise tenant network prefixes:

```bash
openstack bgp speaker create \
  --ip-version 4 \
  --local-as 65000 \
  --advertise-tenant-networks \
  --advertise-floating-ip-host-routes \
  main-bgp-speaker
```

### List Advertised Routes

```bash
openstack bgp speaker list advertised routes main-bgp-speaker
```

### Schedule the BGP Speaker to an Agent

```bash
openstack bgp dragent add speaker <dragent-uuid> main-bgp-speaker
```

`neutron-bgp-dragent` (the dynamic routing agent) runs on the network node and maintains the BGP sessions using the BIRD or GoBGP backend.

## VPNaaS (IPsec VPN)

VPNaaS provides site-to-site IPsec VPN connectivity between Neutron routers and external VPN endpoints.

### Prerequisites

```bash
pip install neutron-vpnaas
```

In `neutron.conf`:

```ini
[DEFAULT]
service_plugins = router,qos,trunk,vpnaas
```

In `vpnaas_agent.ini`:

```ini
[DEFAULT]
ipsec_status_check_interval = 60

[ipsec]
ipsec_helper = neutron_vpnaas.services.vpn.device_drivers.strongswan_ipsec.StrongSwanDriver
```

### Create VPN Service

```bash
openstack vpn service create \
  --router main-router \
  --subnet private-subnet \
  main-vpn-service
```

### Create IKE and IPsec Policies

```bash
openstack vpn ike policy create \
  --auth-algorithm sha256 \
  --encryption-algorithm aes-256 \
  --ike-version v2 \
  --pfs group14 \
  --lifetime units=seconds,value=86400 \
  ike-policy-aes256

openstack vpn ipsec policy create \
  --auth-algorithm sha256 \
  --encryption-algorithm aes-256 \
  --pfs group14 \
  --lifetime units=seconds,value=3600 \
  ipsec-policy-aes256
```

### Create a Site Connection

```bash
openstack vpn ipsec site connection create \
  --vpnservice main-vpn-service \
  --ikepolicy ike-policy-aes256 \
  --ipsecpolicy ipsec-policy-aes256 \
  --peer-address 198.51.100.1 \
  --peer-id 198.51.100.1 \
  --peer-cidr 172.16.0.0/24 \
  --psk vpn-pre-shared-key-change-me \
  --initiator bi-directional \
  site-to-hq
```

`--peer-cidr` is the remote subnet behind the peer VPN endpoint. `--psk` is the pre-shared key (IKEv2 also supports certificate-based auth).

### Check Connection Status

```bash
openstack vpn ipsec site connection list
openstack vpn ipsec site connection show site-to-hq
```

Connection states: `PENDING_CREATE`, `ACTIVE`, `DOWN`, `ERROR`.

## Port Forwarding

Port forwarding (DNAT) allows external traffic on a specific floating IP port to be forwarded to an internal port on a VM. Multiple VM services can share a single floating IP.

```bash
# Forward external TCP:8080 on floating IP 203.0.113.15 to VM port 80
openstack floating ip port forwarding create \
  --internal-ip-address 192.168.1.50 \
  --internal-port 80 \
  --external-port 8080 \
  --protocol tcp \
  203.0.113.15

# Forward SSH (TCP:2222 externally → VM TCP:22)
openstack floating ip port forwarding create \
  --internal-ip-address 192.168.1.50 \
  --internal-port 22 \
  --external-port 2222 \
  --protocol tcp \
  203.0.113.15
```

List and delete port forwarding rules:

```bash
openstack floating ip port forwarding list 203.0.113.15
openstack floating ip port forwarding delete 203.0.113.15 <forwarding-uuid>
```

Enable the service plugin in `neutron.conf`:

```ini
[DEFAULT]
service_plugins = router,qos,trunk,port_forwarding
```

## DNS Integration

### Internal DNS (neutron-only)

Neutron maintains internal DNS records for ports. Every port gets a DNS name based on the port name or instance name and the network's `dns_domain`.

Set a `dns_domain` on a network:

```bash
openstack network set --dns-domain openstack.internal. private-net
```

Set a DNS name on a port:

```bash
openstack port set --dns-name web01 my-port
```

Neutron updates dnsmasq's hosts file with `<IP> web01.openstack.internal.` entries. VMs on the same network can resolve `web01.openstack.internal.` via their DHCP-assigned DNS server (the dnsmasq process).

### Designate Integration (External DNS)

When `dns_domain` is configured and Designate is available, Neutron can automatically publish DNS records to Designate for floating IPs and ports.

In `neutron.conf`:

```ini
[DEFAULT]
external_dns_driver = designate
dns_domain = cloud.example.com.

[designate]
url = http://designate01:9001/v2
auth_url = http://controller01:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = neutronservicepass
allow_reverse_dns_lookup = True
ipv4_ptr_zone_prefix_size = 24
ipv6_ptr_zone_prefix_size = 116
```

When a floating IP is associated with a DNS-named port, Neutron automatically creates an A record in Designate pointing `<dns-name>.<dns-domain>` to the floating IP address.

```bash
# Create a floating IP with a DNS name
openstack floating ip create \
  --dns-domain cloud.example.com. \
  --dns-name web01 \
  provider-net
# Creates: web01.cloud.example.com. A 203.0.113.15
```

### Enable DNS Extension

In `ml2_conf.ini`:

```ini
[ml2]
extension_drivers = port_security,qos,dns
```
