# Magnum Operations

## Cluster Templates

### Create a Cluster Template

```bash
# Minimal Kubernetes template on Fedora CoreOS
openstack coe cluster template create k8s-template \
  --image fedora-coreos-40 \
  --keypair mykey \
  --external-network public \
  --dns-nameserver 8.8.8.8 \
  --master-flavor m1.large \
  --flavor m1.medium \
  --network-driver flannel \
  --coe kubernetes

# Production template with Calico networking, Cinder CSI, cloud provider,
# auto-scaling, load-balanced masters, and pinned Kubernetes version
openstack coe cluster template create k8s-prod-template \
  --image fedora-coreos-40 \
  --keypair prod-keypair \
  --external-network provider-net \
  --dns-nameserver 8.8.8.8 \
  --master-flavor m1.xlarge \
  --flavor m1.large \
  --network-driver calico \
  --volume-driver cinder \
  --docker-storage-driver overlay2 \
  --master-lb-enabled \
  --floating-ip-enabled \
  --coe kubernetes \
  --label kube_tag=v1.30.1 \
  --label container_runtime=containerd \
  --label etcd_tag=3.5.12 \
  --label coredns_tag=1.11.1 \
  --label calico_tag=v3.27.3 \
  --label cloud_provider_enabled=true \
  --label cloud_provider_tag=v1.30.0 \
  --label cinder_csi_enabled=true \
  --label cinder_csi_plugin_tag=v1.30.0 \
  --label auto_scaling_enabled=true \
  --label autoscaler_tag=v1.30.0 \
  --label auto_healing_enabled=true \
  --label master_lb_floating_ip_enabled=true \
  --label metrics_server_enabled=true

# Public template (visible to all projects)
openstack coe cluster template create k8s-shared \
  --image fedora-coreos-40 \
  --keypair infra-key \
  --external-network public \
  --dns-nameserver 8.8.8.8 \
  --master-flavor m1.large \
  --flavor m1.medium \
  --network-driver flannel \
  --coe kubernetes \
  --public

# Template using an existing network (no new network created per cluster)
openstack coe cluster template create k8s-existing-net \
  --image fedora-coreos-40 \
  --keypair mykey \
  --external-network public \
  --fixed-network cluster-network \
  --fixed-subnet cluster-subnet \
  --dns-nameserver 8.8.8.8 \
  --master-flavor m1.large \
  --flavor m1.medium \
  --network-driver calico \
  --no-floating-ip-enabled \
  --coe kubernetes
```

### List Cluster Templates

```bash
# List templates visible to the current project
openstack coe cluster template list

# List all templates (admin)
openstack coe cluster template list --detail

# Show column headers in output
openstack coe cluster template list -c name -c coe -c network_driver -c public
```

### Show a Cluster Template

```bash
openstack coe cluster template show k8s-template
openstack coe cluster template show k8s-template -f json
```

### Update a Cluster Template

```bash
# Set a label on an existing template
openstack coe cluster template update k8s-template replace labels/kube_tag=v1.30.2

# Change the flavor
openstack coe cluster template update k8s-template replace flavor_id=m1.large

# Add the public flag
openstack coe cluster template update k8s-template replace public=true

# Remove a label
openstack coe cluster template update k8s-template remove labels/old_label
```

### Delete a Cluster Template

```bash
openstack coe cluster template delete k8s-template

# Delete multiple templates
openstack coe cluster template delete k8s-template k8s-dev-template
```

---

## Clusters

### Create a Cluster

```bash
# Minimal cluster (1 master, 1 worker)
openstack coe cluster create my-k8s \
  --cluster-template k8s-template

# HA control plane with multiple workers
openstack coe cluster create my-k8s \
  --cluster-template k8s-template \
  --master-count 3 \
  --node-count 5

# Override template flavor and keypair
openstack coe cluster create my-k8s \
  --cluster-template k8s-template \
  --master-count 3 \
  --node-count 5 \
  --master-flavor m1.2xlarge \
  --flavor m1.xlarge \
  --keypair alternate-key

# Override labels at cluster level
openstack coe cluster create my-k8s \
  --cluster-template k8s-template \
  --master-count 3 \
  --node-count 5 \
  --label auto_scaling_enabled=true \
  --label min_node_count=3 \
  --label max_node_count=20

# Set creation timeout (minutes; default comes from template)
openstack coe cluster create my-k8s \
  --cluster-template k8s-template \
  --master-count 3 \
  --node-count 5 \
  --create-timeout 60

# Merge labels from template and cluster level
openstack coe cluster create my-k8s \
  --cluster-template k8s-template \
  --master-count 3 \
  --node-count 5 \
  --merge-labels \
  --label ingress_controller=nginx
```

