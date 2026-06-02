# Ironic Architecture

## Overview

Ironic is a two-process service: `ironic-api` handles REST requests and `ironic-conductor` executes all hardware operations. Hardware is abstracted through a layered driver model: each node is assigned a *hardware type* (the overall hardware platform) and one or more *interfaces* that implement specific capabilities (power control, deployment, boot, etc.).

```
                    ┌────────────────────────────────────────────┐
                    │              Ironic Services                │
                    │                                            │
  REST API ────────►│  ironic-api   (WSGI, no hardware access)  │
                    │       │ RPC (oslo.messaging)              │
                    │       ▼                                    │
                    │  ironic-conductor                         │
                    │   ├── hash ring (node distribution)       │
                    │   ├── deploy / clean / inspect workflows  │
                    │   ├── periodic tasks (sync, heartbeat)    │
                    │   └── driver interfaces                   │
                    └────────────────────────────────────────────┘
                               │ IPMI / Redfish / iDRAC / iLO
                               │ PXE / TFTP / HTTP
                               │
                    ┌──────────┴──────────┐
                    │   Physical Server   │
                    │  (bare metal node)  │
                    │  ┌───────────────┐  │
                    │  │  IPA ramdisk  │  │  (during deploy/clean)
                    │  └───────────────┘  │
                    └─────────────────────┘
```

## Service Processes

### ironic-api

The WSGI front-end for the Ironic v1 REST API. Responsibilities:

- Authenticates requests (Keystone tokens) and enforces policy (oslo.policy)
- Validates node, port, and chassis resource creation and update requests
- Reads node state directly from the database (no hardware contact for GETs)
- Forwards state-changing requests (provision, power, clean, inspect) to ironic-conductor via oslo.messaging RPC
- Handles API microversioning (every feature addition is gated behind a microversion number)
- Exposes the `allocation` API for requesting nodes by traits/resource class

Multiple ironic-api workers can run behind a load balancer. All are stateless.

### ironic-conductor

Executes all operations that touch hardware or manage complex state. Responsibilities:

- Subscribes to RPC calls from ironic-api
- Manages the provisioning state machine for each node (transitions, lock management)
- Implements deploy, undeploy, clean, inspect, rescue, and BIOS/RAID operations
- Runs periodic tasks: power state sync, heartbeat monitoring for deployed nodes, failed node rescue
- Manages PXE/DHCP configuration files for nodes during deploy
- Communicates with IPA (Ironic Python Agent) running on the node during deploy/clean via HTTP

Multiple ironic-conductor processes can run simultaneously. Node ownership is distributed using a **consistent hash ring** (see Internals) so that each conductor handles a subset of nodes. If a conductor fails, its nodes are automatically claimed by surviving conductors.

## Hardware Types

A hardware type is a Python class that declares which interface implementations are supported for a given hardware platform. Ironic ships several built-in hardware types:

| Hardware Type | Target Hardware | Power / Management |
|---|---|---|
| `ipmi` | Generic servers with IPMI (BMC) | `ipmitool` |
| `redfish` | Servers with Redfish BMC (DMTF standard) | `redfish` library |
| `idrac` | Dell PowerEdge with iDRAC | IPMI + iDRAC-specific extensions |
| `ilo` | HPE ProLiant with iLO | iLO REST API |
| `irmc` | Fujitsu PRIMERGY with iRMC | iRMC S4/S5 REST API |
| `xclarity` | Lenovo with XClarity Controller | IPMI |
| `staging-ovirt` | oVirt virtual bare metal (CI) | via oVirt API |
| `fake-hardware` | Testing/CI | Simulated; no real hardware |

Each hardware type specifies its supported interfaces. When a node is created, the conductor validates that the requested driver matches a registered hardware type and that all interface implementations are compatible with it.

## Driver Interfaces

Interfaces are the pluggable units of hardware capability. A node is configured with one implementation per interface type:

### Power Interface

Controls server power state (on/off/reboot).

| Implementation | Transport |
|---|---|
| `ipmitool` | IPMI over LAN (`ipmitool -I lanplus`) |
| `redfish` | Redfish `Systems/{id}/Actions/ComputerSystem.Reset` |
| `ilo` | HPE iLO REST API |
| `idrac-redfish` | iDRAC via Redfish |
| `fake` | Simulated (testing) |

### Management Interface

Controls boot device selection and firmware settings.

