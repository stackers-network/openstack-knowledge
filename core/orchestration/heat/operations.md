# Heat Operations

## Stack Lifecycle

### Create a Stack

```bash
# Basic stack creation
openstack stack create -t web-app.yaml web-app-stack

# With environment file and parameter overrides
openstack stack create \
  -t web-app.yaml \
  -e prod-env.yaml \
  --parameter db_password=secret123 \
  web-app-stack

# Multiple environment files (applied in order; later overrides earlier)
openstack stack create \
  -t web-app.yaml \
  -e base-env.yaml \
  -e prod-env.yaml \
  -e secrets.yaml \
  web-app-stack

# Wait for completion (poll until COMPLETE or FAILED)
openstack stack create --wait \
  -t web-app.yaml \
  -e prod-env.yaml \
  web-app-stack

# Dry-run preview (shows what resources would be created; does not create)
openstack stack create --dry-run \
  -t web-app.yaml \
  -e prod-env.yaml \
  web-app-stack

# Disable rollback on failure (useful for debugging)
openstack stack create --disable-rollback \
  -t web-app.yaml \
  web-app-stack

# Set a custom timeout (minutes)
openstack stack create --timeout 60 \
  -t web-app.yaml \
  web-app-stack

# Tag the stack
openstack stack create \
  -t web-app.yaml \
  --tag env=production,team=platform \
  web-app-stack
```

### List Stacks

```bash
# List stacks in the current project
openstack stack list

# List all stacks (admin; all projects)
openstack stack list --all-projects

# Filter by status
openstack stack list --status CREATE_COMPLETE
openstack stack list --status ROLLBACK_COMPLETE

# Filter by tag
openstack stack list --tag env=production

# Show nested stacks
openstack stack list --nested

# Custom output columns
openstack stack list -c "Stack Name" -c "Stack Status" -c "Creation Time"
```

**Stack status values:**

| Status | Meaning |
|---|---|
| `CREATE_IN_PROGRESS` | Stack creation running |
| `CREATE_COMPLETE` | Stack created successfully |
| `CREATE_FAILED` | Stack creation failed; resources may be partially created |
| `UPDATE_IN_PROGRESS` | Stack update running |
| `UPDATE_COMPLETE` | Stack updated successfully |
| `UPDATE_FAILED` | Stack update failed |
| `DELETE_IN_PROGRESS` | Stack deletion running |
| `DELETE_COMPLETE` | Stack deleted (no longer visible in default list) |
| `DELETE_FAILED` | Stack deletion failed; some resources could not be deleted |
| `ROLLBACK_IN_PROGRESS` | Rolling back after a failed create or update |
| `ROLLBACK_COMPLETE` | Rollback completed; stack is in previous state |
| `ROLLBACK_FAILED` | Rollback itself failed |
| `SUSPEND_IN_PROGRESS` | Suspending all resources |
| `SUSPEND_COMPLETE` | All resources suspended |
| `RESUME_IN_PROGRESS` | Resuming all resources |
| `RESUME_COMPLETE` | All resources resumed |
| `CHECK_IN_PROGRESS` | Checking resource states |
| `CHECK_COMPLETE` | All resources verified against real-world state |
| `SNAPSHOT_IN_PROGRESS` | Creating a stack snapshot |
| `SNAPSHOT_COMPLETE` | Snapshot created |
| `RESTORE_IN_PROGRESS` | Restoring from snapshot |
| `RESTORE_COMPLETE` | Restore completed |

### Show Stack Details

```bash
# Full stack details (status, parameters, outputs, links)
openstack stack show web-app-stack

# Show as JSON
openstack stack show -f json web-app-stack

# Show stack outputs
openstack stack output list web-app-stack
openstack stack output show web-app-stack floating_ip
openstack stack output show web-app-stack db_connection_string

# Show the template that was used to create the stack
openstack stack template show web-app-stack
```

### Update a Stack

```bash
# Update with a new template
openstack stack update \
  -t web-app-v2.yaml \
  -e prod-env.yaml \
  web-app-stack

# Update a single parameter without changing the template
openstack stack update \
  --existing \
  --parameter instance_count=5 \
  web-app-stack

# Update and wait for completion
openstack stack update --wait \
  -t web-app-v2.yaml \
  web-app-stack

# Perform a dry-run update preview
openstack stack update --dry-run \
  -t web-app-v2.yaml \
  web-app-stack

# Update without rollback on failure
openstack stack update --disable-rollback \
  -t web-app-v2.yaml \
  web-app-stack
```

