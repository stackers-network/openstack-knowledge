# Octavia Internals

## Amphora Lifecycle

The amphora is the central primitive in Octavia's amphora provider. Every load balancer backed by the amphora provider maps to one or more Nova VMs running HAProxy. Understanding the full lifecycle is essential for troubleshooting and capacity planning.

### Lifecycle States

```
           ┌──────────────────────────────────────────────────────┐
           │                  Amphora States                      │
           │                                                      │
  boot ──► BOOTING ──► READY ──► ALLOCATED ──► PENDING_DELETE    │
                         ▲           │                │           │
                         │     (assigned to LB)       ▼           │
                   housekeeping   (in use)        DELETED ──► GC  │
                   replenishes                                     │
                                                                   │
  ERROR ◄── heartbeat timeout ── ALLOCATED                       │
    │                                                             │
    └──► failover triggers ──► new amphora in BOOTING            │
           └─────────────────────────────────────────────────────┘
```

### Phase 1: Boot

When a load balancer is created (or a failover is triggered), octavia-worker executes the `create_amphora` Taskflow:

1. Check the spare pool — if a `READY` amphora exists, claim it (transition `READY → ALLOCATED`) and skip to Phase 3
2. Call `POST /servers` on Nova to boot a new VM using:
   - The pre-configured amphora Nova flavor (set in `octavia.conf [controller_worker] amp_flavor_id`)
   - The pre-uploaded amphora Glance image (`amp_image_tag`)
   - A Neutron port on the management network (`amp_network`)
   - A cloud-init or user-data script that starts the amphora-agent
3. Wait for Nova to report the VM `ACTIVE` and for the management port to become `ACTIVE`
4. Record the management IP in the `amphora` database table with status `BOOTING`

### Phase 2: Configure Networking

Once the VM is up, octavia-worker:

1. Plugs the VIP network into the amphora: calls the amphora REST agent `POST /plug/vip/<vip-address>` which:
   - Creates a Neutron port on the VIP subnet
   - Attaches it to the amphora VM via `nova interface-attach`
   - Configures the interface inside the VM with the VIP address and routes
2. For active-standby: plugs a second interface for the keepalived heartbeat network

### Phase 3: Configure HAProxy

octavia-worker calls the amphora REST agent to write the HAProxy configuration:

```
POST https://<amphora-mgmt-ip>:9443/0.5/loadbalancer/<lb-id>/haproxy
Content-Type: application/json

{"config": "<haproxy.cfg content>"}
```

The amphora agent writes the config to `/var/lib/octavia/<lb-id>/haproxy.cfg` and starts (or reloads) HAProxy. HAProxy is managed as a per-listener process, allowing individual listeners to reload without affecting others.

A typical generated HAProxy config:

```
global
    daemon
    log /dev/log local0
    stats socket /var/lib/octavia/<lb-id>/haproxy.sock mode 0660

defaults
    log global
    retries 3
    option redispatch

frontend http_listener_<uuid>
    bind 192.168.1.100:80
    default_backend pool_<pool-uuid>

backend pool_<pool-uuid>
    balance roundrobin
    option httpchk GET /healthz
    server web-1_192.168.1.10 192.168.1.10:8080 check inter 5s fall 3 rise 3
    server web-2_192.168.1.11 192.168.1.11:8080 check inter 5s fall 3 rise 3
```

### Phase 4: Heartbeat

Once active, the amphora-agent sends a UDP heartbeat every `heartbeat_interval` seconds (default 10 s) to the health-manager. The heartbeat payload is a JSON struct (encrypted with the heartbeat key) containing:

```json
{
  "id": "<amphora-uuid>",
  "seq": 12345,
  "listeners": {
    "<listener-uuid>": {
      "status": "OPEN",
      "stats": {
        "active_connections": 42,
        "total_connections": 8192,
        "bytes_in": 5242880,
        "bytes_out": 10485760
      }
    }
  }
}
```

health-manager decrypts and processes each heartbeat:
- Updates `amphora.updated_at` in the database
- Updates listener and pool `operating_status`
- Updates per-member `operating_status` based on HAProxy stats socket data (which the amphora-agent reads each heartbeat interval)

### Phase 5: Failover

Failover is triggered either automatically (when health-manager detects heartbeat timeout) or manually:

1. `health-manager` sets the amphora to `ERROR` state and publishes a failover event to RabbitMQ
2. `octavia-worker` picks up the failover job and executes the `failover_amphora` Taskflow:
   a. Boot a new amphora VM (from spare pool if available)
   b. Plug the same VIP network into the new amphora
   c. Replicate the full HAProxy configuration from the database (not from the failed VM)
   d. For active-standby: update keepalived on the surviving peer to point to the new amphora
   e. Set the new amphora to `ALLOCATED` state
   f. Mark the old amphora for deletion (housekeeping will clean it up)
