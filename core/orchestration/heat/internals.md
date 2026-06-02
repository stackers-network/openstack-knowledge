# Heat Internals

## Resource Plugin Interface

Every Heat resource type is implemented as a Python class that inherits from `heat.engine.resource.Resource`. The class defines the resource's schema (properties, attributes) and implements lifecycle hook methods.

### Core Lifecycle Methods

```python
class MyResource(resource.Resource):

    PROPERTIES = (
        NAME, FLAVOR, IMAGE,
    ) = (
        'name', 'flavor', 'image',
    )

    ATTRIBUTES = (
        INSTANCE_ID, IP_ADDRESS,
    ) = (
        'instance_id', 'ip_address',
    )

    properties_schema = {
        NAME: properties.Schema(
            properties.Schema.STRING,
            _('Name of the resource.'),
            required=True,
        ),
        FLAVOR: properties.Schema(
            properties.Schema.STRING,
            _('Resource flavor.'),
            required=True,
            constraints=[constraints.CustomConstraint('nova.flavor')],
        ),
        IMAGE: properties.Schema(
            properties.Schema.STRING,
            _('Boot image name or ID.'),
            required=True,
        ),
    }

    def handle_create(self):
        """
        Initiate the resource creation.
        Called once. Should start the async operation and return a
        handle (opaque object) that will be passed to check_create_complete.
        For synchronous resources, complete the operation here and return None.
        """
        props = self.properties
        server = self.client('nova').servers.create(
            name=props[self.NAME],
            flavor=props[self.FLAVOR],
            image=props[self.IMAGE],
        )
        return server.id  # handle passed to check_create_complete

    def check_create_complete(self, server_id):
        """
        Poll for completion. Called repeatedly until it returns True.
        Return True when the resource is ready, False to keep polling.
        Raise an exception to signal failure.
        """
        server = self.client('nova').servers.get(server_id)
        if server.status == 'ACTIVE':
            return True
        if server.status == 'ERROR':
            raise exception.ResourceInError(
                resource_status=server.status,
                status_reason=server.fault.get('message', 'Unknown error'),
            )
        return False  # still building — poll again

    def handle_update(self, json_snippet, tmpl_diff, prop_diff):
        """
        Called when a property has changed during a stack update.
        prop_diff contains only the properties that changed.
        Return a handle for check_update_complete, or None for sync operations.
        """
        if self.NAME in prop_diff:
            server = self.client('nova').servers.get(self.resource_id)
            server.update(name=prop_diff[self.NAME])
        return None

    def check_update_complete(self, handle):
        """Poll for update completion (same pattern as check_create_complete)."""
        return True  # synchronous update; always complete

    def handle_delete(self):
        """
        Initiate resource deletion. Return a handle for check_delete_complete.
        Must handle the case where the resource no longer exists (idempotent).
        """
        if self.resource_id is None:
            return None
        try:
            self.client('nova').servers.delete(self.resource_id)
        except Exception as e:
            self.client_plugin('nova').ignore_not_found(e)
        return self.resource_id

    def check_delete_complete(self, server_id):
        """Return True when the resource is fully deleted."""
        if server_id is None:
            return True
        try:
            self.client('nova').servers.get(server_id)
            return False  # still exists
        except Exception as e:
            self.client_plugin('nova').ignore_not_found(e)
            return True  # gone

    def handle_suspend(self):
        """Suspend the resource (optional)."""
        self.client('nova').servers.suspend(self.resource_id)

    def check_suspend_complete(self, handle):
        server = self.client('nova').servers.get(self.resource_id)
        return server.status == 'SUSPENDED'

    def handle_resume(self):
        """Resume the resource (optional)."""
        self.client('nova').servers.resume(self.resource_id)

    def check_resume_complete(self, handle):
        server = self.client('nova').servers.get(self.resource_id)
        return server.status == 'ACTIVE'

    def _resolve_attribute(self, name):
        """Return the value of a named attribute."""
        server = self.client('nova').servers.get(self.resource_id)
        if name == self.INSTANCE_ID:
            return server.id
        if name == self.IP_ADDRESS:
            return server.accessIPv4
```

### Resource Discovery

Resource plugins are registered via Python entry points in `setup.cfg`:

```ini
[entry_points]
heat.constraints =
    nova.keypair = heat.engine.clients.os.nova:KeypairConstraint
    nova.flavor = heat.engine.clients.os.nova:FlavorConstraint
    glance.image = heat.engine.clients.os.glance:ImageConstraint

heat.engine.resources =
    OS::Nova::Server = heat.engine.resources.openstack.nova.server:Server
    OS::Neutron::Net = heat.engine.resources.openstack.neutron.net:Net
```