### Check, Suspend, and Resume

```bash
# Check — reconcile Heat's recorded state against the actual state of each resource
# (e.g. if someone deleted a server manually, check will mark it as gone)
openstack stack check web-app-stack

# Suspend all resources in a stack (puts instances in SUSPENDED state, etc.)
openstack stack suspend web-app-stack

# Resume a suspended stack
openstack stack resume web-app-stack
```

### Delete a Stack

```bash
# Delete a stack and all its resources
openstack stack delete web-app-stack

# Delete without confirmation prompt
openstack stack delete --yes web-app-stack

# Delete and wait for completion
openstack stack delete --wait --yes web-app-stack

# Abandon a stack (remove from Heat without deleting resources)
openstack stack abandon web-app-stack

# Adopt an existing set of resources into a new stack
# (re-import resources previously abandoned)
openstack stack adopt \
  -t web-app.yaml \
  --adopt-file abandon-output.json \
  web-app-stack
```

---

## Resource Operations

### List Resources

```bash
# List all resources in a stack
openstack stack resource list web-app-stack

# List resources including nested stacks
openstack stack resource list --nested-depth 3 web-app-stack

# Filter by resource status
openstack stack resource list --filter status=CREATE_FAILED web-app-stack

# List in a specific format
openstack stack resource list -f json web-app-stack
```

### Show a Resource

```bash
# Show resource details (type, status, physical ID, links)
openstack stack resource show web-app-stack web_server

# Show resource metadata
openstack stack resource metadata web-app-stack web_server

# Mark a resource as needing replacement on the next update
openstack stack resource mark-unhealthy web-app-stack web_server --reason "kernel panic"
```

### Signal a Resource

Some resources accept signals to trigger transitions (wait conditions, auto-scaling policies):

```bash
# Send a signal to a wait condition handle
openstack stack resource signal web-app-stack wait_handle

# Send a signal with JSON data
openstack stack resource signal \
  --data '{"status": "SUCCESS", "reason": "Setup complete", "id": "1"}' \
  web-app-stack wait_handle
```

---

## Event Operations

### List Events

```bash
# List events for a stack (newest first)
openstack stack event list web-app-stack

# List events including nested stacks
openstack stack event list --nested-depth 2 web-app-stack

# Follow events in real time (like tail -f)
openstack stack event list --follow web-app-stack

# List events for a specific resource
openstack stack event list web-app-stack --resource web_server

# Limit output
openstack stack event list --limit 50 web-app-stack
```

### Show an Event

```bash
# Show a specific event (by event ID from the list)
openstack stack event show web-app-stack web_server <event-id>
```

---

## Template Operations

### Validate a Template

```bash
# Validate template syntax and resource types
openstack orchestration template validate -t web-app.yaml

# Validate with environment
openstack orchestration template validate \
  -t web-app.yaml \
  -e prod-env.yaml

# Show resolved parameter types and constraints
openstack orchestration template validate \
  -t web-app.yaml \
  --show-nested
```

### List Resource Types

```bash
# List all available resource types
openstack orchestration resource type list

# Filter by name
openstack orchestration resource type list --filter name=OS::Nova

# Show schema (properties, attributes) for a specific resource type
openstack orchestration resource type show OS::Nova::Server
openstack orchestration resource type show OS::Neutron::Router
openstack orchestration resource type show OS::Heat::AutoScalingGroup
```

### Generate a Template Skeleton

```bash
# Generate a minimal template for a resource type
openstack orchestration template version list
openstack orchestration template generate \
  --template-type hot \
  OS::Nova::Server OS::Neutron::Net OS::Neutron::Subnet
```

---

## Stack Snapshots

```bash
# Create a snapshot of a stack (saves current resource state)
openstack stack snapshot create web-app-stack --name pre-upgrade-snapshot

# List snapshots
openstack stack snapshot list web-app-stack

# Show a snapshot
openstack stack snapshot show web-app-stack <snapshot-id>

# Restore from a snapshot
openstack stack snapshot restore web-app-stack <snapshot-id>

# Delete a snapshot
openstack stack snapshot delete web-app-stack <snapshot-id>
```