3. The load balancer's `provisioning_status` returns to `ACTIVE`

For active-standby HA pairs, the standby amphora absorbs traffic via VRRP within 1–2 seconds while Octavia provisions a replacement. The VIP does not go down during failover in this topology.

### Phase 6: Delete

When a load balancer is deleted:

1. octavia-worker calls the amphora REST agent to stop HAProxy and unplug the VIP
2. Neutron VIP port is deleted
3. Nova VM is terminated
4. Amphora record is marked `DELETED`
5. housekeeping garbage-collects the record after `delete_amphora_later_sec` seconds (default 3600)

## Taskflow-Based Operations

All state-changing operations in Octavia are implemented as Taskflow linear (or graph) flows. Taskflow provides:

- **Persistence**: each task's progress is saved in the database; if octavia-worker crashes mid-operation, the flow can be resumed by another worker
- **Reversion**: if a task fails, Taskflow calls `revert()` on completed tasks in reverse order, cleaning up partially created resources
- **Retry**: individual tasks can be configured with retry policies

Example flow for `CREATE_LOAD_BALANCER`:

```
Flow: create_load_balancer_flow
├── task: GetOrCreateAmphoraForLoadBalancer
│     revert: DeleteAmphoraFromDB
├── task: AllocateVIP
│     revert: DeallocateVIP
├── task: PlugVIPAmphora
│     revert: UnplugVIP
├── task: MapLoadbalancerToAmphora
├── task: GenerateServerPEMFile
├── task: UpdateAmphoraWithCerts
├── task: MarkLBActiveInDB
└── task: MarkAmphoraAllocatedInDB
```

Flows are stored in the `persistence` database (or in the main Octavia DB). List in-flight flows:

```bash
# Check for flows in ERROR state
SELECT * FROM taskflow_flowdetails WHERE state = 'FAILURE';
```

## Provider Driver Framework

Octavia defines a driver interface (`octavia_lib.api.drivers.driver_lib`) that all provider drivers must implement. The interface methods map directly to API operations:

```python
class ProviderDriver:
    def create_vip_port(self, loadbalancer_id, project_id, vip_dictionary): ...
    def loadbalancer_create(self, loadbalancer): ...
    def loadbalancer_delete(self, loadbalancer, cascade=False): ...
    def loadbalancer_update(self, original_load_balancer, new_loadbalancer): ...
    def listener_create(self, listener): ...
    def listener_delete(self, listener): ...
    def listener_update(self, original_listener, new_listener): ...
    def pool_create(self, pool): ...
    def pool_delete(self, pool): ...
    def member_create(self, member): ...
    def member_delete(self, member): ...
    def member_update(self, original_member, new_member): ...
    def health_monitor_create(self, healthmonitor): ...
    def get_supported_flavor_metadata(self): ...
```

The amphora provider implements every method. The OVN provider implements only the subset that OVN's native load balancer supports. When a tenant creates a load balancer with `--provider ovn`, Octavia routes all subsequent child resource operations to the OVN driver.

Drivers report capability metadata used to validate flavor profiles:

```python
def get_supported_flavor_metadata(self):
    return {
        "compute_flavor": "Nova flavor to use for amphorae.",
        "loadbalancer_topology": "SINGLE or ACTIVE_STANDBY.",
    }
```

## Health Monitoring Deep-Dive

The health-manager is the only service that receives data from amphorae (all other communication is worker → amphora). The heartbeat flow:

```
amphora-agent                          health-manager
    │                                       │
    │── UDP heartbeat (every 10s) ─────────►│
    │   (AES-256-CBC encrypted payload)     │
    │                                       │── decrypt payload
    │                                       │── update amphora.updated_at
    │                                       │── parse listener stats
    │                                       │── compare member states
    │                                       │── UPDATE operating_status in DB
    │                                       │
    │  [no heartbeat for 60s]               │── mark amphora ERROR
    │                                       │── publish failover event
```

The encryption key (`heartbeat_key`) is set in `octavia.conf` and must match between amphora-agent and health-manager. Rotate with:

```bash
# Generate new key
openssl rand -hex 32

# Update octavia.conf [health_manager] heartbeat_key
# Restart health-manager
# The amphora-agent picks up new keys via cert rotation (automated by housekeeping)
```

health-manager also reads HAProxy stats socket data forwarded by the amphora-agent to determine per-member health (beyond what HAProxy check results alone provide).

## TLS Certificate Management

