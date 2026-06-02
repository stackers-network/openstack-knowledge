# Ironic Operations

## Prerequisites

Confirm the Ironic API endpoint is reachable and hardware types are registered:

```bash
openstack baremetal driver list
openstack catalog show baremetal
```

All `openstack baremetal` commands require `python-openstackclient` with `python-ironicclient`. State-changing operations (manage, provide, deploy, clean, inspect) transition the node through the provisioning state machine asynchronously.

## Node Management

### Create a Node

```bash
# IPMI node (most common for generic hardware)
openstack baremetal node create \
  --driver ipmi \
  --driver-info ipmi_address=10.0.0.50 \
  --driver-info ipmi_username=admin \
  --driver-info ipmi_password=password \
  --name compute-01

# Redfish node (modern BMCs, Dell/HPE/Lenovo)
openstack baremetal node create \
  --driver redfish \
  --driver-info redfish_address=https://bmc.compute-02.example.com \
  --driver-info redfish_username=admin \
  --driver-info redfish_password=password \
  --driver-info redfish_system_id=/redfish/v1/Systems/1 \
  --name compute-02

# iDRAC node (Dell PowerEdge)
openstack baremetal node create \
  --driver idrac \
  --driver-info ipmi_address=10.0.0.52 \
  --driver-info ipmi_username=root \
  --driver-info ipmi_password=calvin \
  --driver-info redfish_address=https://10.0.0.52 \
  --driver-info redfish_username=root \
  --driver-info redfish_password=calvin \
  --name dell-01

# HPE iLO node
openstack baremetal node create \
  --driver ilo \
  --driver-info ilo_address=10.0.0.53 \
  --driver-info ilo_username=Administrator \
  --driver-info ilo_password=password \
  --name hpe-01

# Set hardware properties at creation time
openstack baremetal node create \
  --driver ipmi \
  --driver-info ipmi_address=10.0.0.54 \
  --driver-info ipmi_username=admin \
  --driver-info ipmi_password=password \
  --property memory_mb=65536 \
  --property cpus=32 \
  --property cpu_arch=x86_64 \
  --property local_gb=400 \
  --name compute-03
```

A newly created node is in `enroll` state with `maintenance=False`.

### List and Show Nodes

```bash
# List all nodes
openstack baremetal node list

# List by provision state
openstack baremetal node list --provision-state available
openstack baremetal node list --provision-state active
openstack baremetal node list --provision-state error

# List with extra fields
openstack baremetal node list --fields uuid,name,provision_state,power_state,maintenance

# Show full node details
openstack baremetal node show compute-01

# Show specific fields
openstack baremetal node show compute-01 --fields provision_state,power_state,last_error

# Show node by UUID
openstack baremetal node show <uuid>
```

### Set Node Properties and Capabilities

```bash
# Set hardware properties (required if not using inspection)
openstack baremetal node set compute-01 \
  --property memory_mb=65536 \
  --property cpus=32 \
  --property cpu_arch=x86_64 \
  --property local_gb=400

# Set capabilities (used for flavor/trait matching)
openstack baremetal node set compute-01 \
  --property capabilities='boot_mode:uefi,boot_option:local'

# Set resource class (maps to Nova flavor)
openstack baremetal node set compute-01 \
  --resource-class baremetal.large

# Set node name
openstack baremetal node set compute-01 \
  --name blade-rack2-slot4

# Set extra metadata
openstack baremetal node set compute-01 \
  --extra rack=rack-02 \
  --extra row=B

# Set deploy image (for standalone deploy without Nova)
openstack baremetal node set compute-01 \
  --instance-info image_source=<glance-image-uuid> \
  --instance-info image_checksum=<sha512> \
  --instance-info root_gb=50 \
  --instance-info capabilities='{"boot_mode": "uefi"}'

# Unset a property
openstack baremetal node unset compute-01 --property memory_mb
```

### Update Driver Info

```bash
# Change IPMI address
openstack baremetal node set compute-01 \
  --driver-info ipmi_address=10.0.0.55

# Change credentials
openstack baremetal node set compute-01 \
  --driver-info ipmi_username=newadmin \
  --driver-info ipmi_password=newpassword

# Remove a driver info key
openstack baremetal node unset compute-01 --driver-info ipmi_port
```

### Delete a Node

The node must be in `manageable` or `available` state (not `active`):

```bash
openstack baremetal node delete compute-01

# Force delete (admin, for stuck nodes)
openstack baremetal node delete --driver-info any compute-01
```

## Provisioning State Transitions

The provisioning state machine controls what operations are valid at each stage. Use these commands to move nodes through the machine:

### Enroll → Manageable (verify credentials)

```bash
openstack baremetal node manage compute-01
```

This triggers `verifying` state: the conductor contacts the BMC to verify credentials. On success, the node moves to `manageable`.

### Manageable → Available (clean and make available for deployment)

```bash
openstack baremetal node provide compute-01
```