---

## Complete HOT Template Example

The following template creates a fully networked single-server application stack with:
- Private network, subnet, and router
- Security group allowing SSH and HTTP
- A Nova server
- A Cinder volume attached to the server
- A floating IP

```yaml
heat_template_version: 2021-04-16

description: |
  Single-server web application with private network, security group,
  persistent volume, and floating IP.

parameters:
  key_name:
    type: string
    description: SSH key pair for instance access
    constraints:
      - custom_constraint: nova.keypair

  flavor:
    type: string
    default: m1.small
    description: Instance flavor
    constraints:
      - custom_constraint: nova.flavor

  image:
    type: string
    default: ubuntu-24.04
    description: Boot image name or ID
    constraints:
      - custom_constraint: glance.image

  public_net:
    type: string
    default: public
    description: External network name for floating IPs

  private_net_cidr:
    type: string
    default: 192.168.100.0/24
    description: CIDR for the private subnet

  private_net_gateway:
    type: string
    default: 192.168.100.1
    description: Gateway address for the private subnet

  volume_size:
    type: number
    default: 20
    description: Data volume size in GB
    constraints:
      - range: {min: 1, max: 500}

  db_password:
    type: string
    description: Database password (hidden)
    hidden: true

resources:

  # ── Networking ─────────────────────────────────────────────────────────────

  private_network:
    type: OS::Neutron::Net
    properties:
      name: web-app-net

  private_subnet:
    type: OS::Neutron::Subnet
    properties:
      network: {get_resource: private_network}
      cidr: {get_param: private_net_cidr}
      gateway_ip: {get_param: private_net_gateway}
      dns_nameservers: [8.8.8.8, 8.8.4.4]
      allocation_pools:
        - start: 192.168.100.10
          end: 192.168.100.200

  router:
    type: OS::Neutron::Router
    properties:
      name: web-app-router
      external_gateway_info:
        network: {get_param: public_net}

  router_interface:
    type: OS::Neutron::RouterInterface
    properties:
      router: {get_resource: router}
      subnet: {get_resource: private_subnet}

  # ── Security ────────────────────────────────────────────────────────────────

  web_security_group:
    type: OS::Neutron::SecurityGroup
    properties:
      name: web-sg
      description: Allow SSH and HTTP/HTTPS inbound
      rules:
        - protocol: tcp
          port_range_min: 22
          port_range_max: 22
          remote_ip_prefix: 0.0.0.0/0
        - protocol: tcp
          port_range_min: 80
          port_range_max: 80
          remote_ip_prefix: 0.0.0.0/0
        - protocol: tcp
          port_range_min: 443
          port_range_max: 443
          remote_ip_prefix: 0.0.0.0/0
        - protocol: icmp
          remote_ip_prefix: 0.0.0.0/0

  # ── Compute ─────────────────────────────────────────────────────────────────

  server_port:
    type: OS::Neutron::Port
    properties:
      network: {get_resource: private_network}
      fixed_ips:
        - subnet: {get_resource: private_subnet}
      security_groups:
        - {get_resource: web_security_group}

  web_server:
    type: OS::Nova::Server
    properties:
      name: web-app-server
      image: {get_param: image}
      flavor: {get_param: flavor}
      key_name: {get_param: key_name}
      networks:
        - port: {get_resource: server_port}
      user_data_format: RAW
      user_data:
        str_replace:
          template: |
            #!/bin/bash
            set -ex
            apt-get update -y
            apt-get install -y nginx
            systemctl enable --now nginx
            echo "DB_PASSWORD=$db_password" >> /etc/app.env
          params:
            $db_password: {get_param: db_password}

  # ── Storage ──────────────────────────────────────────────────────────────────

  data_volume:
    type: OS::Cinder::Volume
    properties:
      name: web-app-data
      size: {get_param: volume_size}
      volume_type: __DEFAULT__

  volume_attachment:
    type: OS::Cinder::VolumeAttachment
    properties:
      volume_id: {get_resource: data_volume}
      instance_uuid: {get_resource: web_server}

  # ── Floating IP ──────────────────────────────────────────────────────────────

  floating_ip:
    type: OS::Neutron::FloatingIP
    properties:
      floating_network: {get_param: public_net}
      port_id: {get_resource: server_port}

outputs:

  floating_ip:
    description: Public floating IP address of the web server
    value: {get_attr: [floating_ip, floating_ip_address]}

  private_ip:
    description: Private IP address of the web server
    value: {get_attr: [web_server, first_address]}

  server_id:
    description: Nova instance UUID
    value: {get_resource: web_server}

  volume_id:
    description: Cinder volume UUID
    value: {get_resource: data_volume}
```

