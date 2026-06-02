# Heat Architecture

## Heat Services

Heat is composed of three discrete processes that communicate over an oslo.messaging bus (typically RabbitMQ). Each service can be scaled independently.

```
                  Client (CLI / Horizon / Magnum)
                           │
                    ┌──────┴──────┐
                    │  heat-api   │  ← REST API (port 8004)
                    │  (WSGI)     │
                    └──────┬──────┘
                           │ oslo.messaging RPC
                    ┌──────┴──────┐
                    │ heat-engine │  ← Resource orchestration
                    │             │  (multiple workers)
                    └──────┬──────┘
                           │
          ┌────────────────┼──────────────────────┐
          │                │                      │
       Nova API       Neutron API           Cinder API
       (servers)      (networks)            (volumes)
```

### heat-api

The HTTP API server. Accepts REST requests from clients, validates them against Keystone, enforces oslo.policy, and dispatches work to `heat-engine` over RPC. Runs as a WSGI application (Apache mod_wsgi or uWSGI).

- Exposes the Heat Orchestration API v1 on port **8004**
- Handles template validation (calling the engine for full validation)
- Handles stack preview (dry-run resource resolution)
- All resource-mutating operations are dispatched asynchronously to the engine

**Endpoint registration (example):**
```bash
openstack service create --name heat --description "Orchestration" orchestration
openstack endpoint create --region RegionOne orchestration public   http://10.0.0.10:8004/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne orchestration internal http://10.0.0.11:8004/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne orchestration admin    http://10.0.0.11:8004/v1/%\(tenant_id\)s
```

### heat-api-cfn

An optional compatibility endpoint that accepts **AWS CloudFormation-compatible** API requests. Translates CloudFormation API calls and templates into native Heat operations.

- Exposes the CFN-compatible API on port **8000**
- Supports CloudFormation template format (JSON or YAML)
- Required if migrating AWS CloudFormation templates or using tools that emit CFN API calls
- Not required for HOT-only deployments; disabled in many modern deployments
- Uses the same `heat-engine` backend

**Endpoint registration:**
```bash
openstack service create --name heat-cfn --description "Orchestration CloudFormation" cloudformation
openstack endpoint create --region RegionOne cloudformation public http://10.0.0.10:8000/v1
```

### heat-engine

The orchestration engine. Receives RPC calls from `heat-api`, resolves the template, builds the resource dependency graph, and drives resource operations through to completion. Handles the full stack lifecycle.

- Runs as a long-lived daemon (typically multiple workers, `num_engine_workers`)
- Each engine worker processes stack operations independently
- Manages the **convergence traversal** (see below)
- Calls OpenStack service APIs (Nova, Neutron, Cinder, Glance, etc.) directly
- Writes stack, resource, and event records to the Heat database
- Uses Keystone trusts to perform deferred operations (auto-scaling, wait conditions) as the stack owner

**Summary table:**

| Service | Default Port | Notes |
|---|---|---|
| heat-api | 8004 | REST API for Heat clients |
| heat-api-cfn | 8000 | AWS CloudFormation compat; optional |
| heat-engine | — (RPC only) | Multiple workers; scale horizontally |

---

## HOT Template Format

HOT (Heat Orchestration Template) is the native Heat template format. Templates are YAML or JSON documents.

### Top-Level Sections

```yaml
heat_template_version: 2021-04-16    # Required. Determines available features.

description: |                        # Optional. Human-readable description.
  Multi-tier web application stack.

parameter_groups:                     # Optional. UI grouping hints.
  - label: Network Configuration
    parameters:
      - public_net
      - private_net_cidr

parameters:                           # Input parameters (values injected at create/update).
  key_name:
    type: string
    description: SSH key pair name
    constraints:
      - custom_constraint: nova.keypair

  image:
    type: string
    default: ubuntu-24.04
    constraints:
      - custom_constraint: glance.image

  flavor:
    type: string
    default: m1.small
    constraints:
      - custom_constraint: nova.flavor

  db_password:
    type: string
    hidden: true                      # Masked in API responses

  public_net:
    type: string
    default: public

  private_net_cidr:
    type: string
    default: 192.168.100.0/24

conditions:                           # Optional. Boolean expressions for conditional resources.
  enable_monitoring:
    equals:
      - get_param: environment
      - production

resources:                            # Required. The resource definitions.
  my_server:
    type: OS::Nova::Server
    properties:
      ...

outputs:                              # Optional. Values to expose after creation.
  server_ip:
    description: Floating IP of the web server
    value: { get_attr: [floating_ip, floating_ip_address] }
```

### Template Versions

