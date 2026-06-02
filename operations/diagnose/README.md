# Diagnose — Symptom-Based Troubleshooting Decision Trees

Use this skill when an operator reports a problem and you need to systematically narrow down the cause. Each section follows the pattern: symptom → diagnostic commands → likely causes → resolution.

**Before any diagnosis**: confirm which release is running (`openstack versions show` or `openstack --version`) and collect the output of `openstack endpoint list` so you know the service topology.

---

## Instance Won't Launch

**Symptom**: `openstack server create` returns an error, or the server reaches ERROR state.

### Step 1 — Check nova-scheduler logs

```bash
# SystemD deployments
journalctl -u devstack@n-sch --since "10 minutes ago" | grep -i "error\|no valid host\|filter"

# Package deployments
tail -200 /var/log/nova/nova-scheduler.log | grep -i "error\|no valid host\|filter\|NoValidHost"
```

Look for `NoValidHost`, filter names that rejected all hosts, or placement errors. The log line includes which filters eliminated hosts.

### Step 2 — Check Placement allocation candidates

```bash
# List allocation candidates for a flavor
openstack allocation candidate list \
  --resource VCPU=2,MEMORY_MB=4096,DISK_GB=40

# If the list is empty, no host can satisfy the request
# Check each compute node's inventory
openstack resource provider list
openstack resource provider inventory list <RESOURCE_PROVIDER_UUID>

# Check existing allocations on a provider
openstack resource provider allocation list <RESOURCE_PROVIDER_UUID>

# Show usages
openstack resource provider usage show <RESOURCE_PROVIDER_UUID>
```

Empty candidate list means either no capacity or a trait/aggregate mismatch.

### Step 3 — Check compute node capacity

```bash
# Hypervisor summary
openstack hypervisor stats show

# Per-hypervisor detail
openstack hypervisor list --long
openstack hypervisor show <HYPERVISOR_HOSTNAME>

# Check for disabled/down compute services
openstack compute service list
openstack compute service list --service nova-compute
```

If a compute service shows `down`, the nova-compute process on that node is not running or not connecting to RabbitMQ. If `disabled`, it was manually disabled.

```bash
# On the affected compute node — check nova-compute
systemctl status devstack@n-cpu   # DevStack
systemctl status nova-compute     # Package
journalctl -u nova-compute --since "10 minutes ago"
```

### Step 4 — Check Neutron port binding

```bash
# After a failed launch, find the port
openstack port list --server <SERVER_ID>

# Show port details — look at binding:vif_type and binding:vnic_type
openstack port show <PORT_ID>
```

If `binding:vif_type` is `binding_failed`, the ML2 mechanism driver could not bind the port to the host.

```bash
# Check Neutron agents on the target compute node
openstack network agent list --host <COMPUTE_HOSTNAME>

# Check Neutron server logs
journalctl -u devstack@q-svc --since "10 minutes ago" | grep -i "port\|bind\|error"
tail -200 /var/log/neutron/neutron-server.log | grep -i "bind\|error"
```

Common causes: L2 agent is down on the compute node, network not reachable from that host, physnet mapping mismatch.

### Step 5 — Check Glance image availability

```bash
openstack image show <IMAGE_ID>
# Verify status = active, size > 0, visibility allows the project
```

If status is `queued` or `killed`, the image upload failed or the Glance store backend is unavailable.

```bash
# Check Glance API logs
journalctl -u devstack@g-api --since "10 minutes ago"
tail -100 /var/log/glance/glance-api.log | grep -i "error"

# Verify the image file actually exists in the store
openstack image show <IMAGE_ID> -f json | jq '.["direct_url"]'
```

### Step 6 — Check Cinder volume (boot-from-volume only)

```bash
openstack volume show <VOLUME_ID>
# Must be status=available (or in-use for attach-existing)

# Check Cinder logs
journalctl -u devstack@c-api --since "10 minutes ago"
tail -100 /var/log/cinder/cinder-api.log | grep -i "error"
```

