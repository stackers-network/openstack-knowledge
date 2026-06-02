# Bare Metal — Ironic

Ironic is the OpenStack Bare Metal provisioning service. It manages the lifecycle of physical servers — enrolling them, inspecting their hardware, deploying operating system images to them, and returning them to an available pool when tenants are done. Ironic integrates with Nova (so `openstack server create` can target a bare metal node), Neutron (for network provisioning), and Glance (for OS images). Hardware is managed through a pluggable driver model supporting IPMI, Redfish, iDRAC, iLO, and others.

## When to Read This Skill

- Enrolling physical servers (nodes) into Ironic
- Configuring IPMI or Redfish credentials for power and management control
- Understanding and navigating the provisioning state machine
- Deploying OS images to bare metal using PXE/iPXE or Redfish virtual media
- Running hardware inspection on enrolled nodes
- Configuring automated or manual cleaning between deployments
- Setting up BIOS and RAID configurations via Ironic
- Integrating Ironic with Nova for tenant-facing bare metal instances
- Troubleshooting failed deployments and understanding IPA (Ironic Python Agent)
- Managing ports, port groups, and Neutron network attachment for bare metal

## Sub-Files

| File | What It Covers |
|---|---|
| [architecture.md](architecture.md) | ironic-api, ironic-conductor; hardware types and driver interfaces; provisioning state machine; cleaning and inspection; Nova/Neutron/Glance integration |
| [operations.md](operations.md) | CLI: node enrollment, port management, power control, provisioning (manage/provide/deploy/undeploy), inspection, BIOS config, RAID config, cleaning |
| [internals.md](internals.md) | Driver composition model; conductor hash ring; deploy steps and clean steps; IPA (Ironic Python Agent) lifecycle; full provisioning state machine with all transitions |

## Quick Reference

```bash
# Enroll a node with IPMI
openstack baremetal node create \
  --driver ipmi \
  --driver-info ipmi_address=10.0.0.50 \
  --driver-info ipmi_username=admin \
  --driver-info ipmi_password=password \
  --name compute-01

# Add a port (MAC address for PXE)
openstack baremetal port create \
  --node compute-01 \
  aa:bb:cc:dd:ee:ff

# Move through the enrollment state machine
openstack baremetal node manage compute-01
openstack baremetal node provide compute-01

# Trigger hardware inspection
openstack baremetal node inspect compute-01

# Deploy an OS image
openstack baremetal node deploy compute-01

# Power operations
openstack baremetal node power on compute-01
openstack baremetal node power off compute-01

# Check node status
openstack baremetal node show compute-01
```

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| Keystone | Authentication and service catalog | All Ironic API calls use Keystone tokens |
| Nova | Manages bare metal instances as Nova servers; schedules via resource providers | `nova-compute` with `ironic` virt driver on conductor hosts |
| Neutron | Provisions network ports for bare metal nodes during deploy/undeploy | Required for tenant VLAN isolation; `baremetal` ML2 mechanism driver |
| Glance | Stores OS images for deployment | Supports raw, qcow2, and whole-disk images |
| Swift / RadosGW | Temporary image storage during direct deploy | Required when agents download from object store |
| DHCP / TFTP / HTTP | PXE/iPXE boot infrastructure | Provided by ironic-conductor or an external service |
| `python-openstackclient` | Unified CLI (`openstack baremetal …`) | Wraps the Ironic v1 REST API |
| `python-ironicclient` | Python bindings | Used by Nova's Ironic virt driver and automation |

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

Ironic release notes: https://docs.openstack.org/releasenotes/ironic/
