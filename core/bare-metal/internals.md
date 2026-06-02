# Ironic Internals

## Driver Composition Model

Every Ironic node is configured as a combination of a hardware type and a set of interface implementations. The hardware type declares which interfaces are supported; the operator selects one implementation per interface when creating or updating a node.

```
Node
 ├── hardware_type: "ipmi"
 ├── power_interface: "ipmitool"
 ├── management_interface: "ipmitool"
 ├── deploy_interface: "direct"
 ├── boot_interface: "ipxe"
 ├── inspect_interface: "inspector"
 ├── bios_interface: "no-bios"
 ├── raid_interface: "agent"
 ├── console_interface: "ipmitool-socat"
 └── vendor_interface: "ipmitool"
```

At runtime, `ironic-conductor` resolves the actual Python class for each interface and calls its methods. For example, when a power-on is requested:

```python
# ironic/conductor/manager.py
node = objects.Node.get_by_uuid(context, node_uuid)
driver = self._get_driver(node.driver)
power_iface = driver.power  # resolves to IPMIPower() for "ipmitool"
power_iface.validate(task)
power_iface.set_power_state(task, states.POWER_ON)
```

### Interface Resolution

When a node is created with `--driver ipmi` (hardware type) and `--boot-interface ipxe`, Ironic:

1. Loads the `IPMIHardware` class from the `ipmi` entrypoint
2. Verifies that `ipxe` is in `IPMIHardware.supported_boot_interfaces`
3. Stores `"ipxe"` in `node.boot_interface`
4. At runtime, loads `iPXEBoot` from the `ipxe` boot interface entrypoint

Interface implementations are registered as Python entry points in `setup.cfg`:

```ini
[entry_points]
ironic.hardware.types =
    ipmi = ironic.drivers.ipmi:IPMIHardware
    redfish = ironic.drivers.redfish:RedfishHardware

ironic.hardware.interfaces.boot =
    ipxe = ironic.drivers.modules.ipxe:iPXEBoot
    pxe = ironic.drivers.modules.pxe:PXEBoot
    redfish-virtual-media = ironic.drivers.modules.redfish.boot:RedfishVirtualMediaBoot
```

## Conductor Hash Ring

When multiple `ironic-conductor` processes run, node operations are distributed among them using a consistent hash ring. This prevents two conductors from simultaneously operating on the same node.

```
conductors registered in DB:
  conductor-01  conductor-02  conductor-03

Hash ring (tokens distributed across 360-degree ring):
  [conductor-01][conductor-02][conductor-03][conductor-01][conductor-02]...

For node UUID "abc123":
  hash("abc123") → position 147 → assigned to conductor-02
```

### Hash Ring Mechanics

- Each conductor registers itself in the `conductors` database table on startup with a heartbeat timestamp
- ironic-conductor updates its heartbeat every `heartbeat_interval` seconds (default 10 s)
- If a conductor's heartbeat is not updated within `heartbeat_timeout` seconds, it is considered dead and removed from the ring
- When the ring changes (conductor added or removed), node ownership shifts: some nodes are reassigned to a different conductor on the next periodic task run
- The conductor that "owns" a node is the only one that will operate on it during periodic tasks; API-triggered operations use `acquire_lock()` which also respects ring ownership

Node takeover when a conductor fails:

```
1. conductor-02 stops sending heartbeats
2. Other conductors detect conductor-02 is gone via DB check
3. Hash ring is recalculated; conductor-02's nodes are redistributed
4. conductor-01 and conductor-03 pick up the orphaned nodes
5. For any node stuck in a transitional state (deploying, cleaning),
   the new owner conductor runs a "takeover" procedure:
   - If IPA is running: re-attach to the IPA heartbeat
   - If no IPA response: fail the node and set provision_state=ERROR
```

## Deploy Steps and Clean Steps

Operations in Ironic are decomposed into ordered lists of *steps*. Each step is a method on an interface class with metadata:

```python
@METRICS.timer('AgentDeployMixin.write_image')
@base.deploy_step(priority=80, argsinfo={
    'image_url': {'description': 'URL of image to deploy'},
    'image_checksum': {'description': 'SHA512 checksum'},
})
def write_image(self, task, image_url=None, image_checksum=None):
    """Write an OS image to the target disk via IPA."""
    ...
```