Third-party plugins (e.g. for custom hardware or external services) can be installed separately and discovered automatically.

---

## Built-in Resource Types — Detailed Notes

### OS::Nova::Server

Key properties:
- `image`: Glance image name or UUID
- `flavor`: Nova flavor name or UUID
- `key_name`: SSH key pair name
- `networks`: list of `{network: ..., port: ..., subnet: ..., fixed_ip: ...}`
- `user_data`: cloud-init user data (string)
- `user_data_format`: `RAW`, `SOFTWARE_CONFIG`, or `HEAT_CFNTOOLS`
- `metadata`: instance metadata key-value pairs
- `availability_zone`: target AZ
- `block_device_mapping_v2`: advanced disk layout (boot from volume, multiple volumes)
- `personality`: inject files into the instance filesystem
- `config_drive`: force config drive (true/false)

Attributes: `instance_id`, `first_address`, `addresses`, `console_urls`, `name`

### OS::Neutron::Net / Subnet / Router / FloatingIP

Standard Neutron resource wrappers. `OS::Neutron::RouterInterface` does not produce a meaningful `resource_id`; it is a relationship resource only.

### OS::Cinder::Volume

- `size`: volume size in GB (required if not created from a snapshot or image)
- `volume_type`: Cinder volume type name
- `snapshot_id`: create from snapshot
- `source_volid`: create from existing volume (clone)
- `image`: create from Glance image
- `availability_zone`: AZ for the volume

### OS::Heat::WaitCondition and OS::Heat::WaitConditionHandle

`WaitConditionHandle` generates a pre-signed URL. The resource being configured (e.g. a Nova server user_data script) calls that URL with a JSON payload when it has finished setup. `WaitCondition` blocks the convergence traversal until the required number of signals arrives.

```yaml
wait_handle:
  type: OS::Heat::WaitConditionHandle

web_server:
  type: OS::Nova::Server
  properties:
    user_data_format: RAW
    user_data:
      str_replace:
        template: |
          #!/bin/bash
          # ... run setup ...
          curl -X POST -H 'Content-Type: application/json' \
            -d '{"status": "SUCCESS", "reason": "Setup done", "id": "1"}' \
            "$wc_url"
        params:
          $wc_url: {get_resource: wait_handle}

wait_condition:
  type: OS::Heat::WaitCondition
  depends_on: web_server
  properties:
    handle: {get_resource: wait_handle}
    count: 1
    timeout: 600
```

### OS::Heat::SoftwareConfig and OS::Heat::SoftwareDeployment

These resources decouple configuration definitions from deployments. `SoftwareConfig` holds a script or structured config and its input/output schema. `SoftwareDeployment` binds a config to a server and manages the deployment lifecycle.

```yaml
setup_config:
  type: OS::Heat::SoftwareConfig
  properties:
    group: script             # 'script', 'ansible', 'puppet', 'salt-minion', etc.
    inputs:
      - name: app_port
        type: Number
    outputs:
      - name: app_url
        type: String
    config: |
      #!/bin/bash
      echo "Listening on port $app_port" > /etc/app.conf
      echo "app_url=http://$(hostname -f):$app_port" > $heat_outputs_path

deploy_config:
  type: OS::Heat::SoftwareDeployment
  properties:
    config: {get_resource: setup_config}
    server: {get_resource: web_server}
    input_values:
      app_port: 8080
    signal_transport: TEMP_URL_SIGNAL   # or CFN_SIGNAL, HEAT_SIGNAL, ZAQAR_SIGNAL
```

Attributes from `SoftwareDeployment`: `deploy_status_code`, `deploy_stdout`, `deploy_stderr`, `deploy_signal_id`, and any outputs defined in the config.

`signal_transport` options:
- `TEMP_URL_SIGNAL`: Swift TempURL; requires Swift
- `CFN_SIGNAL`: CFN metadata endpoint; requires heat-api-cfn
- `HEAT_SIGNAL`: Heat API signal; requires Keystone
- `ZAQAR_SIGNAL`: Zaqar message queue signal
- `NO_SIGNAL`: synchronous; no signaling needed (for testing)

### OS::Heat::AutoScalingGroup and OS::Heat::ScalingPolicy