| Implementation | Capabilities |
|---|---|
| `ipmitool` | Boot device (PXE, disk, BIOS), sensor data |
| `redfish` | Boot device, BIOS attribute setting, firmware update |
| `ilo` | iLO-specific boot, firmware, hardware settings |
| `idrac-redfish` | Dell-specific BIOS and boot configuration |

### Deploy Interface

Orchestrates OS deployment once the node has booted the IPA ramdisk.

| Implementation | Mechanism |
|---|---|
| `direct` | IPA downloads image from HTTP/Swift URL, writes to disk, reboots (default) |
| `ramdisk` | Deploy completes inside the ramdisk without rebooting (for stateless ramdisk-only boots) |
| `anaconda` | Uses Anaconda installer with a kickstart file (RHEL/CentOS) |
| `custom-agent` | User-defined deploy steps only; no built-in image write |

### Boot Interface

Controls how the node boots (both during deploy and after deployment).

| Implementation | Mechanism |
|---|---|
| `pxe` | Traditional PXE; TFTP serves pxelinux/grub config |
| `ipxe` | iPXE; HTTP-based chainloading; faster image download |
| `redfish-virtual-media` | Mounts an ISO via Redfish VirtualMedia; no DHCP/TFTP required |
| `ilo-virtual-media` | HPE iLO ISO mount |
| `idrac-redfish-virtual-media` | Dell iDRAC ISO mount |
| `fake` | Simulated (testing) |

### Inspect Interface

Discovers hardware properties (CPU count, RAM, disk size, NICs, MAC addresses) automatically.

| Implementation | Mechanism |
|---|---|
| `inspector` | Boots the node into the `ironic-inspector` ramdisk; inspector runs hardware discovery |
| `redfish` | Queries Redfish `Systems/{id}` and `Memory`, `Processors`, `EthernetInterfaces` resources |
| `ilo` | HPE iLO REST API hardware inventory |
| `idrac-redfish` | Dell iDRAC hardware inventory via Redfish |
| `no-inspect` | Inspection skipped; properties set manually |

### BIOS Interface

Reads and writes firmware (BIOS/UEFI) settings.

| Implementation | Capabilities |
|---|---|
| `redfish` | Read/write Redfish BIOS attributes |
| `ilo` | HPE iLO BIOS settings |
| `idrac-redfish` | Dell iDRAC BIOS settings |
| `no-bios` | No BIOS management |

### RAID Interface

Configures hardware RAID before deployment.

| Implementation | Target |
|---|---|
| `agent` | Uses IPA RAID commands (software RAID or hardware RAID via vendor tools in ramdisk) |
| `idrac-redfish` | Dell iDRAC hardware RAID configuration |
| `ilo` | HPE Smart Array via iLO |
| `no-raid` | No RAID management |

### Console Interface

Provides serial console access to the node.

| Implementation | Mechanism |
|---|---|
| `ipmitool-socat` | IPMI Serial over LAN → socat TCP |
| `ilo` | iLO console |
| `no-console` | Console disabled |

### Vendor Interface

Hardware-vendor-specific extensions not covered by other interfaces.

## Provisioning State Machine

The provisioning state machine governs all state transitions for a bare metal node. See [internals.md](internals.md) for the full diagram and transition table.

Key states:

| State | Meaning |
|---|---|
| `enroll` | Node is registered; no hardware access attempted yet |
| `verifying` | Conductor is verifying hardware credentials |
| `manageable` | Credentials verified; available for inspection and maintenance |
| `available` | Node is clean and available for deployment (in the Nova resource pool) |
| `deploying` | Deployment workflow is running |
| `wait call-back` | Deployment is waiting for IPA to call back |
| `active` | Node is deployed and running a tenant workload |
| `cleaning` | Automated or manual cleaning is running |
| `inspect failed` | Inspection failed |
| `deploy failed` | Deployment failed |
| `error` | Node encountered an unrecoverable error |

## Cleaning

Cleaning erases or resets hardware between tenant deployments to prevent data leakage and return the node to a known state.

### Automated Cleaning

Runs automatically when a node moves from `active` (or `deploy failed`) back to `available`. Controlled by:

```ini
[conductor]
automated_clean = True
```

The conductor runs the `clean_steps` defined by the deploy, management, and RAID interfaces. For the `direct` deploy interface, the default clean step is:

- `deploy.erase_devices`: wipes all local disks using `shred`, `blkdiscard`, or `ata-secure-erase` (via IPA)

### Manual Cleaning

Triggered by an operator, allowing custom clean steps without returning the node to `available`:

