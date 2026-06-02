# Orchestration — Heat

Heat is the OpenStack Orchestration service. It provisions and manages stacks of cloud resources defined in declarative templates. A single Heat Orchestration Template (HOT) can describe compute instances, networks, subnets, routers, floating IPs, security groups, volumes, load balancers, and auto-scaling groups — along with all the dependency relationships between them. Heat manages the full lifecycle: create, update, suspend, resume, check, and delete.

Heat has been a core OpenStack service since the Icehouse release (2014). The convergence engine (parallel, graph-based resource operations) replaced the legacy sequential stack engine in Newton (2016).

## When to Read This Skill

- Writing HOT templates to provision multi-tier application infrastructure
- Understanding how Heat resolves resource dependency graphs
- Creating or managing environments (parameter defaults, resource type mappings)
- Debugging stack failures and inspecting resource events
- Implementing auto-scaling with `OS::Heat::AutoScalingGroup` and `OS::Heat::ScalingPolicy`
- Delivering software configuration to instances via `OS::Heat::SoftwareConfig` and `OS::Heat::SoftwareDeployment`
- Implementing wait conditions and resource signaling
- Understanding the convergence engine vs legacy stack lifecycle
- Configuring `heat.conf` for production deployments
- Writing custom resource plugins

## Sub-Files

| File | What It Covers |
|---|---|
| [architecture.md](architecture.md) | heat-api, heat-api-cfn, heat-engine; HOT template format and sections; resource types; environments; convergence architecture; inter-service integration |
| [operations.md](operations.md) | CLI and API: stack create/update/delete/check/suspend/resume, resource inspection, event querying, output retrieval, template validation; complete HOT template example; environment files; config reference |
| [internals.md](internals.md) | Resource plugin interface; built-in resource types; dependency graph resolution; convergence engine; stack lock mechanism; nested stacks; resource signaling; software deployment chain |

## Quick Reference

```bash
# Validate a template before deploying
openstack orchestration template validate -t web-app.yaml

# Create a stack
openstack stack create \
  -t web-app.yaml \
  -e prod-env.yaml \
  --parameter db_password=secret123 \
  web-app-stack

# Watch stack progress
openstack stack event list --follow web-app-stack

# List all stacks
openstack stack list

# Show stack details and outputs
openstack stack show web-app-stack
openstack stack output show web-app-stack floating_ip

# Update a stack in place
openstack stack update \
  -t web-app-v2.yaml \
  -e prod-env.yaml \
  web-app-stack

# Check real-world resource status vs Heat's recorded state
openstack stack check web-app-stack

# Suspend all stack resources
openstack stack suspend web-app-stack

# Resume a suspended stack
openstack stack resume web-app-stack

# Delete a stack and all its resources
openstack stack delete web-app-stack

# List resources in a stack
openstack stack resource list web-app-stack

# Inspect a specific resource
openstack stack resource show web-app-stack web_server

# List nested stack events
openstack stack event list --nested-depth 2 web-app-stack

# Show the template stored for a running stack
openstack stack template show web-app-stack
```

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| Keystone | Authentication, service catalog, and trust delegation | Heat creates Keystone trusts so that deferred operations (e.g. auto-scaling) can act as the stack owner |
| Nova | Provisioning `OS::Nova::Server` resources | Most stacks include at least one server |
| Neutron | Provisioning `OS::Neutron::*` resources (networks, subnets, routers, floating IPs, security groups) | Required for any stack with networking resources |
| Cinder | Provisioning `OS::Cinder::Volume` and `OS::Cinder::VolumeAttachment` | Required for stacks with persistent block storage |
| Glance | Resolving image names/IDs for Nova servers | Used indirectly through Nova |
| Octavia | Provisioning `OS::Octavia::LoadBalancer` resources | Optional; required for LBaaS stacks |
| RabbitMQ (or oslo.messaging backend) | RPC between heat-api and heat-engine | Required; all engine calls go through the message bus |
| MariaDB 10.6+ or PostgreSQL 14+ | Heat database (stacks, resources, events, software configs) | Required |
| Memcached | Token and policy caching | Strongly recommended |
| `python-heatclient` / `python-openstackclient` | CLI and Python bindings | `openstack stack *` commands use the Heat v1 REST API |

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

Heat release notes: https://docs.openstack.org/releasenotes/heat/