### Common Resolutions

| Cause | Resolution |
|---|---|
| NoValidHost / no candidates | Free capacity, fix traits, check aggregate membership |
| Compute service down | Restart nova-compute, fix RabbitMQ connectivity |
| Port binding failed | Restart L2 agent on compute node, fix physnet mapping |
| Image not active | Re-upload image, check Glance backend |
| Over quota | Increase quota (`openstack quota set`) or delete unused resources |
| Scheduler filter rejecting all hosts | Review `NoValidHost` log lines to identify which filter; adjust flavor/image metadata or host aggregate |

---

## Instance Has No Network

**Symptom**: Instance boots but cannot reach the network — no DHCP response, no ping, no connectivity.

### Step 1 — Check security groups

```bash
# List security group rules for the server's port
openstack port show <PORT_ID>
# Note the security_group_ids

openstack security group rule list <SECURITY_GROUP_ID>
# Verify ingress/egress rules allow expected traffic
# Default security group blocks all ingress from outside the group
```

Add required rules:

```bash
openstack security group rule create --protocol icmp --ingress <SG_ID>
openstack security group rule create --protocol tcp --dst-port 22 --ingress <SG_ID>
```

### Step 2 — Check port status

```bash
openstack port show <PORT_ID>
# status must be ACTIVE
# binding:vif_type must NOT be binding_failed
# binding:host_id must match the compute node
```

If `status = DOWN` after the instance is ACTIVE, the L2 agent has not plugged the port.

### Step 3 — Check DHCP agent

```bash
# List agents servicing the network
openstack network agent list --network <NETWORK_ID>

# Verify DHCP agent is alive and active
openstack network agent show <DHCP_AGENT_ID>

# Check DHCP namespace on the network node
# Find the namespace: ip netns list | grep qdhcp
sudo ip netns list | grep qdhcp-<NETWORK_ID_PREFIX>

# Verify dnsmasq is running inside the namespace
sudo ip netns exec qdhcp-<NETWORK_ID> ps aux | grep dnsmasq

# Check if the DHCP port exists
openstack port list --network <NETWORK_ID> --device-owner network:dhcp
```

DHCP agent logs:

```bash
journalctl -u devstack@q-dhcp --since "10 minutes ago"
tail -100 /var/log/neutron/neutron-dhcp-agent.log | grep -i "error\|<NETWORK_ID_SHORT>"
```

### Step 4 — Check L2 agent on the compute node

```bash
# OVS deployments
openstack network agent list --host <COMPUTE_HOST> --agent-type "Open vSwitch agent"

# OVN deployments
openstack network agent list --host <COMPUTE_HOST> --agent-type "OVN Controller agent"

# Linux Bridge deployments
openstack network agent list --host <COMPUTE_HOST> --agent-type "Linux bridge agent"
```

If the agent is down, SSH to the compute node and check:

```bash
# OVS
systemctl status neutron-openvswitch-agent
journalctl -u neutron-openvswitch-agent --since "10 minutes ago"

# OVN
systemctl status ovn-controller
ovn-appctl -t ovn-controller connection-status

# Linux Bridge
systemctl status neutron-linuxbridge-agent
```

### Step 5 — Check bridge and OVS configuration

```bash
# On the compute node — OVS
sudo ovs-vsctl show
# Verify the integration bridge (br-int) exists
# Verify the physical bridge (br-ex, br-provider) exists and has the physical NIC
# Verify the patch ports between br-int and br-ex

# Check the tap device for the instance is on br-int
sudo ovs-vsctl list-ports br-int | grep tap

# OVN — check logical port status
sudo ovn-nbctl show | grep -A5 "<PORT_ID_PREFIX>"
sudo ovn-sbctl show | grep -A5 "<PORT_ID_PREFIX>"

# Linux Bridge
brctl show
ip link show
```

### Step 6 — Check MTU mismatch

