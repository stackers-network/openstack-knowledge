# Orchestration — Magnum

Magnum is the OpenStack Container Orchestration Engine (COE) service. It provisions and manages container clusters — primarily Kubernetes — on top of OpenStack infrastructure. Magnum uses Heat to create the underlying Nova instances, Neutron networks, Cinder volumes, and load balancers, then installs and configures the chosen COE on top of them. The result is a fully functional Kubernetes cluster whose kubeconfig is returned to the user.

Magnum has been a core OpenStack service since the Liberty release (2015). Kubernetes is the primary supported COE; Docker Swarm and Mesos are deprecated and removed in 2024.x+ releases.

## When to Read This Skill

- Provisioning Kubernetes clusters on OpenStack with a single CLI command
- Creating and managing cluster templates (standard, reusable cluster configurations)
- Scaling cluster node groups up or down
- Upgrading clusters to a new Kubernetes version
- Retrieving kubeconfig credentials for a cluster
- Understanding how Magnum generates Heat stacks for cluster provisioning
- Configuring cluster auto-scaling and auto-healing
- Managing node groups (multi-node-group clusters with mixed flavors)
- Configuring `magnum.conf` for production deployments
- Troubleshooting cluster creation failures via Heat stack events

## Sub-Files

| File | What It Covers |
|---|---|
| [architecture.md](architecture.md) | magnum-api, magnum-conductor; cluster templates; clusters and node groups; COE support (Kubernetes focus); Heat stack integration; cluster drivers |
| [operations.md](operations.md) | CLI: cluster templates, clusters, node groups, resize, upgrade, kubeconfig; config file reference |
| [internals.md](internals.md) | Cluster driver interface; Heat template generation; node group lifecycle; cluster state machine; certificate management; auto-healing; auto-scaling |

## Quick Reference

```bash
# Create a cluster template
openstack coe cluster template create k8s-template \
  --image fedora-coreos-40 \
  --keypair mykey \
  --external-network public \
  --dns-nameserver 8.8.8.8 \
  --master-flavor m1.large \
  --flavor m1.medium \
  --network-driver flannel \
  --coe kubernetes

# Create a Kubernetes cluster
openstack coe cluster create my-k8s \
  --cluster-template k8s-template \
  --master-count 3 \
  --node-count 5

# Watch cluster status
openstack coe cluster show my-k8s

# Get kubeconfig
openstack coe cluster config my-k8s
export KUBECONFIG=$(pwd)/config
kubectl get nodes

# Resize worker nodes
openstack coe cluster resize my-k8s 10

# List node groups
openstack coe nodegroup list my-k8s

# Add a GPU node group
openstack coe nodegroup create my-k8s gpu-workers \
  --node-count 2 \
  --flavor m1.xlarge-gpu \
  --image fedora-coreos-40

# Delete a cluster
openstack coe cluster delete my-k8s
```

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| Keystone | Authentication, service catalog, trust delegation | Magnum creates Keystone trusts for cluster operations |
| Heat | Infrastructure provisioning via generated HOT templates | All cluster infrastructure is a Heat stack |
| Nova | Kubernetes master and worker node VMs | Heat provisions Nova servers for all nodes |
| Neutron | Cluster networking (tenant net, LB, floating IPs) | Per-cluster network with optional Octavia LB for the API server |
| Cinder | etcd volumes and persistent volume claims (via CSI) | Optional; recommended for etcd durability |
| Octavia | Load balancer for the Kubernetes API server | Optional; used when `master-lb-enabled=true` |
| Barbican | Cluster TLS certificate storage | Optional; used when `cert-manager-api=true` |
| Glance | Cluster node OS images (Fedora CoreOS, Ubuntu, etc.) | Images must have Kubernetes pre-installed or use cloud-init |
| RabbitMQ (or oslo.messaging backend) | RPC between magnum-api and magnum-conductor | Required |
| MariaDB 10.6+ or PostgreSQL 14+ | Magnum database (cluster templates, clusters, certificates) | Required |
| `python-magnumclient` / `python-openstackclient` | CLI | `openstack coe *` commands |

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

Magnum release notes: https://docs.openstack.org/releasenotes/magnum/