---

## Environment File Example

```yaml
# prod-env.yaml
# Apply with: openstack stack create -t main.yaml -e prod-env.yaml my-stack

parameters:
  # Exact values for this deployment
  key_name: prod-keypair
  flavor: m1.large
  image: ubuntu-24.04
  public_net: provider-net
  private_net_cidr: 10.20.0.0/24
  private_net_gateway: 10.20.0.1
  volume_size: 100
  db_password: MySuperSecretProdPassword!

parameter_defaults:
  # Fallback defaults; used when a parameter is not set by higher-priority sources
  flavor: m1.medium
  image: ubuntu-22.04
  volume_size: 20

resource_registry:
  # Override OS::Nova::Server with a custom organization template
  # (e.g., adds mandatory monitoring agents)
  MyOrg::Server: file:///etc/heat/templates/base-server.yaml
  # Prevent Heat from managing keypairs (they are pre-provisioned)
  OS::Nova::KeyPair: OS::Heat::None
```

---

## Heat Configuration Reference

Configuration file: `/etc/heat/heat.conf`

```ini
[DEFAULT]
# Number of engine worker processes to spawn
num_engine_workers = 4

# Maximum number of resources per stack
max_resources_per_stack = 1000

# Maximum number of stacks per project (0 = unlimited)
max_stacks_per_tenant = 100

# Default stack creation timeout (minutes)
stack_action_timeout = 60

# Enable convergence engine (default: true)
convergence_engine = true

# RPC transport URL (RabbitMQ)
transport_url = rabbit://heat:rabbit_pass@10.0.0.10:5672/

# Log to file
log_file = /var/log/heat/heat.log
log_dir = /var/log/heat

# Debug logging
debug = false

[heat_api]
# Bind address and port for heat-api
bind_host = 0.0.0.0
bind_port = 8004
workers = 4

[heat_api_cfn]
# Bind address and port for heat-api-cfn (CloudFormation compat)
bind_host = 0.0.0.0
bind_port = 8000
workers = 2

[database]
# SQLAlchemy connection string
connection = mysql+pymysql://heat:heat_db_pass@10.0.0.10/heat

# Connection pool settings
max_pool_size = 20
max_overflow = 20
pool_timeout = 30

[keystone_authtoken]
# Keystone middleware for heat-api authentication
www_authenticate_uri = http://10.0.0.10:5000
auth_url = http://10.0.0.10:5000
memcached_servers = 10.0.0.10:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = heat
password = heat_service_password

[clients_keystone]
# Keystone endpoint used by heat-engine for trust operations
auth_uri = http://10.0.0.10:5000

[trustee]
# Service user for creating and consuming Keystone trusts
auth_type = password
auth_url = http://10.0.0.10:5000
username = heat
password = heat_service_password
user_domain_id = default

[oslo_messaging_rabbit]
# RabbitMQ connection settings (if using oslo_messaging backend)
rabbit_ha_queues = true
heartbeat_timeout_threshold = 60

[oslo_policy]
enforce_scope = true
enforce_new_defaults = true

[paste_deploy]
api_paste_config = /etc/heat/api-paste.ini
```

---

## Service Management

```bash
# Start Heat services (systemd)
systemctl start openstack-heat-api
systemctl start openstack-heat-api-cfn
systemctl start openstack-heat-engine

# Enable on boot
systemctl enable openstack-heat-api
systemctl enable openstack-heat-api-cfn
systemctl enable openstack-heat-engine

# View logs
journalctl -u openstack-heat-engine -f
tail -f /var/log/heat/heat-engine.log

# Initialize or upgrade the database schema
heat-manage db_sync

# Clean up old stack events from the database
heat-manage purge_deleted -g 30  # purge events older than 30 days
```