Step metadata:

| Field | Meaning |
|---|---|
| `priority` | Execution order; higher priority runs first; 0 = disabled |
| `interface` | Which interface this step belongs to |
| `step` | Method name |
| `requires_ramdisk` | Whether IPA must be running on the node |
| `argsinfo` | Named arguments that can be passed via the API |

### Deploy Step Execution Order (default `direct` deploy)

```
priority 100:  deploy.deploy           (kick off the full deploy workflow)
priority 80:   deploy.write_image      (IPA writes image to disk)
priority 60:   deploy.inject_files     (inject authorized_keys, cloud-init data)
priority 40:   deploy.prepare_image    (convert qcow2 to raw if needed)
```

### Clean Step Execution Order (default automated clean)

```
priority 90:   management.reset_bios_to_default   (optional, if BIOS interface supports it)
priority 10:   deploy.erase_devices               (shred/blkdiscard all disks)
```

Operators can add custom clean steps by shipping a plugin that registers an interface with `@base.clean_step(priority=N)` methods.

### Manual Clean Step Syntax

The API accepts a list of step dicts:

```json
[
  {"interface": "deploy",     "step": "erase_devices",       "args": {}},
  {"interface": "raid",       "step": "delete_configuration", "args": {}},
  {"interface": "raid",       "step": "create_configuration", "args": {}},
  {"interface": "management", "step": "inject_nmi",          "args": {}}
]
```

Steps are executed in the order provided for manual cleaning. For automated cleaning, steps are sorted by `priority` descending across all active interfaces.

## IPA — Ironic Python Agent

IPA is a Python application that runs inside a lightweight ramdisk image on the bare metal node during deploy and clean operations. It provides a REST HTTP API that `ironic-conductor` calls to execute operations directly on the node's hardware.

### IPA Lifecycle During Deployment

```
1. Node is powered on
2. DHCP assigns an IP; PXE/iPXE boots the IPA ramdisk
3. IPA starts and performs "lookup":
   GET https://<ironic-api>:6385/v1/lookup?addresses=<mac1>,<mac2>
   → Returns the node UUID and conductor API endpoint

4. IPA sends periodic heartbeats:
   POST https://<ironic-api>:6385/v1/heartbeat/<node-uuid>
   {"callback_url": "https://<ipa-ip>:9999/v1", "agent_token": "..."}

5. ironic-conductor receives first heartbeat → "continue" signal
6. conductor calls IPA to start deploy steps:
   POST https://<ipa-ip>:9999/v1/commands
   {"name": "standby.cache_image", "params": {"image_info": {...}}}

7. IPA downloads the image, writes it to disk:
   POST https://<ipa-ip>:9999/v1/commands
   {"name": "deploy.write_image", ...}

8. conductor calls IPA to finalize (write config drive, set bootloader):
   POST https://<ipa-ip>:9999/v1/commands
   {"name": "deploy.install_bootloader", ...}

9. conductor tells IPA to reboot:
   POST https://<ipa-ip>:9999/v1/commands
   {"name": "system_passthrough.reboot", ...}

10. Node reboots into the deployed OS
11. ironic-conductor sets node provision_state=ACTIVE
```

### IPA Lookup

IPA discovers which Ironic node it is by looking up its MAC addresses:

```
GET /v1/lookup?addresses=aa:bb:cc:dd:ee:ff&addresses=aa:bb:cc:dd:ee:00
```

Ironic matches the MAC addresses against the ports table and returns the node UUID and configuration. If no match is found, the lookup returns 404 and IPA retries.

### IPA Heartbeat

IPA heartbeats every 10 seconds (configurable). The heartbeat carries:

- `callback_url`: the HTTPS URL where ironic-conductor can reach IPA
- `agent_token`: a one-time token issued by Ironic to authenticate the agent
- `version`: IPA version

The `agent_token` mechanism prevents rogue agents from hijacking node deployments.

### IPA Commands

IPA exposes a command execution API. Commands are asynchronous — the conductor polls for results:

```
POST /v1/commands
{"name": "deploy.write_image", "params": {...}}
→ {"command_name": "write_image", "command_status": "RUNNING", "command_result": null}

GET /v1/commands?wait=10
→ {"commands": [{"command_status": "SUCCEEDED", "command_result": {...}}]}
```

Key IPA command namespaces:

| Namespace | Commands |
|---|---|
| `deploy` | `write_image`, `prepare_image`, `install_bootloader`, `get_clean_steps`, `execute_clean_step` |
| `standby` | `cache_image`, `sync` |
| `system_passthrough` | `reboot`, `power_off`, `get_cpu_count`, `get_memory_info` |
| `raid` | `create_configuration`, `delete_configuration`, `get_logical_disks` |
| `image` | `get_os_install_command` (Anaconda deploy) |

## Provisioning State Machine

The full state machine with all valid transitions:

```
                            +----------+
                            |  enroll  | ◄── initial state after node create
                            +----------+
                                 │
                         manage  │  (verify driver credentials)
                                 ▼
                          +------------+
                     ┌───►│ verifying  │
                     │    +------------+
                     │         │ success
                     │         ▼
                     │   +------------+
                     │   │ manageable │ ◄──────────────────────────────────┐
                     │   +------------+                                    │
                     │        │ │                                          │
                     │provide │ │ inspect                                  │
                     │        │ ▼                                          │
                     │        │  +------------------+                      │
                     │        │  │    inspecting    │                      │
                     │        │  +------------------+                      │
                     │        │     │ success / fail                       │
                     │        │     ▼                                      │
                     │        │  +------------------+                      │
                     │        │  │  inspect failed  │ ─── manage ────────►│
                     │        │  +------------------+                      │
                     │        │ success                                    │
                     │        ▼                                            │
                     │   +-----------+                                     │
                     │   │ available │ ◄── (spare pool / Nova resource)   │
                     │   +-----------+                                     │
                     │        │ deploy (Nova or standalone)               │
                     │        ▼                                            │
                     │   +-----------+                                     │
                     │   │ deploying │                                     │
                     │   +-----------+                                     │
                     │        │ IPA starts / PXE boot                     │
                     │        ▼                                            │
                     │   +--------------+                                  │
                     │   │ wait call-   │ ◄── IPA heartbeat received      │
                     │   │ back         │                                  │
                     │   +--------------+                                  │
                     │        │ deploy complete / fail                    │
                     │    ┌───┴───────┐                                   │
                     │    │           │                                    │
                     │    ▼           ▼                                   │
                     │ +--------+  +---------------+                      │
                     │ │ active │  │ deploy failed │ ─── undeploy ───────►│
                     │ +--------+  +---------------+     or manage        │
                     │     │                                               │
                     │undeploy│  (Nova delete or standalone)               │
                     │        ▼                                            │
                     │  +----------+                                       │
                     │  │ deleting │  (tear-down, unplug network)          │
                     │  +----------+                                       │
                     │        │ automated clean starts                    │
                     │        ▼                                            │
                     │  +-----------+                                      │
                     │  │ cleaning  │                                      │
                     │  +-----------+                                      │
                     │        │ success                                    │
                     │        └──────────────────────────────────────────►│
                     │                                          (available)│
                     │                                                     │
                     │  +------------------+                               │
                     └──│  clean failed    │ ─── manage ─────────────────►│
                        +------------------+

Additional states:

  rescue ──► rescuing ──► rescue failed ──► manage ──► manageable
                │
                └──► rescued (node in rescue OS for operator access)

  manageable ──► servicing ──► service failed ──► manage ──► manageable
                     │
                     └──► available / active (post-service)
```

### Transition Table

