# Nova Operations

## Server Lifecycle

### Create a Server

```bash
# Minimal: flavor + image + network
openstack server create \
  --flavor m1.small \
  --image ubuntu-24.04 \
  --network private-net \
  my-instance

# Full: key pair, security group, user-data, availability zone, boot volume
openstack server create \
  --flavor m1.medium \
  --image ubuntu-24.04 \
  --network private-net \
  --key-name mykey \
  --security-group web-sg \
  --user-data /path/to/cloud-init.yaml \
  --availability-zone nova:compute01.example.com \
  --description "Web server" \
  my-instance

# Boot from an existing Cinder volume
openstack server create \
  --flavor m1.medium \
  --volume my-boot-volume \
  --network private-net \
  --key-name mykey \
  my-volume-instance

# Boot from image and create a new volume (volume-backed)
openstack server create \
  --flavor m1.medium \
  --image ubuntu-24.04 \
  --boot-from-volume 50 \
  --network private-net \
  --key-name mykey \
  my-vb-instance

# Schedule on a specific host (admin)
openstack server create \
  --flavor m1.small \
  --image ubuntu-24.04 \
  --network private-net \
  --availability-zone nova:compute03.example.com \
  my-pinned-instance
```

### List and Inspect Servers

```bash
# List servers in current project
openstack server list

# List all servers across all projects (admin)
openstack server list --all-projects

# Filter by status
openstack server list --status ERROR
openstack server list --status ACTIVE

# Filter by host (admin)
openstack server list --all-projects --host compute01.example.com

# Show full details of a server
openstack server show my-instance

# Show server events/actions (useful for troubleshooting)
openstack server event list my-instance
openstack server event show my-instance <request-id>

# Show diagnostics (hypervisor metrics)
openstack server show --diagnostics my-instance
```

### Power State Operations

```bash
# Soft reboot (ACPI shutdown signal then power on)
openstack server reboot --soft my-instance

# Hard reboot (power cycle, like pressing the reset button)
openstack server reboot --hard my-instance

# Stop (power off; instance remains on host, resources held)
openstack server stop my-instance

# Start a stopped server
openstack server start my-instance

# Pause (freeze CPU/memory in place; fast; requires hypervisor support)
openstack server pause my-instance
openstack server unpause my-instance

# Suspend (save state to disk; slower; uses no CPU)
openstack server suspend my-instance
openstack server resume my-instance
```

### Shelve and Unshelve

Shelving offloads the instance from the compute host (frees up RAM and CPU) while preserving the instance record and its associated resources (volumes, floating IPs).

```bash
# Shelve: snapshot the instance, then power off and offload from host
openstack server shelve my-instance

# Unshelve: restore from snapshot back onto a host
openstack server unshelve my-instance

# Unshelve to a specific availability zone (2022.1+)
openstack server unshelve --availability-zone nova my-instance
```

### Delete a Server

```bash
# Normal delete (waits for graceful shutdown)
openstack server delete my-instance

# Force delete (skips graceful shutdown; use with caution)
openstack server delete --force my-instance

# Delete multiple servers
openstack server delete server1 server2 server3
```

### Rebuild a Server

Rebuild re-images the instance in place: the root disk is replaced with a new image, but the instance stays on the same host with the same UUID, flavor, and network ports.

```bash
# Rebuild with the same image
openstack server rebuild my-instance --image ubuntu-24.04

# Rebuild with a different image
openstack server rebuild my-instance --image ubuntu-22.04

# Rebuild with new admin password and user-data
openstack server rebuild my-instance \
  --image ubuntu-24.04 \
  --admin-password NewP@ss123 \
  --user-data /path/to/new-user-data.yaml
```

### Resize a Server

```bash
# Resize to a larger flavor (moves to new host if required)
openstack server resize --flavor m1.large my-instance

# Poll until status is VERIFY_RESIZE
openstack server show my-instance -c status

# Confirm the resize (destroys old disk, keeps new)
openstack server resize confirm my-instance

# Revert the resize (back to original flavor and host)
openstack server resize revert my-instance
```

