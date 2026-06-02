# Magnum Architecture

## Magnum Services

Magnum is composed of two processes that communicate over an oslo.messaging bus (typically RabbitMQ).

```
  Client (CLI / Horizon / CI pipeline)
              │
       ┌──────┴──────┐
       │ magnum-api  │  ← REST API (port 9511)
       │  (WSGI)     │
       └──────┬──────┘
              │ oslo.messaging RPC
       ┌──────┴────────┐
       │ magnum-       │  ← Cluster orchestration
       │ conductor     │  (template generation, Heat calls,
       └──────┬────────┘   certificate management)
              │
    ┌─────────┼──────────────────┐
    │         │                  │
 Heat API  Keystone API    Barbican API
 (stacks)  (trusts)        (certs)
```

### magnum-api

The HTTP REST API server. Accepts requests from clients, validates tokens with Keystone, enforces oslo.policy, and dispatches work to `magnum-conductor` over RPC.

- Exposes the Magnum API v1 on port **9511**
- Handles CRUD for cluster templates, clusters, node groups, and certificates
- Runs as a WSGI application (Apache mod_wsgi or uWSGI)
- Performs input validation and policy checks before forwarding to the conductor

**Endpoint registration:**
```bash
openstack service create --name magnum --description "Container Infra" container-infra
openstack endpoint create --region RegionOne container-infra public   http://10.0.0.10:9511/v1
openstack endpoint create --region RegionOne container-infra internal http://10.0.0.11:9511/v1
openstack endpoint create --region RegionOne container-infra admin    http://10.0.0.11:9511/v1
```

### magnum-conductor

The orchestration engine. Receives RPC calls from `magnum-api` and manages the full cluster lifecycle:

- Generates HOT templates for the requested cluster type and driver
- Calls Heat to create, update, or delete the cluster's infrastructure stack
- Monitors Heat stack progress and propagates status back to the Magnum database
- Manages cluster CA and TLS certificate issuance (via Barbican or local storage)
- Handles node group scale-out/scale-in by updating the Heat stack
- Creates and manages Keystone trusts for deferred operations

Multiple conductor workers can run simultaneously. They use database row-locking to avoid duplicating work on the same cluster.

**Summary table:**

| Service | Default Port | Notes |
|---|---|---|
| magnum-api | 9511 | REST API for all Magnum operations |
| magnum-conductor | — (RPC only) | Multiple workers; horizontally scalable |

---

## Cluster Templates

A **cluster template** (historically called "bay model") defines the reusable configuration for a class of clusters. It specifies the OS image, flavors, network parameters, COE type, and optional feature flags. Cluster templates are project-scoped but can be marked `public` to be shared across projects.

### Key Cluster Template Parameters

| Parameter | Description | Example |
|---|---|---|
| `--coe` | Container Orchestration Engine | `kubernetes` |
| `--image` | Glance image for cluster nodes | `fedora-coreos-40` |
| `--keypair` | SSH key pair for node access | `mykey` |
| `--external-network` | Neutron external network for floating IPs | `public` |
| `--dns-nameserver` | DNS server for cluster nodes | `8.8.8.8` |
| `--master-flavor` | Nova flavor for master nodes | `m1.large` |
| `--flavor` | Nova flavor for worker nodes | `m1.medium` |
| `--network-driver` | Pod networking driver | `flannel`, `calico` |
| `--volume-driver` | Storage driver | `cinder` |
| `--docker-storage-driver` | Container storage driver | `overlay2` |
| `--master-lb-enabled` | Place masters behind an Octavia LB | `true` |
| `--floating-ip-enabled` | Assign floating IPs to nodes | `true` |
| `--fixed-network` | Existing Neutron network to use | `cluster-net` |
| `--fixed-subnet` | Existing Neutron subnet to use | `cluster-subnet` |
| `--server-type` | `vm` (default) or `bm` (bare metal) | `vm` |
| `--http-proxy` / `--https-proxy` | Proxy for cluster node outbound traffic | `http://proxy.example.com:3128` |
| `--no-proxy` | Comma-separated no-proxy list | `10.0.0.0/8,localhost` |
| `--labels` | Key=value driver-specific options | (see below) |
| `--public` | Expose template to all projects | flag |
| `--hidden` | Hide template from non-admin list | flag |

