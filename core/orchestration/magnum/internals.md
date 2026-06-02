# Magnum Internals

## Cluster Driver Interface

Every cluster driver inherits from `magnum.drivers.common.driver.Driver` and implements a set of lifecycle methods. Drivers live under `magnum/drivers/<driver_name>/` and are registered via Python entry points.

### Core Driver Methods

```python
class Driver:

    @property
    def provides(self):
        """
        Return a list of dicts describing what this driver provides:
          [{'server_type': 'vm', 'os': 'fedora-coreos', 'coe': 'kubernetes'}]
        """
        raise NotImplementedError

    def create_cluster(self, context, cluster, create_timeout):
        """
        Provision a new cluster. Called by magnum-conductor.

        Typical implementation:
        1. Generate Heat template parameters from cluster + cluster_template
        2. Call heat_client.stacks.create(stack_name, template, params, env_files)
        3. Store the Heat stack_id in cluster.stack_id
        4. Update cluster.status = 'CREATE_IN_PROGRESS'
        """

    def update_cluster(self, context, cluster, scale_manager=None, rollback=False):
        """
        Update an existing cluster (scale, parameter change).
        Calls heat_client.stacks.update() with new parameters.
        """

    def delete_cluster(self, context, cluster):
        """
        Delete the cluster's Heat stack and all its resources.
        Calls heat_client.stacks.delete(cluster.stack_id).
        """

    def upgrade_cluster(self, context, cluster, cluster_template,
                        max_batch_size, nodegroup):
        """
        Rolling upgrade to a new cluster template (new Kubernetes version).
        Replaces node images in batches: each batch updates the Heat stack
        and waits for the nodes to rejoin before proceeding.
        """

    def create_nodegroup(self, context, cluster, nodegroup):
        """
        Add a new node group to an existing cluster.
        Updates the cluster's Heat stack to add a new ResourceGroup.
        """

    def update_nodegroup(self, context, cluster, nodegroup):
        """
        Update node group properties (count, min/max for auto-scaling).
        Updates the Heat stack parameter for this node group.
        """

    def delete_nodegroup(self, context, cluster, nodegroup):
        """
        Remove a node group from the cluster.
        Updates the Heat stack to remove the node group's ResourceGroup.
        """

    def get_monitor(self, context, cluster):
        """
        Return a monitor object that checks cluster health.
        Used by the periodic health check task.
        """

    def get_scale_manager(self, context, db_cluster, cluster):
        """
        Return a scale manager that determines which nodes to remove
        when scaling down (e.g., prefers removing unschedulable nodes).
        """

    def get_driver_type(self):
        """Return a string identifier, e.g. 'kubernetes'."""
```

### Driver Directory Layout

```
magnum/drivers/k8s_fedora_coreos_v1/
├── driver.py               # Driver class implementation
├── template_def.py         # Maps cluster fields → Heat parameters
├── monitor.py              # Cluster health monitor
├── scale_manager.py        # Node selection for scale-down
└── templates/
    ├── cluster.yaml                  # Top-level Heat template
    ├── kubecluster.yaml              # Master + worker nested stack
    ├── kubemaster.yaml               # Master node group template
    ├── kubeminion.yaml               # Worker node group template
    ├── kubecluster_network.yaml      # Networking resources
    ├── lb_etcd.yaml                  # etcd load balancer
    ├── nodegroup.yaml                # Generic node group template
    └── fragments/
        ├── configure-kubernetes-master.sh
        ├── configure-kubernetes-minion.sh
        ├── install-coredns.sh
        ├── install-calico.sh
        ├── install-cinder-csi.sh
        ├── install-cloud-controller-manager.sh
        └── configure-etcd.sh
```

### Template Definition

`template_def.py` maps Magnum cluster and cluster-template fields to Heat template parameters:

```python
class K8sFedoraCoreOSTemplateDefinition(BaseTemplateDefinition):

    def get_params(self, context, cluster_template, cluster, **kwargs):
        extra_params = kwargs.pop('extra_params', {})

        extra_params['number_of_masters'] = cluster.master_count
        extra_params['number_of_minions'] = cluster.node_count
        extra_params['master_flavor'] = cluster.master_flavor_id
        extra_params['minion_flavor'] = cluster.flavor_id
        extra_params['server_image'] = cluster_template.image_id
        extra_params['ssh_key_name'] = cluster_template.keypair_id
        extra_params['external_network'] = cluster_template.external_network_id
        extra_params['dns_nameserver'] = cluster_template.dns_nameserver
        extra_params['network_driver'] = cluster_template.network_driver
        extra_params['kube_tag'] = cluster_template.labels.get('kube_tag')
        extra_params['etcd_tag'] = cluster_template.labels.get('etcd_tag')
        extra_params['container_runtime'] = cluster_template.labels.get(
            'container_runtime', 'containerd')
        extra_params['cloud_provider_enabled'] = cluster_template.labels.get(
            'cloud_provider_enabled', 'false')
        # ... many more label → parameter mappings

        return super().get_params(
            context, cluster_template, cluster,
            extra_params=extra_params, **kwargs)
```

---

## Heat Template Generation for Kubernetes Clusters

When `create_cluster` is called, the conductor:

1. Instantiates the driver via the driver registry
2. Calls `driver.get_template_definition()` to get a `TemplateDef` object
3. Calls `template_def.get_params(context, cluster_template, cluster)` to build a flat dict of Heat parameters
4. Calls `template_def.get_env_files(cluster_template, cluster)` to get environment file paths
5. Calls `heat_client.stacks.create()`:
   ```python
   heat_client.stacks.create(
       stack_name=cluster.name,
       template=rendered_template_yaml,
       environment=env_dict,
       parameters=params_dict,
       timeout_mins=create_timeout,
   )
   ```
6. Stores the returned `stack.id` on the cluster record

### Generated Heat Stack Structure

For a 3-master, 5-worker Kubernetes cluster on Fedora CoreOS, the Heat stack looks like:

```
<cluster-name>                   ← OS::Heat::Stack (top level, created by conductor)
├── network                      ← OS::Heat::Stack (networking)
│   ├── private_network          ← OS::Neutron::Net
│   ├── private_subnet           ← OS::Neutron::Subnet
│   ├── router                   ← OS::Neutron::Router
│   └── router_interface         ← OS::Neutron::RouterInterface
├── api_lb                       ← OS::Heat::Stack (master LB, if enabled)
│   ├── loadbalancer             ← OS::Octavia::LoadBalancer
│   ├── listener                 ← OS::Octavia::Listener
│   ├── pool                     ← OS::Octavia::Pool
│   └── floating_ip              ← OS::Neutron::FloatingIP
├── kube_masters                 ← OS::Heat::Stack (master node group)
│   ├── master_0                 ← OS::Nova::Server
│   ├── master_1                 ← OS::Nova::Server
│   ├── master_2                 ← OS::Nova::Server
│   ├── etcd_volume_0            ← OS::Cinder::Volume
│   ├── etcd_volume_1            ← OS::Cinder::Volume
│   ├── etcd_volume_2            ← OS::Cinder::Volume
│   └── pool_members             ← OS::Octavia::PoolMember × 3
└── kube_minions                 ← OS::Heat::Stack (default worker group)
    └── minion_group             ← OS::Heat::ResourceGroup (or AutoScalingGroup)
        ├── minion_0             ← OS::Nova::Server
        ├── minion_1             ← OS::Nova::Server
        ├── minion_2             ← OS::Nova::Server
        ├── minion_3             ← OS::Nova::Server
        └── minion_4             ← OS::Nova::Server
```

Additional node groups each add a new `OS::Heat::Stack` child to the top-level stack.

---

## Node Group Lifecycle

### Adding a Node Group

When `create_nodegroup` is called:

1. Magnum renders a new `nodegroup.yaml` template fragment with the node group parameters
2. The conductor calls `heat_client.stacks.update()` on the cluster's existing stack, passing the new resource in the environment
3. Heat processes the stack update: creates the new `OS::Heat::Stack` for the node group and provisions the `OS::Nova::Server` instances
4. Once the Heat stack reaches `UPDATE_COMPLETE`, the conductor sets the node group status to `CREATE_COMPLETE`
5. The new Kubernetes nodes register with the API server (via bootstrap token or cloud-init kubeadm join command)

### Scaling a Node Group

Scale-out (increase `node_count`):

1. `update_nodegroup` is called with the new `node_count`
2. Conductor calls `heat_client.stacks.update()` with the new count parameter
3. Heat adds the additional `OS::Nova::Server` instances to the `ResourceGroup`
4. New VMs boot and run kubeadm join (or cloud-init join script)
5. Nodes appear in `kubectl get nodes` as `Ready`

Scale-in (decrease `node_count`):

1. `scale_manager.get_nodes_for_removal(cluster, delta)` selects which nodes to remove:
   - Prefers nodes already marked `NotReady` or `SchedulingDisabled` in Kubernetes
   - Prefers nodes with lowest pod count (to minimize disruption)
   - Returns a list of Nova server UUIDs
2. Conductor passes the list to Heat as `removal_policies` on the `ResourceGroup`
3. Heat deletes those specific VMs and updates the `ResourceGroup` count
4. Kubernetes detects the nodes are gone and marks them `NotReady` then removes them from the node list

---

## Cluster State Machine

```
                        ┌─────────────────────┐
                        │  (not yet created)  │
                        └──────────┬──────────┘
                                   │ cluster create
                                   ▼
                        ┌──────────────────────┐
                        │  CREATE_IN_PROGRESS  │
                        └──────────┬───────────┘
               success  │          │ failure
                    ┌───┘          └────────────┐
                    ▼                           ▼
         ┌──────────────────┐       ┌─────────────────┐
         │  CREATE_COMPLETE │       │   CREATE_FAILED  │
         └────────┬─────────┘       └─────────────────┘
                  │
         ┌────────┴────────────────────────────┐
         │                                     │
         │ cluster update /            cluster delete
         │ nodegroup op                        │
         ▼                                     ▼
┌──────────────────────┐           ┌──────────────────────┐
│  UPDATE_IN_PROGRESS  │           │  DELETE_IN_PROGRESS  │
└──────────┬───────────┘           └──────────┬───────────┘
 success   │   failure              success   │   failure
    ┌──────┘   └────────┐              ┌──────┘   └─────────┐
    ▼                   ▼              ▼                    ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│UPDATE_COMPLETE│  │UPDATE_FAILED │  │DELETE_COMPLETE│  │DELETE_FAILED │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

The cluster status is derived by polling the Heat stack status. The conductor has a periodic task (`_sync_cluster_status`) that runs every `periodic_interval` seconds:

```python
def _sync_cluster_status(self):
    for cluster in db.cluster_list_by_status('CREATE_IN_PROGRESS'):
        stack = heat_client.stacks.get(cluster.stack_id)
        if stack.stack_status == 'CREATE_COMPLETE':
            cluster.status = 'CREATE_COMPLETE'
            cluster.api_address = _extract_api_address(stack.outputs)
        elif stack.stack_status == 'CREATE_FAILED':
            cluster.status = 'CREATE_FAILED'
            cluster.status_reason = stack.stack_status_reason
        db.cluster_update(cluster)