### Migrate a Server

```bash
# Cold migration: stop, copy disk, start on new host (admin)
openstack server migrate my-instance

# Confirm cold migration (same as resize confirm)
openstack server resize confirm my-instance

# Live migration: keep running, move to any available host
openstack server live-migrate my-instance

# Live migration to a specific host
openstack server live-migrate --host compute04.example.com my-instance

# Block migration (copies local disk; use when no shared storage)
openstack server live-migrate --block-migrate my-instance
```

### Evacuate a Server

Evacuation moves an instance off a dead compute host to a new one. Requires the source host to be down (fenced).

```bash
# Evacuate to any available host
openstack server evacuate my-instance

# Evacuate to a specific host
openstack server evacuate --host compute05.example.com my-instance

# Evacuate all instances from a host (admin bulk operation)
openstack server list --all-projects --host compute02.example.com -f value -c ID | \
  xargs -I {} openstack server evacuate {}
```

### Add and Manage Attachments

```bash
# Attach a Cinder volume
openstack server add volume my-instance my-volume

# Attach at a specific device path
openstack server add volume --device /dev/vdb my-instance my-volume

# Detach a volume
openstack server remove volume my-instance my-volume

# Add a floating IP
openstack server add floating ip my-instance 203.0.113.42

# Remove a floating IP
openstack server remove floating ip my-instance 203.0.113.42

# Add a security group
openstack server add security group my-instance database-sg

# Remove a security group
openstack server remove security group my-instance database-sg
```

### Console Access

```bash
# Get VNC console URL (novnc)
openstack console url show --novnc my-instance

# Get SPICE console URL
openstack console url show --spice my-instance

# Get serial console URL
openstack console url show --serial my-instance

# Read the console log (last 50 lines)
openstack console log show --lines 50 my-instance
```

### Server Backup

```bash
# Create a snapshot image of the instance
openstack server image create --name my-instance-snap-2025-04-01 my-instance

# Create a scheduled backup (daily, keep 7 copies)
openstack server backup create \
  --name my-instance-backup \
  --type daily \
  --rotate 7 \
  my-instance
```

---

## Flavors

Flavors define the virtual hardware profile (vCPU count, RAM, root disk, ephemeral disk, swap).

```bash
# List all flavors
openstack flavor list

# List including private flavors (admin)
openstack flavor list --all

# Show flavor details
openstack flavor show m1.small

# Create a standard flavor
openstack flavor create \
  --vcpus 2 \
  --ram 4096 \
  --disk 40 \
  --ephemeral 0 \
  --swap 0 \
  --public \
  m1.medium

# Create a private flavor for a specific project
openstack flavor create \
  --vcpus 16 \
  --ram 65536 \
  --disk 200 \
  --private \
  hpc.xlarge

# Grant access to a private flavor
openstack flavor set --project engineering hpc.xlarge

# Set extra specs (used by scheduler filters and virt driver)
openstack flavor set m1.medium \
  --property hw:cpu_policy=dedicated \
  --property hw:mem_page_size=large \
  --property aggregate_instance_extra_specs:ssd=true

# Remove an extra spec
openstack flavor unset --property hw:cpu_policy m1.medium

# Delete a flavor
openstack flavor delete m1.medium
```

### Common Flavor Extra Specs

| Extra Spec | Values | Effect |
|---|---|---|
| `hw:cpu_policy` | `dedicated`, `shared` | CPU pinning |
| `hw:mem_page_size` | `large`, `small`, `any` | Huge page allocation |
| `hw:numa_nodes` | integer | NUMA topology |
| `hw:cpu_sockets` | integer | vCPU socket topology |
| `hw:cpu_cores` | integer | vCPU core topology |
| `hw:cpu_threads` | integer | vCPU thread topology |
| `hw:vif_multiqueue_enabled` | `true` | Enable virtio-net multiqueue |
| `hw:boot_menu` | `true` | Show boot menu in console |
| `aggregate_instance_extra_specs:<key>` | any | Match host aggregate metadata |
| `trait:<trait-name>` | `required` | Require Placement trait on host |