```yaml
asg:
  type: OS::Heat::AutoScalingGroup
  properties:
    resource:
      type: OS::Nova::Server
      properties:
        image: ubuntu-24.04
        flavor: m1.small
        key_name: {get_param: key_name}
        networks:
          - network: {get_resource: private_network}
    min_size: 1
    max_size: 10
    desired_capacity: 3
    cooldown: 60          # seconds between scale operations

scale_out_policy:
  type: OS::Heat::ScalingPolicy
  properties:
    adjustment_type: change_in_capacity    # or 'exact_capacity', 'percent_change_in_capacity'
    auto_scaling_group_id: {get_resource: asg}
    scaling_adjustment: 2                  # add 2 instances
    cooldown: 120

scale_in_policy:
  type: OS::Heat::ScalingPolicy
  properties:
    adjustment_type: change_in_capacity
    auto_scaling_group_id: {get_resource: asg}
    scaling_adjustment: -1                 # remove 1 instance
    cooldown: 120
```

The `ScalingPolicy` resource exposes an `alarm_url` attribute — a pre-signed webhook URL that triggers the policy when called. Point an Aodh alarm at this URL:

```bash
openstack alarm create \
  --type gnocchi_aggregation_by_metrics_threshold \
  --name cpu-high-alarm \
  --metric $(openstack metric metric list -c id -f value) \
  --threshold 80 \
  --comparison-operator gt \
  --aggregation-method mean \
  --granularity 300 \
  --alarm-action $(openstack stack output show my-stack scale_out_url -f value -c output_value)
```

### OS::Heat::ResourceGroup

Creates N copies of a resource definition. The `resource_def` is instantiated once per count. Each copy is addressable as `{get_resource: my_group}[0]`, `[1]`, etc.

```yaml
server_group:
  type: OS::Heat::ResourceGroup
  properties:
    count: 3
    resource_def:
      type: OS::Nova::Server
      properties:
        name: server-%index%
        image: ubuntu-24.04
        flavor: m1.small
        networks:
          - network: {get_resource: private_network}
```

---

## Dependency Graph Resolution

Heat builds a dependency graph for each template:

1. **Explicit dependencies** via `depends_on`:
   ```yaml
   resources:
     router_interface:
       type: OS::Neutron::RouterInterface
       depends_on: [router, subnet]
   ```

2. **Implicit dependencies** from intrinsic functions. Every `get_resource` and `get_attr` reference creates a directed edge. Heat traverses these statically at template load time.

3. **Cycle detection**: Heat validates that the dependency graph is a DAG. Circular dependencies (A depends on B depends on A) raise a `CircularDependencyException` during template validation.

4. **Parallel scheduling**: Once the DAG is built, resources whose all dependencies are satisfied are eligible for immediate scheduling. The convergence engine dispatches them to worker threads simultaneously.

Example DAG for the HOT template in operations.md:

```
floating_ip
    └── server_port
           ├── private_network
           └── web_security_group

web_server
    └── server_port

volume_attachment
    ├── web_server
    └── data_volume

router_interface
    ├── router
    └── private_subnet
           └── private_network
```

`private_network`, `router`, `web_security_group`, and `data_volume` have no dependencies and are created first in parallel. `private_subnet` waits only for `private_network`. `server_port` waits for `private_network` and `web_security_group`. And so on.

---

## Convergence Engine

### Traversal Records

For each stack operation (create, update, delete), the convergence engine writes a set of **traversal records** to the `sync_point` table in the Heat database. Each record represents one resource in one direction (forward/reverse).

The key fields:
- `stack_id` — the stack being operated on
- `traversal_id` — unique UUID per operation (distinguishes current from superseded operations)
- `is_update` — whether this is an update (vs initial create or delete)
- `resource_id` — which resource
- `requires` — set of resource IDs that must complete before this one

### Worker Polling Loop

```
while True:
    record = db.get_next_traversal_record()
    if record is None:
        sleep(poll_interval)
        continue
    resource = load_resource(record.resource_id)
    if all_dependencies_complete(record.requires):
        dispatch_to_worker(resource, record)
```

Workers are `eventlet` green threads within each `heat-engine` process. Multiple engine processes running on different hosts share the same database, providing both scale-out and fault tolerance.

### Stack Lock Mechanism

In the convergence model, there is no single stack-level lock. Instead, each resource has an **in-progress** flag in the `sync_point` table. A resource can only be acted on by one worker at a time. If two engine workers try to process the same resource, the database `SELECT FOR UPDATE` ensures only one proceeds.