```bash
openstack baremetal node clean compute-01 \
  --clean-steps '[{"interface": "deploy", "step": "erase_devices"}]'

# Multiple clean steps in order
openstack baremetal node clean compute-01 \
  --clean-steps '[
    {"interface": "raid", "step": "delete_configuration"},
    {"interface": "raid", "step": "create_configuration"},
    {"interface": "deploy", "step": "erase_devices"}
  ]'
```

## Inspection

Inspection populates node properties automatically by booting the node into a discovery ramdisk:

```
ironic-conductor
    │ power on + set boot device to PXE
    ▼
Node boots ironic-inspector ramdisk
    │ hardware discovery (CPU, RAM, disks, NICs, MAC addresses)
    ▼
ironic-inspector (or conductor for redfish in-band)
    │ POST /v1/continue with inventory
    ▼
ironic-conductor
    │ populate node properties and ports from inventory
    ▼
node.properties = {
    "memory_mb": 65536,
    "cpus": 32,
    "cpu_arch": "x86_64",
    "local_gb": 400
}
```

## Nova Integration

Nova manages bare metal nodes as a special virt driver (`ironic` virt driver on `nova-compute`). The integration flow:

1. ironic-conductor reports each `available` node as a Nova resource provider with a custom resource class (e.g., `CUSTOM_BAREMETAL_LARGE`)
2. Nova scheduler selects a bare metal resource provider matching the instance's flavor resource class
3. Nova calls `ironic-api` to reserve the node (transitions to `deploying`)
4. Nova passes the image UUID (Glance), network info (Neutron ports), and config drive content to Ironic
5. Ironic deploys the image to the node
6. On successful deployment, Nova marks the instance `ACTIVE`

Bare metal flavors in Nova must specify a resource class:

```bash
openstack flavor create \
  --ram 65536 \
  --disk 400 \
  --vcpus 32 \
  --property resources:CUSTOM_BAREMETAL_LARGE=1 \
  --property resources:VCPU=0 \
  --property resources:MEMORY_MB=0 \
  --property resources:DISK_GB=0 \
  baremetal.large
```

## Neutron Integration

When a node is deployed, Ironic configures Neutron to provision the correct VLAN on the node's switchport. The flow:

1. Ironic creates a Neutron port for each baremetal port with `binding:vnic_type=baremetal`
2. Neutron's `baremetal` ML2 mechanism driver receives the port binding request
3. The mechanism driver calls the network switch API (via the `networking-baremetal` or `networking-generic-switch` plugin) to configure the access VLAN on the physical switchport connected to the node's NIC
4. After deployment, the switchport is updated to the tenant VLAN
5. On undeploy, the switchport is reverted to the cleaning/provisioning network VLAN

## Glance Integration

Node deploy images are stored in Glance and referenced by UUID in the node's `instance_info`:

```bash
openstack baremetal node set compute-01 \
  --instance-info image_source=<glance-image-uuid> \
  --instance-info image_checksum=<md5-or-sha512> \
  --instance-info root_gb=50
```

Supported image formats:
- **raw**: copied directly to disk; fastest deployment
- **qcow2**: converted to raw by IPA on the node using `qemu-img`
- **whole-disk image**: no partitioning needed; written directly to the target disk
- **partition image**: Ironic creates a partition table and writes the image to the root partition

## Configuration Overview

Key `ironic.conf` settings:

```ini
[DEFAULT]
enabled_hardware_types = ipmi,redfish,idrac,ilo
enabled_power_interfaces = ipmitool,redfish,idrac-redfish,ilo
enabled_management_interfaces = ipmitool,redfish,idrac-redfish,ilo
enabled_deploy_interfaces = direct,ramdisk
enabled_boot_interfaces = pxe,ipxe,redfish-virtual-media
enabled_inspect_interfaces = inspector,redfish,no-inspect
enabled_bios_interfaces = redfish,ilo,no-bios
enabled_raid_interfaces = agent,no-raid
enabled_console_interfaces = ipmitool-socat,no-console
enabled_vendor_interfaces = ipmitool,no-vendor

default_deploy_interface = direct
default_boot_interface = ipxe
default_inspect_interface = inspector

[conductor]
automated_clean = True
clean_callback_timeout = 1800
deploy_callback_timeout = 1800
inspect_timeout = 1800

[deploy]
http_url = http://192.168.1.10:8080/   # HTTP server for image serving
http_root = /var/lib/ironic/httpboot/

[pxe]
tftp_server = 192.168.1.10
tftp_root = /var/lib/ironic/tftpboot/
pxe_append_params = nofb nomodeset vga=normal
```