```

---

## Certificate Management

Every Kubernetes cluster provisioned by Magnum has its own **cluster CA** (Certificate Authority). This CA signs:

- The Kubernetes API server's TLS certificate
- etcd peer and client certificates
- The `admin` client certificate (returned in kubeconfig)
- Kubelet client certificates for each node

### Certificate Storage Backends

Magnum supports two certificate storage backends, configured with `cert_manager_type`:

**`local`** (default for development):
- CA and certificates stored in the Magnum database
- Simple but the private key is accessible to the Magnum service user

**`barbican`** (recommended for production):
- CA private key stored in Barbican (OpenStack Key Manager)
- Each cluster's CA key is a separate Barbican secret
- Magnum service user needs the `creator` role on Barbican
- Configure with: `cert_manager_type = barbican`

### Certificate Issuance Flow (Cluster Creation)

```
magnum-conductor
  1. Generate cluster CA key pair (RSA 4096 or ECDSA P-384)
  2. Self-sign the CA certificate (10-year validity)
  3. Store CA cert + private key in Barbican (or local DB)

  4. Generate API server key pair
  5. Create CSR with SANs: cluster API IP, cluster API FQDN,
     service cluster IP (10.96.0.1), 127.0.0.1, kubernetes.default.svc
  6. Sign with cluster CA → API server certificate

  7. Generate etcd key pair, sign with cluster CA
  8. Generate admin key pair, sign with cluster CA → admin kubeconfig

Heat template
  9. Inject CA cert, API server cert+key, etcd cert+key into Nova server
     user_data (base64-encoded, or via Heat SoftwareConfig)
     cloud-init writes them to /etc/kubernetes/pki/
```

### Kubeconfig Generation

When `openstack coe cluster config my-k8s` is called:

1. `magnum-api` retrieves the cluster's CA certificate and admin client certificate from the certificate manager
2. Assembles a kubeconfig YAML:
   ```yaml
   apiVersion: v1
   kind: Config
   clusters:
   - cluster:
       certificate-authority-data: <base64-ca-cert>
       server: https://203.0.113.10:6443
     name: my-k8s
   users:
   - name: admin
     user:
       client-certificate-data: <base64-admin-cert>
       client-key-data: <base64-admin-key>
   contexts:
   - context:
       cluster: my-k8s
       user: admin
     name: my-k8s
   current-context: my-k8s
   ```
3. Writes it to `./config` in the current directory (or `--dir` path)

### CA Rotation

`openstack coe ca rotate my-k8s` triggers:

1. Generate a new CA key pair
2. Re-sign all node certificates with the new CA
3. Restart kube-apiserver, kube-controller-manager, kubelet on all nodes with new certs
4. Update the Magnum database and certificate manager with the new CA

This is a disruptive operation: the Kubernetes API server is briefly unavailable during rotation.

---

## Auto-Healing

Auto-healing detects unhealthy nodes and replaces them without manual intervention.

### Implementation

When `auto_healing_enabled=true` is set via label, Magnum installs **Draino** and the **Cluster Autoscaler node problem detector** into the cluster during provisioning.

The auto-healing loop:

```
Node Problem Detector (DaemonSet on each node)
  │ detects: KernelDeadlock, ContainerRuntimeUnhealthy,
  │          ReadonlyFilesystem, NetworkUnavailable
  │
  ▼
Condition published to Node.status.conditions
  │
  ▼
Draino (Deployment, watches Node conditions)
  │ if node condition matches configured triggers:
  │   1. Cordons the node (prevents new pod scheduling)
  │   2. Drains the node (evicts existing pods)
  │   3. Annotates node as "draino/drain-completed"
  │
  ▼
Magnum periodic health check (magnum-conductor)
  │ calls heat_client: detect drained + NotReady nodes
  │ triggers stack update to replace the node
  │ Heat deletes the old server, creates a new one
  ▼