### List Clusters

```bash
openstack coe cluster list

# Show detailed output
openstack coe cluster list --detail

# Show specific columns
openstack coe cluster list -c name -c status -c master_count -c node_count -c api_address
```

**Cluster status values:**

| Status | Meaning |
|---|---|
| `CREATE_IN_PROGRESS` | Heat stack being built |
| `CREATE_COMPLETE` | Cluster is ready |
| `CREATE_FAILED` | Cluster creation failed |
| `UPDATE_IN_PROGRESS` | Scaling or update in progress |
| `UPDATE_COMPLETE` | Update finished successfully |
| `UPDATE_FAILED` | Update failed |
| `DELETE_IN_PROGRESS` | Heat stack being deleted |
| `DELETE_COMPLETE` | Cluster fully deleted |
| `DELETE_FAILED` | Deletion failed (resources may remain) |
| `RESUME_COMPLETE` | Cluster resumed after suspend |
| `RESTORE_COMPLETE` | Cluster restored from snapshot |

### Show a Cluster

```bash
openstack coe cluster show my-k8s

# Show in JSON (useful for scripting)
openstack coe cluster show my-k8s -f json

# Get the Heat stack ID for a cluster
openstack coe cluster show my-k8s -c stack_id -f value
```

### Get Cluster Credentials (kubeconfig)

```bash
# Write kubeconfig to ./config in the current directory
openstack coe cluster config my-k8s

# Write to a specific path
openstack coe cluster config my-k8s --dir /home/user/.kube/

# Force overwrite existing file
openstack coe cluster config my-k8s --force

# Set KUBECONFIG and use kubectl immediately
eval $(openstack coe cluster config my-k8s)
kubectl get nodes
kubectl get namespaces
kubectl cluster-info
```

### Resize a Cluster (Scale Workers)

```bash
# Scale the default worker node group to 10 nodes
openstack coe cluster resize my-k8s 10

# Scale a specific node group
openstack coe cluster resize my-k8s 10 \
  --nodegroup gpu-workers

# Scale down: remove specific nodes (by Nova server UUID)
openstack coe cluster resize my-k8s 3 \
  --nodes-to-remove <uuid1> <uuid2>
```

### Upgrade a Cluster

```bash
# Upgrade to a new cluster template (new Kubernetes version)
# The new template should have a higher kube_tag
openstack coe cluster upgrade my-k8s new-k8s-template-id

# Upgrade with a maximum rolling batch size (nodes upgraded in batches)
openstack coe cluster upgrade my-k8s new-k8s-template-id \
  --max-batch-size 2

# Upgrade only a specific node group
openstack coe cluster upgrade my-k8s new-k8s-template-id \
  --nodegroup default-worker
```

### Delete a Cluster

```bash
openstack coe cluster delete my-k8s

# Delete multiple clusters
openstack coe cluster delete my-k8s my-k8s-dev

# Force delete (skip graceful node drain — use with care)
openstack coe cluster delete --force my-k8s
```

---

## Node Groups

### Create a Node Group

```bash
# Add a standard node group with a different flavor
openstack coe nodegroup create my-k8s high-memory-workers \
  --node-count 3 \
  --flavor m1.mem-xlarge \
  --image fedora-coreos-40

# Add a GPU node group
openstack coe nodegroup create my-k8s gpu-workers \
  --node-count 2 \
  --flavor m1.xlarge-gpu \
  --image fedora-coreos-40 \
  --label nvidia_driver_version=550.54.14

# Auto-scaling node group with min/max bounds
openstack coe nodegroup create my-k8s autoscale-workers \
  --node-count 3 \
  --min-node-count 2 \
  --max-node-count 15 \
  --flavor m1.medium \
  --image fedora-coreos-40 \
  --label auto_scaling_enabled=true

# Node group with a custom availability zone
openstack coe nodegroup create my-k8s az2-workers \
  --node-count 5 \
  --flavor m1.medium \
  --image fedora-coreos-40 \
  --availability-zone nova-az2
```

### List Node Groups

```bash
openstack coe nodegroup list my-k8s

# Show detailed information
openstack coe nodegroup list my-k8s --detail

# Filter by role
openstack coe nodegroup list my-k8s --role worker
openstack coe nodegroup list my-k8s --role master
```

### Show a Node Group

```bash
openstack coe nodegroup show my-k8s default-worker
openstack coe nodegroup show my-k8s gpu-workers
```

### Update a Node Group

```bash
# Change node count
openstack coe nodegroup update my-k8s gpu-workers replace node_count=4

# Update min/max for auto-scaling
openstack coe nodegroup update my-k8s autoscale-workers replace min_node_count=3
openstack coe nodegroup update my-k8s autoscale-workers replace max_node_count=20
```