---

## Key Pairs

```bash
# Create a new key pair (generates and downloads private key)
openstack keypair create mykey > ~/.ssh/mykey.pem
chmod 600 ~/.ssh/mykey.pem

# Import an existing public key
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey

# List key pairs
openstack keypair list

# Show a key pair (public key + fingerprint)
openstack keypair show mykey

# Delete a key pair
openstack keypair delete mykey
```

---

## Quotas

```bash
# Show current project quota and usage
openstack quota show

# Show quota for a specific project (admin)
openstack quota show --project engineering

# Show compute quota details only
openstack quota show --compute

# Set project quotas (admin)
openstack quota set \
  --instances 50 \
  --cores 200 \
  --ram 512000 \
  --key-pairs 20 \
  --server-groups 10 \
  --server-group-members 10 \
  engineering

# Reset a quota to the default
openstack quota delete --compute engineering

# Show default quotas (the baseline for all projects)
openstack quota show --default
```

### Common Quota Fields

| Field | Default | Description |
|---|---|---|
| `instances` | 10 | Maximum running instances |
| `cores` | 20 | Maximum total vCPUs |
| `ram` | 51200 | Maximum total RAM (MB) |
| `key_pairs` | 100 | Maximum key pairs per user |
| `server_groups` | 10 | Maximum server groups |
| `server_group_members` | 10 | Maximum instances per server group |

---

## Server Groups

Server groups apply anti-affinity or affinity scheduling policies to a set of instances.

```bash
# Create an anti-affinity group (instances spread across hosts)
openstack server group create --policy anti-affinity web-tier

# Create an affinity group (instances land on the same host)
openstack server group create --policy affinity batch-workers

# Soft anti-affinity (best effort, not hard requirement)
openstack server group create --policy soft-anti-affinity api-nodes

# Launch an instance into a server group
openstack server create \
  --flavor m1.small \
  --image ubuntu-24.04 \
  --network private-net \
  --hint group=<server-group-uuid> \
  web-01

# List server groups
openstack server group list

# Show members of a server group
openstack server group show web-tier
```

---

## Hypervisor Management (Admin)

```bash
# List hypervisors with resource summary
openstack hypervisor list --long

# Show details of a specific hypervisor
openstack hypervisor show compute01.example.com

# Show hypervisor statistics across all hosts
openstack hypervisor stats show

# List compute services and their state
openstack compute service list

# Disable a compute service (drain host before maintenance)
openstack compute service set --disable \
  --disable-reason "Scheduled maintenance 2025-04-05" \
  compute02.example.com nova-compute

# Re-enable a compute service
openstack compute service set --enable compute02.example.com nova-compute

# Force-delete a service record (after host permanently removed)
openstack compute service delete <service-id>
```

---

## Host Aggregates and Availability Zones

```bash
# Create an aggregate
openstack aggregate create ssd-hosts

# Add a host to an aggregate
openstack aggregate add host ssd-hosts compute03.example.com

# Set aggregate metadata (matched by scheduler filters)
openstack aggregate set --property ssd=true ssd-hosts

# Create an availability zone via an aggregate
openstack aggregate create --zone az-gpu gpu-zone

# Add a host to the AZ aggregate
openstack aggregate add host gpu-zone compute05.example.com

# List aggregates
openstack aggregate list

# Show aggregate details
openstack aggregate show ssd-hosts
```

---

## Configuration Reference

Nova reads `/etc/nova/nova.conf`. The most important sections follow.

### [DEFAULT]

```ini
[DEFAULT]
# Transport URL for RabbitMQ
transport_url = rabbit://nova:nova-pass@controller:5672/nova

# Block of IPs to use for instance metadata
my_ip = 10.0.0.11

# Enable compute (set on compute nodes only)
enabled_apis = osapi_compute,metadata

# Log directory
log_dir = /var/log/nova
```

### [api]