New node boots and joins the Kubernetes cluster
```

### Health Monitor

The `monitor.py` in each driver implements `get_nodes_not_healthy()` which:

1. Calls the Kubernetes API via `kubectl` or the Python kubernetes client
2. Returns a list of nodes that are `NotReady` or have problem conditions
3. `magnum-conductor` compares this list against a threshold; if the unhealthy count exceeds `auto_healing_max_count`, it triggers a stack update to replace them

---

## Auto-Scaling

### Kubernetes Cluster Autoscaler

When `auto_scaling_enabled=true`, Magnum installs the **Kubernetes Cluster Autoscaler** configured for the OpenStack provider. The autoscaler:

1. Monitors pod scheduling failures (pods stuck in `Pending` due to insufficient resources)
2. Determines which node groups can be expanded to schedule pending pods
3. Calls the Magnum API: `POST /v1/clusters/{id}/nodegroups/{ng_id}` with a new `node_count`
4. Magnum's `update_nodegroup` → Heat stack update → new VMs boot → nodes join

Similarly, on scale-in:

1. Autoscaler identifies nodes that have been underutilized for `scale-down-unneeded-time` (default 10 minutes)
2. Checks that all pods on the node can be rescheduled (respects PodDisruptionBudgets)
3. Calls Magnum API to reduce `node_count`
4. Magnum's scale manager selects which node to remove and passes it to Heat

### Autoscaler Configuration (via cluster template labels)

| Label | Description | Default |
|---|---|---|
| `auto_scaling_enabled` | Enable the autoscaler | `false` |
| `autoscaler_tag` | Autoscaler Docker image tag | matches `kube_tag` |
| `min_node_count` | Minimum nodes per node group | `1` |
| `max_node_count` | Maximum nodes per node group | `10` |
| `scale_down_enabled` | Allow autoscaler to scale down | `true` |
| `scale_down_unneeded_time` | Time a node must be underutilized before removal | `10m` |
| `scale_down_delay_after_add` | Wait after scale-out before considering scale-in | `10m` |
| `scale_down_utilization_threshold` | CPU/memory utilization threshold for "unneeded" | `0.5` |

### Interaction with Heat

The autoscaler uses the **Magnum API** rather than calling Heat directly. This is important: the autoscaler never has Heat credentials. Magnum's `update_nodegroup` handles all Heat interaction, meaning autoscaler-driven changes appear in the Magnum audit log, respect Magnum quotas, and are reflected in `openstack coe cluster show` output.

---

## Keystone Trust Integration

Magnum uses Keystone **trusts** so that long-running background operations (cluster updates, auto-scaling, health checks) can be performed as the cluster owner — even after the original user's session token has expired.

### Trust Creation Flow

1. When a cluster is created, the requesting user (trustor) creates a trust delegating a subset of roles to the Magnum service user (trustee) for the user's project:
   ```
   trustor: <cluster-owner-user-id>
   trustee: <magnum-service-user-id>
   roles: [member]
   project: <cluster-owner-project-id>
   ```
2. The trust is stored (encrypted) in the Magnum database linked to the cluster record
3. When `magnum-conductor` needs to call Heat or Kubernetes on behalf of the cluster owner, it exchanges the trust for a project-scoped token using:
   ```
   POST /v3/auth/tokens
   {"auth": {"identity": {"methods": ["token"], "token": {"id": "<service-token>"}},
             "scope": {"OS-TRUST:trust": {"id": "<trust-id>"}}}}
   ```
4. The resulting token is scoped to the cluster owner's project and used for all Heat API calls for that cluster

### Why Trusts Matter

Without trusts, the Magnum service user would need to be an admin (to operate on other users' projects) or users would need to provide long-lived passwords. Trusts enable least-privilege delegation: Magnum can only do what the cluster owner themselves could do, scoped to their project.

---

## Database Schema (Key Tables)

| Table | Purpose |
|---|---|
| `cluster_template` | Cluster template definitions (image, flavor, labels, etc.) |
| `cluster` | Cluster records (status, stack_id, api_address, cert references) |
| `nodegroup` | Node group records per cluster (count, min, max, flavor) |
| `x509keypair` | CA and client certificates (used by `local` cert manager) |
| `quota` | Per-project resource quotas (cluster count) |
| `federation` | Multi-region cluster federation records |
