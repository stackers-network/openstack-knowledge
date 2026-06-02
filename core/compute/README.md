# Compute — Nova + Placement

Nova is the OpenStack Compute service. It provisions, schedules, and manages the lifecycle of virtual machine instances (and bare-metal nodes via Ironic). Placement is the resource-tracking service that Nova (and other services) use to claim compute, memory, disk, and custom resources before scheduling decisions are made.

Nova has been a core OpenStack service since the Austin release (2010). Placement was extracted from Nova's codebase and became an independent service in the Stein release (2019).

## When to Read This Skill

- Deploying Nova and Placement for the first time
- Launching, managing, or troubleshooting instances
- Tuning the scheduler (filters, weighers, aggregates)
- Understanding cells v2 architecture and API/cell database layout
- Diagnosing scheduling failures or "No valid host" errors
- Performing or troubleshooting live migrations
- Managing resource inventories, traits, and allocation candidates via Placement
- Resizing, evacuating, shelving, or rebuilding instances
- Configuring quota limits for projects and users
- Adding or replacing compute hosts

## Sub-Files

| File | What It Covers |
|---|---|
| [architecture.md](architecture.md) | Nova services, cells v2 layout, scheduling pipeline, hypervisor support, inter-service integration (Neutron, Cinder, Glance, Placement) |
| [operations.md](operations.md) | CLI and API: server lifecycle, flavors, keypairs, migrations, quotas, console, backup, full config reference |
| [internals.md](internals.md) | Scheduler filters and weighers, cells v2 DB layout, conductor as DB proxy, RPC versioning, instance state machine, compute manager hooks |
| [placement.md](placement.md) | Placement service: resource providers, resource classes, inventories, allocations, traits, allocation candidates, nested providers, reshaper |
| [live-migration.md](live-migration.md) | Live migration types, prerequisites, pre-copy vs post-copy, monitoring, force-complete, troubleshooting, config reference |

## Quick Reference

```bash
# Launch an instance
openstack server create \
  --flavor m1.small \
  --image ubuntu-24.04 \
  --network private-net \
  --key-name mykey \
  my-instance

# List all instances (all projects, admin)
openstack server list --all-projects

# Show instance details
openstack server show my-instance

# Stop and start an instance
openstack server stop my-instance
openstack server start my-instance

# Reboot (soft = ACPI signal; hard = power cycle)
openstack server reboot --soft my-instance
openstack server reboot --hard my-instance

# Delete an instance
openstack server delete my-instance

# List hypervisors and their resource usage
openstack hypervisor list --long

# Check placement resource providers
openstack resource provider list

# Show allocations for an instance
openstack resource provider allocation show <instance-uuid>
```

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| Keystone | Authentication and service catalog | Every Nova API call is validated against Keystone |
| Glance | Image service | Nova downloads images to compute nodes on first boot |
| Neutron | Network ports and virtual NICs | Nova calls Neutron to create ports before spawning |
| Placement | Resource tracking and scheduling claims | Queried by nova-scheduler before choosing a host |
| Cinder | Block storage volumes | Optional; required for volume-backed instances and volume attach |
| libvirt / KVM | Default hypervisor | Most production deployments use KVM via libvirt |
| RabbitMQ (or oslo.messaging backend) | Async RPC between Nova services | Required; all inter-service calls go through the message bus |
| MariaDB 10.6+ or PostgreSQL 14+ | Nova API DB + per-cell DBs | API DB and at least one cell DB required |
| Memcached | Cache for scheduler and API | Strongly recommended; reduces DB load |

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

Nova release notes: https://docs.openstack.org/releasenotes/nova/
Placement release notes: https://docs.openstack.org/releasenotes/placement/