```ini
[api]
auth_strategy = keystone
```

### [keystone_authtoken]

```ini
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova-service-pass
```

### [scheduler]

```ini
[scheduler]
# How many hosts to pass to the scheduler per instance
max_attempts = 3

# Period between scheduler periodic tasks (seconds)
periodic_task_interval = 60

# Enable periodic discovery of new compute hosts
discover_hosts_in_cells_interval = 300
```

### [filter_scheduler]

```ini
[filter_scheduler]
# Ordered list of enabled filters
enabled_filters = ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter,AggregateInstanceExtraSpecsFilter,AvailabilityZoneFilter,PciPassthroughFilter

# Number of candidate hosts to consider before weighing
host_subset_size = 1

# Track instance counts in DB for NumInstancesFilter
track_instance_changes = True
```

### [compute]

```ini
[compute]
# CPU allocation ratio: overcommit factor
cpu_allocation_ratio = 4.0

# RAM allocation ratio
ram_allocation_ratio = 1.5

# Disk allocation ratio
disk_allocation_ratio = 1.0

# Interval between resource tracker updates (seconds)
resource_tracker_poll_interval = 60

# Reserved host RAM (MB) — excluded from Placement inventory
reserved_host_memory_mb = 512

# Reserved host disk (MB)
reserved_host_disk_mb = 0
```

### [libvirt]

```ini
[libvirt]
# Virtualisation type: kvm, qemu, xen, lxc
virt_type = kvm

# Connection URI to libvirt daemon
connection_uri = qemu:///system

# Disk image format: qcow2, raw
images_type = qcow2

# Live migration URI (tcp or tls)
live_migration_uri = qemu+ssh://nova@%s/system

# Use SSH tunnelling for live migration
live_migration_tunnelled = False

# Seconds before a live migration is aborted
live_migration_completion_timeout = 800

# Timeout for post-copy migration phase (seconds)
live_migration_timeout_action = abort

# Snapshot format
snapshot_image_format = qcow2

# CPU mode: host-passthrough, host-model, custom, none
cpu_mode = host-model
```

### [placement]

```ini
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement-service-pass
```

### [vnc]

```ini
[vnc]
# Enable VNC for this compute node
enabled = True

# novnc proxy address (controller node)
novncproxy_base_url = http://controller:6080/vnc_auto.html

# Address nova-compute binds VNC to (this host)
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
```

### [notifications]

```ini
[notifications]
# Notification driver: noop, messagingv2
driver = messagingv2

# Transport URL for notifications (can differ from main bus)
transport_url = rabbit://nova:nova-pass@controller:5672/nova_notifications

# Notification format: unversioned, versioned, both
notification_format = versioned
```

### [oslo_concurrency]

```ini
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

### [glance]

```ini
[glance]
api_servers = http://controller:9292
```

### [neutron]

```ini
[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = neutron-service-pass
service_metadata_proxy = True
metadata_proxy_shared_secret = metadata-proxy-secret
```

### [cinder]

```ini
[cinder]
os_region_name = RegionOne
```

---

## Database Management

```bash
# Sync the API database schema
nova-manage api_db sync

# Sync the cell0 database schema
nova-manage db sync --database nova_cell0

# Sync the cell1 (default cell) database schema
nova-manage db sync

# Create cell0 (run once on initial deploy)
nova-manage cell_v2 map_cell0 --database-connection \
  mysql+pymysql://nova:nova-pass@controller/nova_cell0

# Create cell1 (run once on initial deploy)
nova-manage cell_v2 create_cell \
  --name cell1 \
  --transport-url "rabbit://nova:nova-pass@controller:5672/nova_cell1" \
  --database-connection "mysql+pymysql://nova:nova-pass@controller/nova_cell1"

# Discover new compute hosts added to a cell
nova-manage cell_v2 discover_hosts --cell-uuid <cell-uuid> --verbose

# List all cells
nova-manage cell_v2 list_cells

# Check database migrations status
nova-manage db version
nova-manage api_db version
```