| From State | Trigger | To State | Notes |
|---|---|---|---|
| `enroll` | `manage` | `verifying` | Conductor verifies BMC credentials |
| `verifying` | (success) | `manageable` | Credentials valid |
| `verifying` | (fail) | `enroll` | Credentials invalid; check driver-info |
| `manageable` | `inspect` | `inspecting` | Boot inspector ramdisk |
| `manageable` | `provide` | `cleaning` / `available` | Runs automated clean first |
| `manageable` | `clean` | `cleaning` | Manual clean |
| `inspecting` | (success) | `manageable` | Properties and ports populated |
| `inspecting` | (fail) | `inspect failed` | Check last_error |
| `inspect failed` | `manage` | `manageable` | Fix and retry |
| `available` | `deploy` | `deploying` | Boot node into IPA ramdisk |
| `deploying` | (IPA boot) | `wait call-back` | IPA sends heartbeat |
| `wait call-back` | (success) | `active` | Deploy steps completed |
| `wait call-back` | (fail) | `deploy failed` | Check last_error |
| `active` | `undeploy` | `deleting` | Tear down instance |
| `deleting` | (complete) | `cleaning` / `available` | Run automated clean |
| `deploy failed` | `undeploy` | `deleting` | Clean up partial deploy |
| `deploy failed` | `manage` | `manageable` | Skip clean, go to manageable |
| `cleaning` | (success) | `available` (auto) or `manageable` (manual) | Clean steps done |
| `cleaning` | (fail) | `clean failed` | Check last_error |
| `clean failed` | `manage` | `manageable` | Fix clean config and retry |
| `active` | `rescue` | `rescuing` | Boot rescue ramdisk for operator |
| `rescuing` | (success) | `rescued` | Node in rescue OS |
| `rescued` | `unrescue` | `active` | Return to deployed state |

## Periodic Tasks

ironic-conductor runs several periodic tasks:

| Task | Interval | Purpose |
|---|---|---|
| `_sync_power_states` | 60 s | Query BMC for actual power state; reconcile with DB; sync if mismatch |
| `_check_deploy_timeouts` | 60 s | Fail deployments that have exceeded `deploy_callback_timeout` |
| `_check_clean_timeouts` | 60 s | Fail cleanings that have exceeded `clean_callback_timeout` |
| `_sync_local_state` | 180 s | Recalculate hash ring; take over nodes from failed conductors |
| `_do_takeover` | 180 s | Attempt to re-attach to IPA agents on nodes in `wait call-back` |
| `_check_orphan_nodes` | 60 s | Find nodes whose conductor no longer exists; reassign |
| `handle_sync_power_status` | 60 s | Update `updated_at` for all active nodes |

## Node Locking

ironic-conductor uses database-level locking to serialize operations on a node. Before starting any operation, the conductor acquires an exclusive lock:

```python
with task_manager.acquire(context, node_uuid, shared=False) as task:
    # Only one conductor can hold this lock at a time
    task.driver.deploy.deploy(task)
```

`shared=True` is used for read-only operations (power state sync). `shared=False` is required for state changes. If a conductor crashes while holding a lock, the lock is automatically released after `node_locked_retry_attempt` retries.

## Config Drive

Ironic can inject a Nova-compatible config drive into deployed instances. The config drive is a small FAT32 filesystem image mounted at `/dev/disk/by-label/config-2` and contains:

- `/openstack/latest/meta_data.json` (instance metadata, SSH keys, hostname)
- `/openstack/latest/user_data` (cloud-init user-data)
- `/openstack/latest/network_data.json` (Neutron network configuration)

The config drive is generated by Nova (for Nova-managed deployments) or can be passed directly to the Ironic API (for standalone deployments):

```bash
# Generate and base64-encode a config drive
CONFIG_DRIVE=$(genisoimage -o - -input-charset utf-8 \
  -volid config-2 -joliet -rock \
  -graft-points openstack/latest/meta_data.json=meta_data.json | base64 -w0)

openstack baremetal node set compute-01 \
  --instance-info configdrive="$CONFIG_DRIVE"
```

## Fast-Track Deployment

Fast-track deployment (introduced in Stein, enabled by default in recent releases) avoids rebooting the node into IPA when it is already running IPA from a previous inspection or deployment attempt. This reduces deployment time from ~5 minutes to ~30 seconds for subsequent deployments:

```ini
[deploy]
fast_track = True
```

With fast-track enabled, if a node's IPA is already running and heartbeating, the conductor skips the PXE boot phase and proceeds directly to deploy steps.