For non-convergence operations (stack abandon, snapshot, restore), a **stack-wide lock** is acquired in the `stack_lock` table using an optimistic locking pattern (compare-and-swap on a `lock_id` UUID).

### Convergence and Stack Updates

When a stack update is triggered while a previous update (or create) is still running:

1. Heat generates a new `traversal_id` for the new operation.
2. Outstanding workers for the old traversal detect the mismatch and stop processing.
3. The new traversal takes over from the current resource states.

This prevents stale workers from interfering with the new intended state.

---

## Nested Stacks

A nested stack is a `OS::Heat::Stack` resource (or an environment registry mapping) that references a separate template. The child stack is a full stack in its own right, with its own resources, events, and outputs.

```yaml
# parent.yaml
resources:
  network_stack:
    type: OS::Heat::Stack
    properties:
      template: {get_file: network.yaml}
      parameters:
        cidr: 10.30.0.0/24

  app_stack:
    type: OS::Heat::Stack
    depends_on: network_stack
    properties:
      template: {get_file: app.yaml}
      parameters:
        network_id: {get_attr: [network_stack, outputs, network_id]}
```

Nested stacks appear in `openstack stack list --nested` and in event lists with `--nested-depth N`.

**Environment provider resources** are a higher-level pattern where an environment maps a custom type name to a template file, making nested stacks transparent to the user:

```yaml
# environment.yaml
resource_registry:
  MyOrg::NetworkStack: network.yaml
  MyOrg::AppStack: app.yaml
```

```yaml
# main.yaml (cleaner)
resources:
  network:
    type: MyOrg::NetworkStack
  app:
    type: MyOrg::AppStack
    depends_on: network
```

---

## Resource Signaling

Resources that accept signals implement `handle_signal(details)`:

```python
def handle_signal(self, details=None):
    """
    Called when a signal is received (via the Heat API or a pre-signed URL).
    details: dict with signal payload, or None.
    """
    if details and details.get('status') == 'FAILURE':
        raise SignalFailure('Resource reported failure: %s' % details.get('reason'))
    self._db_data['signal_count'] = self._db_data.get('signal_count', 0) + 1
    self.data_set('signal_count', str(self._db_data['signal_count']))
```

`WaitConditionHandle` signals arrive at Heat's resource signal endpoint:

```
POST /v1/{tenant_id}/stacks/{stack_name}/{stack_id}/resources/{resource_name}/signal
```

or via a pre-signed Swift TempURL (no Keystone token required — used from within instances).

---

## Software Deployment Chain

The full chain for `OS::Heat::SoftwareDeployment` with `signal_transport: TEMP_URL_SIGNAL`:

```
heat-engine creates SoftwareConfig
       │
       ▼
heat-engine creates SoftwareDeployment
  → generates a Swift TempURL for signal delivery
  → writes deployment metadata to the server's Heat metadata endpoint
       │
       ▼
Nova server boots
  → heat-config-notify or os-apply-config (via os-collect-config)
     polls the Heat metadata API endpoint
     retrieves deployment metadata
       │
       ▼
heat-config hook (e.g., bash, ansible, puppet) runs the config script
  → reads inputs from deployment metadata
  → executes the script/playbook
  → on completion, POSTs result JSON to the Swift TempURL
       │
       ▼
Swift accepts the signal → Heat is notified
  → SoftwareDeployment transitions to COMPLETE (or FAILED)
  → outputs become available via get_attr
```

Required packages on the guest OS:
- `os-collect-config` — polls for Heat metadata
- `os-apply-config` — applies config to the OS
- `os-refresh-config` — triggers re-application on metadata change
- `heat-config` — dispatcher for hook types (script, ansible, puppet, etc.)

---

## Database Schema (Key Tables)

| Table | Purpose |
|---|---|
| `stack` | One row per stack (name, status, template, environment, parameters) |
| `resource` | One row per resource per stack (type, status, physical_resource_id) |
| `resource_data` | Key-value store for resource-private state (used by plugins) |
| `event` | Stack and resource lifecycle events |
| `raw_template` | Stored template JSON/YAML blobs |
| `user_creds` | Keystone trust credentials for deferred operations |
| `software_config` | SoftwareConfig resource definitions |
| `software_deployment` | SoftwareDeployment instances and their status |
| `sync_point` | Convergence traversal records (per-resource, per-operation) |
| `stack_lock` | Stack-wide operation locks (for snapshot/abandon) |
| `snapshot` | Stack snapshot records |