| Version String | Release | Key Features Added |
|---|---|---|
| `2013-05-23` | Icehouse | Initial HOT version |
| `2014-10-16` | Juno | `repeat` function, `conditions` preview |
| `2015-04-30` | Kilo | `conditions`, `if` function |
| `2016-04-08` | Newton | `equals`, `not`, `and`, `or` condition functions |
| `2016-10-14` | Ocata | `yaql` function |
| `2017-02-24` | Pike | `merge_list`, `contains` |
| `2018-08-31` | Rocky | `digest` function |
| `2021-04-16` | Wallaby | Latest stable; recommended for new templates |

Always use `heat_template_version: 2021-04-16` for new templates. Older versions are still parsed but may lack features.

### Intrinsic Functions

| Function | Description | Example |
|---|---|---|
| `get_param` | Read a parameter value | `{get_param: key_name}` |
| `get_resource` | Get the reference (ID) of another resource | `{get_resource: my_network}` |
| `get_attr` | Read an attribute from a resource | `{get_attr: [server, first_address]}` |
| `get_file` | Embed a local file's contents as a string | `{get_file: scripts/setup.sh}` |
| `str_replace` | String substitution with named tokens | `{str_replace: {template: "Hello $name", params: {$name: World}}}` |
| `list_join` | Join a list into a string | `{list_join: [',', [a, b, c]]}` |
| `repeat` | Generate a list by iterating over values | `{repeat: {template: ..., for_each: {%item%: [1, 2, 3]}}}` |
| `if` | Conditional value selection | `{if: [enable_monitoring, m1.medium, m1.small]}` |
| `equals` | Equality test (for conditions) | `{equals: [{get_param: env}, prod]}` |
| `not` | Boolean negation | `{not: {equals: [...]}}` |
| `and` / `or` | Boolean combination | `{and: [cond1, cond2]}` |
| `digest` | Compute hash of a value | `{digest: [sha256, {get_param: secret}]}` |
| `yaql` | Evaluate a YAQL expression | `{yaql: {expression: "$.data.len()", data: [1,2,3]}}` |

---

## Resource Types

Heat ships with a large library of built-in resource types. The most commonly used:

### Compute

| Type | Description |
|---|---|
| `OS::Nova::Server` | Virtual machine instance |
| `OS::Nova::KeyPair` | SSH key pair |
| `OS::Nova::ServerGroup` | Server group (affinity/anti-affinity) |
| `OS::Nova::Flavor` | Custom flavor definition (admin only) |

### Networking

| Type | Description |
|---|---|
| `OS::Neutron::Net` | Tenant network |
| `OS::Neutron::Subnet` | Subnet within a network |
| `OS::Neutron::Router` | L3 router |
| `OS::Neutron::RouterInterface` | Attach a subnet to a router |
| `OS::Neutron::FloatingIP` | Allocate a floating IP |
| `OS::Neutron::FloatingIPAssociation` | Assign floating IP to a port |
| `OS::Neutron::Port` | Neutron port with specific IP/security groups |
| `OS::Neutron::SecurityGroup` | Security group |
| `OS::Neutron::SecurityGroupRule` | Individual security group rule |

### Block Storage

| Type | Description |
|---|---|
| `OS::Cinder::Volume` | Persistent block volume |
| `OS::Cinder::VolumeAttachment` | Attach a volume to a server |
| `OS::Cinder::EncryptedVolumeType` | Encrypted volume type |

### Orchestration / Heat-specific

| Type | Description |
|---|---|
| `OS::Heat::Stack` | Nested stack (references another template) |
| `OS::Heat::ResourceGroup` | Create N copies of a resource |
| `OS::Heat::AutoScalingGroup` | Auto-scaling group of resources |
| `OS::Heat::ScalingPolicy` | Scale-out/scale-in policy for an ASG |
| `OS::Heat::WaitCondition` | Block until a signal is received |
| `OS::Heat::WaitConditionHandle` | URL endpoint for wait condition signals |
| `OS::Heat::SoftwareConfig` | Cloud-init or config management script definition |
| `OS::Heat::SoftwareDeployment` | Apply a SoftwareConfig to a server |
| `OS::Heat::SoftwareDeploymentGroup` | Apply a SoftwareConfig to multiple servers |
| `OS::Heat::StructuredConfig` | JSON-structured SoftwareConfig |
| `OS::Heat::StructuredDeployment` | Apply StructuredConfig to a server |
| `OS::Heat::RandomString` | Generate a random string (passwords, secrets) |
| `OS::Heat::Value` | Store and expose a computed value |

### Load Balancing