If automated cleaning is enabled, this runs the clean workflow before moving to `available`. The node will appear in Nova's resource pool once `available`.

### Available → Deploying (manual/standalone deploy)

```bash
# Set instance info first (if not using Nova)
openstack baremetal node set compute-01 \
  --instance-info image_source=<glance-uuid> \
  --instance-info image_checksum=<sha512> \
  --instance-info root_gb=50

# Trigger deploy
openstack baremetal node deploy compute-01
```

Nova handles this automatically when `openstack server create` targets a bare metal flavor.

### Active → Available (undeploy/release)

```bash
openstack baremetal node undeploy compute-01
```

This triggers the tear-down workflow: Ironic unprovisions the node, runs automated cleaning if enabled, and returns it to `available`.

### Any State → Manageable (abort or recover)

```bash
# Return to manageable for maintenance
openstack baremetal node manage compute-01

# From deploy failed or error states
openstack baremetal node abort compute-01   # abort current operation
openstack baremetal node manage compute-01  # then return to manageable
```

### Maintenance Mode

Put a node in maintenance mode to prevent the scheduler from using it and suppress automated recovery attempts:

```bash
# Set maintenance with a reason
openstack baremetal node maintenance set compute-01 \
  --reason "DIMM replacement scheduled"

# Clear maintenance
openstack baremetal node maintenance unset compute-01

# List nodes in maintenance
openstack baremetal node list --maintenance
```

## Inspection

Hardware inspection discovers and sets node properties automatically.

```bash
# Trigger inspection (node must be in manageable state)
openstack baremetal node manage compute-01   # if in enroll
openstack baremetal node inspect compute-01

# Watch inspection progress
watch openstack baremetal node show compute-01 --fields provision_state,last_error

# After inspection, review what was discovered
openstack baremetal node show compute-01 --fields properties,extra
openstack baremetal port list --node compute-01
```

Inspection populates `node.properties`:
- `memory_mb`, `cpus`, `cpu_arch`, `local_gb`
- Creates or updates port records from discovered MAC addresses

After successful inspection, the node returns to `manageable` state.

## Port Management

Ports represent the physical NICs on a bare metal node. Ironic uses port MAC addresses for PXE boot and Neutron integration.

### Create Ports

```bash
# Add the primary NIC port (used for PXE boot)
openstack baremetal port create \
  --node compute-01 \
  aa:bb:cc:dd:ee:ff

# Add with a Neutron local link connection (for switch-level VLAN provisioning)
openstack baremetal port create \
  --node compute-01 \
  --local-link-connection switch_id=0a:1b:2c:3d:4e:5f \
  --local-link-connection port_id=Ethernet1/12 \
  --local-link-connection switch_info=switch-rack2 \
  aa:bb:cc:dd:ee:ff

# Add a port with PXE enabled explicitly
openstack baremetal port create \
  --node compute-01 \
  --pxe-enabled True \
  aa:bb:cc:dd:ee:ff

# Add a bond member port (part of a port group)
openstack baremetal port create \
  --node compute-01 \
  --portgroup <portgroup-uuid> \
  aa:bb:cc:dd:ee:01
```

### List and Show Ports

```bash
openstack baremetal port list
openstack baremetal port list --node compute-01
openstack baremetal port show aa:bb:cc:dd:ee:ff
openstack baremetal port show <port-uuid>
```

### Update and Delete Ports

```bash
openstack baremetal port set <port-uuid> --pxe-enabled False
openstack baremetal port unset <port-uuid> --extra owner
openstack baremetal port delete <port-uuid>
```

### Port Groups (Bonding)

```bash
# Create a port group for NIC bonding
openstack baremetal portgroup create \
  --node compute-01 \
  --name bond0 \
  --mode active-backup \
  --address aa:bb:cc:dd:ee:00

# Assign ports to the group
openstack baremetal port set <port1-uuid> --portgroup bond0
openstack baremetal port set <port2-uuid> --portgroup bond0
```

## Power Operations

```bash
# Power on
openstack baremetal node power on compute-01

# Power off (graceful or hard depending on driver)
openstack baremetal node power off compute-01

# Hard power off (force, no ACPI)
openstack baremetal node power off compute-01 --power-timeout 30

# Reboot
openstack baremetal node reboot compute-01

# Check power state
openstack baremetal node show compute-01 --fields power_state
```

## Driver and Interface Management

```bash
# List all available hardware types and their interfaces
openstack baremetal driver list

# Show detailed interface support for a hardware type
openstack baremetal driver show ipmi
openstack baremetal driver show redfish

# List registered conductors
openstack baremetal conductor list

# Show conductor details (which hardware types it handles)
openstack baremetal conductor show <hostname>
```

## BIOS Configuration

BIOS settings require a driver with a BIOS interface (e.g., `redfish`, `ilo`, `idrac-redfish`).