```bash
# Check network MTU setting
openstack network show <NETWORK_ID> | grep mtu

# Check instance MTU (inside instance or via console)
openstack console log show <SERVER_ID> | grep -i mtu
# Or: ip link show eth0 (inside instance)

# Check physical NIC MTU on compute node
ip link show <PHYSICAL_NIC>

# Check OVS bridge MTU
ovs-vsctl list interface br-int | grep mtu
```

MTU mismatch causes intermittent packet loss or inability to reach larger MTU destinations. Fix: set network MTU to match physical fabric minus encapsulation overhead (VXLAN: subtract 50, GRE: subtract 42).

```bash
openstack network set --mtu <CORRECT_MTU> <NETWORK_ID>
```

### Common Resolutions

| Cause | Resolution |
|---|---|
| Security group blocks traffic | Add correct ingress/egress rules |
| DHCP agent down | Restart neutron-dhcp-agent, reschedule network |
| L2 agent down on compute | Restart OVS/OVN/LinuxBridge agent |
| tap device not bridged | Restart nova-compute and L2 agent |
| MTU mismatch | Set correct MTU on Neutron network |
| No DHCP namespace | `openstack network agent add network <DHCP_AGENT_ID> <NETWORK_ID>` |

---

## API Returns 401 Unauthorized

**Symptom**: OpenStack CLI or API calls return `401 Unauthorized` or `The request you have made requires authentication`.

### Step 1 — Check token validity

```bash
# Issue a new token and inspect it
openstack token issue
# If this fails, credentials or Keystone endpoint is the problem

# Check token expiry
openstack token issue -f json | jq '.expires'

# Verify environment variables
env | grep OS_
# Check OS_AUTH_URL, OS_USERNAME, OS_PASSWORD, OS_PROJECT_NAME, OS_DOMAIN_NAME
```

### Step 2 — Check endpoint catalog

```bash
openstack catalog list
# Verify the service endpoints are correct and reachable
openstack endpoint list
openstack endpoint list --service identity

# Test connectivity to Keystone
curl -v <OS_AUTH_URL>/v3
```

### Step 3 — Check keystone_authtoken configuration

Every service that validates tokens has a `[keystone_authtoken]` section:

```bash
# Nova
grep -A 20 '\[keystone_authtoken\]' /etc/nova/nova.conf

# Neutron
grep -A 20 '\[keystone_authtoken\]' /etc/neutron/neutron.conf

# Cinder
grep -A 20 '\[keystone_authtoken\]' /etc/cinder/cinder.conf
```

Verify `auth_url`, `username`, `password`, `project_name`, and domain settings. Confirm the service user exists:

```bash
openstack user show nova
openstack user show neutron
openstack role assignment list --user nova --project service
```

### Step 4 — Check Keystone service logs

```bash
journalctl -u devstack@keystone --since "10 minutes ago" | grep -i "error\|401\|invalid"
tail -100 /var/log/keystone/keystone.log | grep -i "error\|401"

# Apache-fronted Keystone
tail -100 /var/log/apache2/keystone_error.log
tail -100 /var/log/httpd/keystone_error.log
```

### Step 5 — Check policy files

```bash
# Locate and check policy files
ls /etc/nova/policy.yaml /etc/nova/policy.json 2>/dev/null
ls /etc/neutron/policy.yaml 2>/dev/null

# Validate a specific rule
oslopolicy-checker \
  --policy /etc/nova/policy.yaml \
  --rule "os_compute_api:servers:create" \
  --target project_id=<PROJECT_ID> \
  --credentials user_id=<USER_ID>,roles=member
```

### Common Resolutions

| Cause | Resolution |
|---|---|
| Token expired | Re-authenticate (`openstack token issue`) |
| Wrong credentials | Fix `OS_PASSWORD`, `OS_USERNAME` in RC file |
| Service user missing | Create service user and assign `service` role |
| Wrong auth_url | Fix `[keystone_authtoken] auth_url` in service config |
| Policy denying access | Review policy.yaml, check role assignments |
| Clock skew > 5 minutes | Sync NTP on all nodes |