### Important Labels

Labels are freeform key=value pairs passed to the cluster driver. Common ones for Kubernetes:

| Label | Description | Example |
|---|---|---|
| `kube_tag` | Kubernetes version (e.g., Hyperkube image tag) | `v1.30.1` |
| `container_runtime` | Container runtime | `containerd` |
| `etcd_tag` | etcd version | `3.5.12` |
| `coredns_tag` | CoreDNS version | `1.11.1` |
| `calico_tag` | Calico version (when network-driver=calico) | `v3.27.3` |
| `flannel_tag` | Flannel version | `v0.24.0` |
| `cloud_provider_enabled` | Enable OpenStack cloud provider | `true` |
| `cloud_provider_tag` | Cloud provider version | `v1.30.0` |
| `cinder_csi_enabled` | Enable Cinder CSI plugin | `true` |
| `cinder_csi_plugin_tag` | Cinder CSI plugin version | `v1.30.0` |
| `autoscaler_tag` | Cluster Autoscaler version | `v1.30.0` |
| `auto_scaling_enabled` | Enable Cluster Autoscaler | `true` |
| `auto_healing_enabled` | Enable auto-healing (node health monitoring) | `true` |
| `auto_healing_controller` | Auto-healing implementation | `draino` |
| `master_lb_floating_ip_enabled` | Assign floating IP to master LB | `true` |
| `ingress_controller` | Install an ingress controller | `nginx` |
| `metrics_server_enabled` | Install metrics-server | `true` |
| `cert_manager_api` | Enable Magnum cert-manager API | `true` |

---

## Clusters

A **cluster** is an instantiation of a cluster template. When a cluster is created, Magnum:

1. Creates a Keystone trust scoped to the requesting user's project
2. Generates a complete HOT template (or a set of Heat environment files) for the cluster
3. Calls Heat to create a stack with the generated template
4. Monitors the Heat stack until it reaches `CREATE_COMPLETE` or `CREATE_FAILED`
5. Retrieves cluster endpoint and certificate information from the Heat stack outputs
6. Stores the cluster record (status, API endpoint, CA certificate) in the Magnum database

### Cluster Fields

| Field | Description |
|---|---|
| `name` | Human-readable cluster name |
| `cluster_template_id` | Template this cluster is based on |
| `master_count` | Number of Kubernetes master nodes |
| `node_count` | Number of initial worker nodes |
| `keypair` | SSH key pair (overrides template setting) |
| `master_flavor_id` | Master flavor (overrides template) |
| `flavor_id` | Worker flavor (overrides template) |
| `labels` | Per-cluster label overrides |
| `create_timeout` | Max minutes to wait for creation |
| `discovery_url` | etcd discovery endpoint (auto-generated if not set) |
| `stack_id` | The Heat stack UUID for this cluster |
| `api_address` | Kubernetes API server URL |
| `coe_version` | Kubernetes version running on the cluster |
| `container_version` | Container runtime version |
| `status` | Current cluster status (see state machine) |
| `status_reason` | Human-readable reason for current status |

---

## Node Groups

A **node group** is a pool of worker nodes within a cluster that share the same flavor, image, and configuration. Every cluster has at least two default node groups:

- `default-master` — the master/control-plane nodes
- `default-worker` — the initial worker nodes

Additional custom node groups can be added to a cluster after creation (e.g., a GPU node group, a high-memory node group). Each node group is implemented as a resource group within the cluster's Heat stack.

### Node Group Fields

| Field | Description |
|---|---|
| `name` | Node group name |
| `cluster_id` | Parent cluster |
| `node_count` | Current number of nodes in this group |
| `min_node_count` | Minimum for auto-scaling |
| `max_node_count` | Maximum for auto-scaling |
| `flavor_id` | Nova flavor for nodes in this group |
| `image_id` | Glance image override |
| `labels` | Node group-level label overrides |
| `role` | `master` or `worker` |
| `is_default` | True for the two default groups |
| `status` | Current node group status |

---

## COE Support

### Kubernetes (Primary, Supported)

Kubernetes is the only actively maintained COE in modern Magnum releases. The Kubernetes driver:

- Generates Heat templates that provision Fedora CoreOS (recommended), Ubuntu, or other cloud-init-compatible images
- Configures kubeadm or the Magnum-specific Kubernetes boot scripts
- Sets up the Kubernetes API server, controller manager, scheduler, and etcd on master nodes
- Joins worker nodes to the cluster using a bootstrap token
- Installs the OpenStack cloud provider (`cloud-controller-manager`) for Node lifecycle integration
- Installs the Cinder CSI plugin for persistent volume support
- Optionally installs Calico or Flannel for pod networking
- Optionally installs the Kubernetes Cluster Autoscaler

**Supported Kubernetes versions**: Magnum 2025.1 supports Kubernetes 1.28, 1.29, and 1.30. Specific version support depends on the image and `kube_tag` label.

### Docker Swarm (Deprecated, Removed)

Docker Swarm support was removed in the 2024.1 (Caracal) release.

### Mesos / DC/OS (Deprecated, Removed)

Mesos support was removed in the 2023.1 (Antelope) release.

---

## Heat Stack Integration

Magnum does not maintain its own infrastructure management layer — it delegates entirely to Heat. The relationship is:

```
Cluster (Magnum DB record)
    └── Heat Stack (created by Magnum, owned by heat service user)
              ├── OS::Heat::Stack (master node group)
              │         ├── OS::Nova::Server × N (master VMs)
              │         ├── OS::Cinder::Volume × N (etcd volumes)
              │         └── OS::Neutron::Port × N
              ├── OS::Heat::Stack (default worker node group)
              │         ├── OS::Heat::AutoScalingGroup (or ResourceGroup)
              │         │         └── OS::Nova::Server × N (worker VMs)
              │         └── OS::Neutron::Port × N
              ├── OS::Neutron::Net (cluster private network)
              ├── OS::Neutron::Subnet
              ├── OS::Neutron::Router
              ├── OS::Neutron::RouterInterface
              ├── OS::Octavia::LoadBalancer (master API LB, optional)
              └── OS::Neutron::FloatingIP (master LB or first master)
```

When a cluster is created, `magnum-conductor`:

1. Calls `driver.create_cluster(cluster)` on the appropriate cluster driver
2. The driver renders a Jinja2 template (from `magnum/drivers/<driver>/templates/`) into a complete HOT template plus environment files
3. `magnum-conductor` calls `heat_client.stacks.create(...)` with the rendered template
4. A periodic task in `magnum-conductor` polls the Heat stack status and syncs it back to the Magnum cluster record

The Heat stack is owned by the Magnum service user (via trust). Operators can inspect it with `openstack stack show <stack-id>` and drill into resource events with `openstack stack event list --nested-depth 3 <stack-id>`.

---

## Cluster Drivers

A cluster driver is a Python class that implements the driver interface for a specific COE and OS combination. Drivers live in `magnum/drivers/`.

**Current Kubernetes drivers:**

| Driver Directory | Description |
|---|---|
| `k8s_fedora_coreos_v1` | Kubernetes on Fedora CoreOS (recommended) |
| `k8s_fedora_atomic_v1` | Kubernetes on Fedora Atomic Host (legacy) |
| `k8s_ubuntu_v1` | Kubernetes on Ubuntu |

Each driver directory contains:
- `driver.py` — implements the `Driver` interface
- `templates/` — Jinja2 HOT template fragments (cluster topology, master group, worker group, network, etc.)
- `image/` — scripts and cloud-init fragments installed into the cluster nodes

The driver interface (key methods):

```python
class Driver:
    def create_cluster(self, context, cluster, create_timeout): ...
    def update_cluster(self, context, cluster, scale_manager, rollback): ...
    def delete_cluster(self, context, cluster): ...
    def upgrade_cluster(self, context, cluster, cluster_template, max_batch_size, nodegroup): ...
    def create_nodegroup(self, context, cluster, nodegroup): ...
    def update_nodegroup(self, context, cluster, nodegroup): ...
    def delete_nodegroup(self, context, cluster, nodegroup): ...
    def get_monitor(self, context, cluster): ...
    def get_scale_manager(self, context, db_cluster, cluster): ...
    def get_driver_type(self): ...
```