```bash
# Get all BIOS settings (node must be manageable)
openstack baremetal node bios setting list compute-01

# Show a specific setting
openstack baremetal node bios setting show compute-01 NumaGroupSizeOpt

# Set BIOS attributes
openstack baremetal node bios setting set compute-01 \
  NumaGroupSizeOpt=Clustered \
  PowerRegulator=StaticHighPerf \
  BootMode=Uefi

# Reset all BIOS settings to factory defaults
openstack baremetal node bios setting purge compute-01
```

BIOS changes take effect after the next reboot. The node must run through a clean step or a deploy cycle to apply them.

## RAID Configuration

RAID configuration is applied as a clean step. Define the target RAID configuration on the node, then run cleaning.

```bash
# Set target RAID configuration (RAID 1 with two disks)
openstack baremetal node set compute-01 \
  --target-raid-config '{
    "logical_disks": [
      {
        "size_gb": 100,
        "raid_level": "1",
        "is_root_volume": true
      }
    ]
  }'

# Set target RAID config with explicit physical disk selection
openstack baremetal node set compute-01 \
  --target-raid-config '{
    "logical_disks": [
      {
        "size_gb": 400,
        "raid_level": "5",
        "disk_hints": {
          "rotational": true,
          "minimum_size_gb": 300
        }
      }
    ]
  }'

# Apply the RAID config via manual cleaning
openstack baremetal node clean compute-01 \
  --clean-steps '[
    {"interface": "raid", "step": "delete_configuration"},
    {"interface": "raid", "step": "create_configuration"}
  ]'

# After cleaning, show the current RAID config
openstack baremetal node show compute-01 --fields raid_config
```

## Cleaning

### Manual Cleaning

```bash
# Erase all disks (secure erase or shred)
openstack baremetal node clean compute-01 \
  --clean-steps '[{"interface": "deploy", "step": "erase_devices"}]'

# Erase disks with metadata only (faster; wipes partition table, not full disk)
openstack baremetal node clean compute-01 \
  --clean-steps '[{"interface": "deploy", "step": "erase_devices_metadata"}]'

# Full cleaning sequence: RAID delete, disk erase, RAID recreate
openstack baremetal node clean compute-01 \
  --clean-steps '[
    {"interface": "raid", "step": "delete_configuration"},
    {"interface": "deploy", "step": "erase_devices"},
    {"interface": "raid", "step": "create_configuration"}
  ]'
```

### Monitor Cleaning Progress

```bash
# Watch clean step progress
watch openstack baremetal node show compute-01 --fields provision_state,clean_step

# Check last error if cleaning failed
openstack baremetal node show compute-01 --fields last_error
```

After successful manual cleaning, the node returns to `manageable` state (not `available`). Run `openstack baremetal node provide compute-01` to make it available.

## Allocations (Node Reservation API)

The allocation API lets you request a node by resource class and traits without specifying a node UUID:

```bash
# Request any available baremetal.large node
openstack baremetal allocation create \
  --resource-class baremetal.large \
  --name my-server-1

# Request a node with specific traits
openstack baremetal allocation create \
  --resource-class baremetal.gpu \
  --trait CUSTOM_NVIDIA_A100 \
  --name gpu-server-1

# Wait for allocation to complete
openstack baremetal allocation show my-server-1

# List allocations
openstack baremetal allocation list

# Release an allocation
openstack baremetal allocation delete my-server-1
```

## Node Traits

Traits allow fine-grained scheduling by attaching capability strings to nodes:

```bash
# Add a trait
openstack baremetal node add trait compute-01 CUSTOM_NVME_STORAGE

# Add multiple traits
openstack baremetal node add trait compute-01 \
  CUSTOM_NVME_STORAGE \
  CUSTOM_25GBE_NIC

# List traits on a node
openstack baremetal node show compute-01 --fields traits

# Remove a trait
openstack baremetal node remove trait compute-01 CUSTOM_NVME_STORAGE
```

Standard traits come from the os-traits library (e.g., `HW_CPU_X86_VMX`, `STORAGE_DISK_SSD`). Custom traits must start with `CUSTOM_`.

## Troubleshooting

```bash
# Show the last error for a failed node
openstack baremetal node show compute-01 --fields last_error,provision_state

# Check conductor logs (on the conductor host)
journalctl -u ironic-conductor --since "15 minutes ago"

# Check IPA logs during deploy (IPA logs are forwarded to the conductor)
# or SSH to the node during deploy (if console/SSH is configured)
openstack baremetal node console enable compute-01
openstack baremetal node console show compute-01

# Validate node driver configuration (checks credentials, boot config, etc.)
openstack baremetal node validate compute-01

# Force a power state sync
openstack baremetal node show compute-01 --fields power_state
# If power state is incorrect, the conductor periodic task will sync it

# Abort a stuck deployment
openstack baremetal node abort compute-01

# Reset a node stuck in error state
openstack baremetal node manage compute-01   # back to manageable
openstack baremetal node provide compute-01  # then to available (runs clean)
```