| Type | Description |
|---|---|
| `OS::Octavia::LoadBalancer` | Octavia load balancer |
| `OS::Octavia::Listener` | Frontend listener |
| `OS::Octavia::Pool` | Backend server pool |
| `OS::Octavia::PoolMember` | Add a server to a pool |
| `OS::Octavia::HealthMonitor` | Pool health check |

### List all available resource types

```bash
openstack orchestration resource type list
openstack orchestration resource type show OS::Nova::Server
```

---

## Environments

An environment is a YAML file that augments a template with:

- **Parameter defaults** (override template defaults without modifying the template)
- **Parameter values** (inject values at stack create/update)
- **Resource registry** (map type aliases to custom templates or existing resources)
- **Event sinks** (route resource events to webhooks)

### Environment File Structure

```yaml
# prod-env.yaml
parameters:
  flavor: m1.large
  image: ubuntu-24.04
  key_name: prod-keypair
  public_net: provider-net
  private_net_cidr: 10.10.0.0/24
  db_password: supersecretpassword

parameter_defaults:
  # Applied when a parameter is not supplied by the user or a higher-priority env
  flavor: m1.small
  image: ubuntu-22.04

resource_registry:
  # Map a type alias to a custom nested template
  MyOrg::WebServer: file:///etc/heat/templates/webserver.yaml
  # Map to a provider URL
  MyOrg::DBCluster: https://example.com/templates/db-cluster.yaml
  # Freeze an existing resource (prevent Heat from managing it)
  OS::Nova::KeyPair: OS::Heat::None

event_sinks:
  - type: zaqar-queue
    target_queue: heat-events
    ttl: 86400
```

### Multiple Environments

Multiple environment files are applied in order; later files override earlier ones:

```bash
openstack stack create \
  -t main.yaml \
  -e base-env.yaml \
  -e prod-env.yaml \
  -e secrets.yaml \
  my-stack
```

---

## Convergence Architecture

Heat's convergence engine (introduced in Newton, default since Ocata) manages stack operations using a **directed acyclic graph (DAG)** of resource nodes and performs operations in parallel wherever dependencies allow.

### How Convergence Works

1. **Template parsing**: Heat parses the template and resolves all intrinsic functions to build a dependency graph where each resource is a node and `get_resource` / `get_attr` references are directed edges.

2. **Graph traversal**: Heat generates a set of **resource traversal records** in the database, one per resource per direction (forward for create/update, reverse for delete). Each record acts as a work item for engine workers.

3. **Worker polling**: Engine workers pick up traversal records. A resource is processed only when all its dependencies have completed (forward direction) or all its dependents have completed (reverse direction).

4. **Parallel execution**: Resources with no outstanding dependencies are processed simultaneously across multiple engine workers.

5. **State machine**: Each resource transitions through states: `IN_PROGRESS → COMPLETE` (success) or `IN_PROGRESS → FAILED` (failure). The overall stack state reflects the aggregate of all resource states.

### Convergence vs Legacy Engine

| Aspect | Convergence (default) | Legacy (deprecated) |
|---|---|---|
| Resource ordering | Parallel, graph-based | Sequential, depth-first |
| Update strategy | In-place with graph diff | Full re-traversal |
| Lock granularity | Per-resource | Per-stack (global lock) |
| Failed resource handling | Other independent resources continue | Stack halts on first failure |
| Database writes | Traversal records per resource | Stack-level state |

### Stack Update with Convergence

During an update Heat computes the **diff** between the old and new template dependency graphs:

- Resources present in both templates are updated in-place (if properties changed) or left unchanged.
- Resources added to the new template are created.
- Resources removed from the new template are deleted.
- The update traversal runs forward (for new/changed resources) and reverse (for deleted resources) concurrently.

---

## Inter-Service Integration

### Keystone (Trust Delegation)

Heat uses Keystone **trusts** to perform deferred operations as the stack owner. When a stack is created, Heat stores a trust credential so that auto-scaling events, wait condition callbacks, and stack updates can be executed asynchronously on behalf of the original user — even after their session token has expired.

The `heat` service user must have the `heat_stack_owner` role assigned (or equivalent policy):

```bash
openstack role add --user heat --project service heat_stack_owner
```

### Nova, Neutron, Cinder

Heat calls these services' standard REST APIs directly from `heat-engine`. The credentials come from the trust. Heat does not use RPC for cross-service calls; it uses HTTP clients (`heatclient`, `novaclient`, `neutronclient`, etc.) wrapped in a consistent retry/backoff layer.

### Aodh (Alarming) — Auto-Scaling

`OS::Heat::ScalingPolicy` generates a signed webhook URL. An Aodh alarm is pointed at that URL. When the alarm fires, Aodh calls the URL, which triggers Heat to execute the scaling policy (add or remove instances from the `OS::Heat::AutoScalingGroup`).