### Amphora REST Agent Certificates

Communication between octavia-worker and the amphora REST agent uses mutual TLS:

- Octavia acts as its own CA (`[certificates] ca_certificate`, `ca_private_key`)
- During amphora provisioning, octavia-worker generates a server cert for the amphora and a client cert for itself
- The amphora-agent is configured with the server cert; octavia-worker presents its client cert on every HTTPS call
- housekeeping rotates certs before `cert_expiry_buffer` days before expiry (default 14 days)

### TERMINATED_HTTPS Listener Certificates

When a listener uses `TERMINATED_HTTPS` protocol, HAProxy terminates TLS and forwards plain HTTP to back-end members. The TLS certificate is stored in Barbican:

```bash
# Create a PKCS12 bundle from cert + key
openssl pkcs12 -export \
  -in server.crt \
  -inkey server.key \
  -certfile ca-chain.crt \
  -out bundle.p12 \
  -passout pass:

# Store in Barbican as a certificate container
openstack secret store \
  --name web-cert \
  --payload-content-type "application/octet-stream" \
  --payload-content-encoding base64 \
  --payload $(base64 -w0 bundle.p12)

# Reference in the listener
openstack loadbalancer listener create \
  --name https-listener \
  --protocol TERMINATED_HTTPS \
  --protocol-port 443 \
  --default-tls-container-ref $(openstack secret list --name web-cert -f value -c "Secret href") \
  web-lb
```

SNI (Server Name Indication) is supported via `--sni-container-refs` to serve multiple certificates on one listener port.

## Octavia Flavors Deep-Dive

Flavors give operators fine-grained control over what infrastructure backs each load balancer. The data model:

```
FlavorProfile
 ├── provider_name: "amphora"
 └── flavor_data (JSON):
       ├── compute_flavor: "<nova-flavor-id>"    # size of amphora VM
       ├── loadbalancer_topology: "SINGLE"        # or "ACTIVE_STANDBY"
       ├── amp_image_tag: "amphora-image-v2"     # override default image
       └── availability_zone: "nova-az1"          # pin amphora placement

Flavor
 ├── flavorprofile_id → FlavorProfile
 ├── name: "ha-medium"
 ├── description: "HA medium load balancer"
 └── is_public: True
```

When octavia-worker processes a `CREATE_LOAD_BALANCER` task for a load balancer with a flavor:

1. Load the `FlavorProfile` from the database
2. Parse `flavor_data` JSON
3. Override the global `octavia.conf` defaults with flavor-specific values (Nova flavor, topology, image tag)
4. Proceed with the normal amphora boot flow using those overridden values

This allows a single Octavia deployment to offer `small` (m1.small, SINGLE), `medium` (m1.medium, SINGLE), and `ha-large` (m1.xlarge, ACTIVE_STANDBY) tiers to tenants without separate deployments.

## HAProxy Reload vs Restart

Octavia uses HAProxy's zero-downtime reload mechanism when updating configuration (adding/removing members, changing weights, etc.):

1. octavia-worker pushes the new config to the amphora REST agent
2. The amphora-agent validates the config with `haproxy -c`
3. The agent sends `SIGUSR2` to the running HAProxy master process
4. HAProxy forks a new worker with the new config; existing connections are drained on the old worker
5. Once all old connections are closed, the old worker exits

This means member additions and pool changes are live within seconds with no dropped connections.

## Configuration Reference

Key `octavia.conf` sections:

```ini
[controller_worker]
amp_flavor_id = <nova-flavor-uuid>        # Nova flavor for amphorae
amp_image_tag = amphora                   # Glance tag for amphora image
amp_ssh_key_name = octavia_ssh_key        # Nova key pair for debug SSH access
amp_network = <mgmt-network-uuid>         # Management network
amp_secgroup_list = <sg-uuid>             # Security groups for amphorae
workers = 4                               # Concurrent Taskflow workers

[health_manager]
bind_ip = 192.168.200.1                   # IP of management interface on HM host
bind_port = 5555                          # UDP port for heartbeats
heartbeat_interval = 10                   # Seconds between amphora heartbeats
heartbeat_timeout = 60                    # Seconds before amphora marked ERROR
health_update_threads = 4

[house_keeping]
spare_amphora_pool_size = 2
cleanup_interval = 30                     # Seconds between housekeeping runs
amphora_expiry_age = 86400               # Age before expired amphorae are purged

[certificates]
cert_manager = barbican_cert_manager
ca_certificate = /etc/octavia/certs/ca_01.pem
ca_private_key = /etc/octavia/certs/ca_01.key
cert_expiry_buffer = 14                   # Days before expiry to rotate
```