### Delete a Node Group

```bash
openstack coe nodegroup delete my-k8s gpu-workers

# Only custom node groups can be deleted;
# default-master and default-worker cannot be deleted independently
```

---

## Certificate Management

```bash
# Show the cluster CA certificate (PEM format)
openstack coe ca show my-k8s

# Rotate the cluster CA (generates new CA; existing client certs become invalid)
openstack coe ca rotate my-k8s

# Sign a CSR against the cluster CA
# (used for adding additional admin users to a cluster)
openstack coe ca sign my-k8s --csr client.csr
```

---

## Quota Management

```bash
# Show quota for the current project
openstack coe quota list

# Show quota for a specific project (admin)
openstack coe quota list --project <project-id>

# Create a quota for a project (admin)
openstack coe quota create --project <project-id> --resource Cluster --hard-limit 20

# Update a quota
openstack coe quota update --project <project-id> --resource Cluster --hard-limit 30

# Delete a quota (reverts to default)
openstack coe quota delete --project <project-id> --resource Cluster
```

---

## Debugging Cluster Failures

Cluster creation failures are almost always Heat stack failures. Use these commands to diagnose:

```bash
# Get the Heat stack ID for a failed cluster
STACK_ID=$(openstack coe cluster show my-k8s -c stack_id -f value)

# Show the top-level Heat stack
openstack stack show $STACK_ID

# List all events (including nested) for the cluster's Heat stack
openstack stack event list --nested-depth 3 $STACK_ID

# List failed resources
openstack stack resource list --nested-depth 3 --filter status=CREATE_FAILED $STACK_ID

# Show a specific failed resource
openstack stack resource show $STACK_ID <resource-name>

# SSH to a master node (if it was created before failure)
MASTER_IP=$(openstack coe cluster show my-k8s -c master_addresses -f value | head -1)
ssh -i ~/.ssh/mykey fedora@$MASTER_IP

# On the master node, check cloud-init logs
sudo journalctl -u cloud-init -f
sudo cat /var/log/cloud-init-output.log

# Check kubelet and etcd
sudo systemctl status kubelet
sudo systemctl status etcd
sudo journalctl -u kubelet --since "30 minutes ago"
```

---

## Magnum Configuration Reference

Configuration file: `/etc/magnum/magnum.conf`

```ini
[DEFAULT]
# RPC transport URL
transport_url = rabbit://magnum:rabbit_pass@10.0.0.10:5672/

# Log settings
log_file = /var/log/magnum/magnum.log
log_dir  = /var/log/magnum
debug    = false

# Number of magnum-conductor workers
conductor_count = 4

[api]
# magnum-api bind address and port
host = 0.0.0.0
port = 9511
workers = 4

[database]
connection = mysql+pymysql://magnum:magnum_db_pass@10.0.0.10/magnum
max_pool_size = 20
max_overflow  = 20
pool_timeout  = 30

[keystone_authtoken]
www_authenticate_uri = http://10.0.0.10:5000
auth_url             = http://10.0.0.10:5000
memcached_servers    = 10.0.0.10:11211
auth_type            = password
project_domain_name  = Default
user_domain_name     = Default
project_name         = service
username             = magnum
password             = magnum_service_password

[trust]
# Service user for Keystone trust operations
# Magnum creates a trust so the conductor can act on behalf of the cluster owner
trustee_domain_admin_password = trust_domain_password
trustee_domain_admin_id       = <domain-admin-user-uuid>
trustee_domain_id             = <magnum-trustee-domain-uuid>
cluster_user_trust            = true

[cinder]
# Default volume type for etcd and boot volumes
default_docker_volume_type = __DEFAULT__

[oslo_policy]
enforce_scope       = true
enforce_new_defaults = true

[oslo_messaging_rabbit]
rabbit_ha_queues             = true
heartbeat_timeout_threshold  = 60

[cluster]
# Maximum number of nodes per cluster
max_nodecount = 500

[barbican_client]
# Barbican endpoint for certificate storage (optional)
endpoint_type = internalURL
```

---

## Service Management

```bash
# Start services (systemd)
systemctl start openstack-magnum-api
systemctl start openstack-magnum-conductor

# Enable on boot
systemctl enable openstack-magnum-api
systemctl enable openstack-magnum-conductor

# View logs
journalctl -u openstack-magnum-conductor -f
tail -f /var/log/magnum/magnum-conductor.log

# Initialize or upgrade the database schema
magnum-db-manage upgrade

# List available cluster driver types
openstack coe cluster template list --supported-drivers
```