---

## API Returns 403 Forbidden

**Symptom**: API calls return `403 Forbidden` — authentication succeeds but authorization fails.

### Step 1 — Identify the policy rule being checked

```bash
# The 403 response body usually names the rule
# Example: "Policy doesn't allow os_compute_api:servers:create to be performed"

# Locate the policy file
ls /etc/<service>/policy.yaml
```

### Step 2 — Check role assignments

```bash
# List roles for the user in the project
openstack role assignment list \
  --user <USER_ID_OR_NAME> \
  --project <PROJECT_ID_OR_NAME> \
  --names

# List all roles in the system
openstack role list

# Check for system-scoped roles (admin operations often need this)
openstack role assignment list --user <USER> --system all --names
```

### Step 3 — Validate with oslopolicy-checker

```bash
oslopolicy-checker \
  --policy /etc/nova/policy.yaml \
  --rule "os_compute_api:os-hypervisors:list" \
  --credentials roles=admin,project_id=<PROJECT_ID>
```

### Common Resolutions

| Cause | Resolution |
|---|---|
| Missing role | `openstack role add --user X --project Y <ROLE>` |
| Default policy too restrictive | Review and customize policy.yaml for the required role |
| Scope mismatch (project vs system) | Assign system-scoped admin role for admin operations |
| Deprecated policy rules | Upgrade policy.yaml to match the release defaults |

---

## API Returns 500 Internal Server Error

**Symptom**: API calls return `500 Internal Server Error`.

### Step 1 — Read service logs immediately

```bash
# Identify which service returned the 500
# Check that service's API log

# Nova
journalctl -u devstack@n-api --since "5 minutes ago" | grep -i "error\|traceback\|exception"
tail -200 /var/log/nova/nova-api.log | grep -i "error\|traceback"

# Neutron
journalctl -u devstack@q-svc --since "5 minutes ago" | grep -i "error\|traceback"
tail -200 /var/log/neutron/neutron-server.log

# Cinder
tail -200 /var/log/cinder/cinder-api.log | grep -i "error\|traceback"

# Glance
tail -200 /var/log/glance/glance-api.log | grep -i "error\|traceback"

# Keystone (often Apache)
tail -200 /var/log/apache2/error.log | grep -i "keystone"
```

Python tracebacks in the log pinpoint the exact failure.

### Step 2 — Check database connectivity

```bash
# Test MySQL/MariaDB connection
mysql -u <SERVICE_USER> -p<PASSWORD> -h <DB_HOST> <SERVICE_DB> -e "SELECT 1"

# Check connection pool from service config
grep -E "connection|pool" /etc/<service>/<service>.conf | grep -i "database\|sql"

# Check for too many connections
mysql -u root -p -e "SHOW STATUS LIKE 'Threads_connected';"
mysql -u root -p -e "SHOW VARIABLES LIKE 'max_connections';"
```

### Step 3 — Check RabbitMQ connectivity

Services that use RPC (Nova, Neutron, Cinder) fail with 500 when they cannot reach RabbitMQ:

```bash
# Check RabbitMQ is running
rabbitmqctl status

# Check service config for transport_url
grep "transport_url" /etc/<service>/<service>.conf

# Test connectivity from the service host
nc -zv <RABBITMQ_HOST> 5672

# Check for memory or disk alarms that block publishing
rabbitmqctl cluster_status | grep -A5 "alarms"
```

### Step 4 — Check configuration syntax

```bash
# Nova
nova-manage config validate 2>&1 | head -50

# Cinder
cinder-manage config validate 2>&1 | head -50

# For any service — check for missing required options
grep -i "error" /var/log/<service>/<service>-api.log | grep -i "config\|option\|unknown"
```

### Step 5 — Run DB migration check

```bash
nova-manage db version
nova-manage api_db version
neutron-db-manage current
cinder-manage db version
```

If the current version does not match the expected version for the release, a migration was not run.

```bash
# Run pending migrations (after backup)
nova-manage db sync
nova-manage api_db sync
neutron-db-manage upgrade heads
cinder-manage db sync
```

### Common Resolutions

| Cause | Resolution |
|---|---|
| DB connectivity failure | Fix `[database] connection` in service config, restart DB |
| DB migration not run | Run `<service>-manage db sync` after backing up DB |
| RabbitMQ unreachable | Fix `transport_url`, restart RabbitMQ |
| Config syntax error | Fix the offending option, restart service |
| Service ran out of workers | Increase `[DEFAULT] api_workers` |

---

## Volume Attach Fails

**Symptom**: `openstack server add volume` fails, or the volume stays in `attaching` state.

### Step 1 — Check Cinder volume status

```bash
openstack volume show <VOLUME_ID>
# status must be 'available' for a fresh attach
# If status is 'error', 'in-use', or 'attaching' (stuck), investigate further

# Check volume is not already attached
openstack volume show <VOLUME_ID> -f json | jq '.attachments'
```

### Step 2 — Check Nova instance status

```bash
openstack server show <SERVER_ID>
# Instance must be ACTIVE or SHUTOFF for attach
# SHELVED_OFFLOADED, SUSPENDED, MIGRATING — attachment may fail
```

### Step 3 — Check target protocol

**iSCSI:**

```bash
# On the compute node where the instance runs
iscsiadm -m session
# Should show the target if discovery succeeded

# Check iSCSI initiator
cat /etc/iscsi/initiatorname.iscsi

# Manual discovery (replace with your Cinder target IP)
iscsiadm -m discovery -t sendtargets -p <CINDER_TARGET_IP>:3260

# Check nova-compute logs for iSCSI errors
journalctl -u nova-compute --since "10 minutes ago" | grep -i "iscsi\|volume\|attach\|error"
```

**Ceph RBD:**

```bash
# Verify RBD access from the compute node
rbd ls <CINDER_POOL>
rbd info <CINDER_POOL>/<VOLUME_ID>

# Check ceph auth
ceph auth get client.cinder
ceph auth get client.nova

# Test RBD map
sudo rbd map <CINDER_POOL>/<VOLUME_ID> --id cinder --keyring /etc/ceph/ceph.client.cinder.keyring
```

**NFS:**

```bash
# Verify NFS mount
df -h | grep nfs
mount | grep nfs

# Check NFS share is accessible
showmount -e <NFS_SERVER>
```

### Step 4 — Check multipath (iSCSI with multipath)

```bash
# Check multipathd status
systemctl status multipathd
multipath -ll

# Check for failed paths
multipath -ll | grep -i "fail\|ghost"

# Check nova config for multipath
grep "use_multipath_for_image_xfer\|volume_use_multipath" /etc/nova/nova.conf
```

### Step 5 — Check driver-specific Cinder logs

```bash
journalctl -u devstack@c-vol --since "10 minutes ago" | grep -i "error\|traceback\|volume"
tail -200 /var/log/cinder/cinder-volume.log | grep -i "error\|traceback\|attach"

# Check which backend is configured
grep -A 20 '\[DEFAULT\]' /etc/cinder/cinder.conf | grep "enabled_backends\|volume_driver"
```

### Common Resolutions

| Cause | Resolution |
|---|---|
| Volume in error state | `openstack volume set --state available <VOL_ID>` then retry |
| iSCSI initiator not configured | Install `open-iscsi`, configure initiator name |
| RBD keyring missing on compute | Copy client.cinder and client.nova keyrings to compute nodes |
| Multipath misconfigured | Fix `/etc/multipath.conf`, restart `multipathd` |
| Cinder volume service down | Restart `cinder-volume`, check connectivity to storage backend |
| Stuck in 'attaching' state | `openstack volume set --state available <VOL_ID>` and reset attachment |



---

## Infrastructure Issues

For infrastructure-level troubleshooting (slow APIs, service startup failures, RabbitMQ, database), see [infrastructure.md](infrastructure.md).
