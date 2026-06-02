# Manila Operations

## Shares

### Create a Share

```bash
openstack share create --share-type default --name my-share NFS 10
```

- Protocol: `NFS`, `CIFS`, `CephFS`, `GlusterFS`, `HDFS`, `MAPRFS` (driver-dependent)
- Size: in GiB
- `--share-type`: required unless a default share type is configured
- `--availability-zone`: pin the share to a specific AZ
- `--share-network`: required for DHSS=true share types
- `--description`: human-readable description
- `--public`: make the share visible to all projects (admin only)
- `--share-group`: place in an existing share group
- `--snapshot-id`: create from an existing snapshot
- `--metadata key=value`: arbitrary key-value metadata

### List Shares

```bash
openstack share list
openstack share list --all-projects          # admin: show all tenants
openstack share list --status available
openstack share list --share-type default
openstack share list --long                  # include extra columns
openstack share list --name my-share
```

### Show Share Details

```bash
openstack share show my-share
```

The output includes `export_locations` — the mount paths clients use. For NFS these look like `10.0.1.10:/path/to/share`.

### Delete a Share

```bash
openstack share delete my-share
openstack share delete my-share --force      # admin: bypass status checks
```

A share must not have dependent snapshots. Delete snapshots first, or use `--force`.

### Set Share Properties

```bash
openstack share set my-share --name new-name
openstack share set my-share --description "Updated description"
openstack share set my-share --public True
openstack share set my-share --property key=value
openstack share unset my-share --property key
```

### Extend a Share

```bash
openstack share extend my-share 20
```

Grows the share to 20 GiB. The new size must be larger than the current size. The share must be in `available` state. Extension may be immediate or require backend provisioning time.

### Shrink a Share

```bash
openstack share shrink my-share 15
```

Reduces the share to 15 GiB. The new size must be smaller than the current size and larger than current usage. Not all backends support shrink. The share must be in `available` state.

### Show Export Locations

```bash
openstack share export location list my-share
openstack share export location show my-share <export-location-id>
```

Export locations are mount paths. For NFS: `10.0.1.10:/path/to/share`. For CephFS: `10.0.0.5:6789:/volumes/_nogroup/share-uuid`.

## Share Types

### Create a Share Type

```bash
openstack share type create my-type true
```

The second positional argument is the value of `driver_handles_share_servers`:
- `true` — DHSS=true (Manila manages share servers)
- `false` — DHSS=false (backend manages its own servers)

Additional options:
```bash
openstack share type create my-type false \
  --extra-spec snapshot_support=True \
  --extra-spec create_share_from_snapshot_support=True \
  --extra-spec revert_to_snapshot_support=True \
  --description "NFS shares on NetApp"
```

### List Share Types

```bash
openstack share type list
openstack share type list --all    # admin: include private types
openstack share type show my-type
```

### Set/Unset Extra Specs

```bash
openstack share type set my-type --extra-spec snapshot_support=True
openstack share type set my-type --extra-spec replication_type=readable
openstack share type unset my-type --extra-spec snapshot_support
```

### Delete a Share Type

```bash
openstack share type delete my-type
```

Cannot delete a type that has shares using it.

## Share Networks

Share networks are only needed for DHSS=true share types.

### Create a Share Network

```bash
openstack share network create \
  --neutron-net-id NET_ID \
  --neutron-subnet-id SUBNET_ID \
  share-net
```

- `NET_ID`: UUID of the Neutron network
- `SUBNET_ID`: UUID of the Neutron subnet within that network
- `--description`: optional description
- `--availability-zone`: restrict to a specific AZ

```bash
# Look up your Neutron network and subnet UUIDs
openstack network list
openstack subnet list --network private
```

### List and Show Share Networks

```bash
openstack share network list
openstack share network show share-net
```

### Add a Subnet to an Existing Share Network

```bash
openstack share network subnet create \
  --neutron-net-id NET_ID2 \
  --neutron-subnet-id SUBNET_ID2 \
  --availability-zone az2 \
  share-net
```

This extends an existing share network into another AZ or subnet.

### Delete a Share Network

```bash
openstack share network delete share-net
```

Fails if share servers (and thus shares) are still using the network. Delete shares first.

## Access Rules

Access rules control which clients can mount a share.

### Grant Access

```bash
openstack share access create my-share ip 10.0.0.0/24 --access-level rw
```

Access types:

| Type | Value Format | Description |
|---|---|---|
| `ip` | CIDR (IPv4 or IPv6) | Client IP range — most common for NFS |
| `user` | username | Windows/Active Directory user — for CIFS |
| `cert` | TLS certificate common name | For CephFS with x509 |
| `cephx` | CephX key name | For CephFS with CephX auth |

Access levels:
- `rw` — read-write (default)
- `ro` — read-only

```bash
# Multiple access rules can be added
openstack share access create my-share ip 192.168.1.0/24 --access-level ro
openstack share access create my-share user DOMAIN\\alice --access-level rw
```

### List Access Rules

```bash
openstack share access list my-share
```

### Delete an Access Rule

```bash
openstack share access delete my-share <access-id>
```

Get the `<access-id>` from `openstack share access list my-share`.

## Snapshots

### Create a Snapshot

```bash
openstack share snapshot create my-share --name my-snap
openstack share snapshot create my-share --name my-snap --description "Before upgrade"
openstack share snapshot create my-share --force   # snapshot even if share is busy
```

### List Snapshots

```bash
openstack share snapshot list
openstack share snapshot list --share my-share
openstack share snapshot list --status available
openstack share snapshot list --all-projects   # admin
```

### Show a Snapshot

```bash
openstack share snapshot show my-snap
```

### Delete a Snapshot

```bash
openstack share snapshot delete my-snap
openstack share snapshot delete my-snap --force
```

### Create a Share from a Snapshot

```bash
openstack share create \
  --share-type default \
  --snapshot-id <snapshot-id> \
  --name share-from-snap \
  NFS 10
```

The new share inherits the snapshot's content. Requires `create_share_from_snapshot_support=True` on the share type.

### Revert a Share to a Snapshot

```bash
openstack share revert-to-snapshot my-snap
```

Reverts `my-share` in-place to the state of `my-snap`. Destructive — all data written after the snapshot is lost. Requires `revert_to_snapshot_support=True` on the share type.

## Share Groups

### Create a Share Group Type

```bash
openstack share group type create my-group-type default
```

Associates the group type with the `default` share type. Multiple share types can be specified.

### Create a Share Group

```bash
openstack share group create \
  --share-group-type my-group-type \
  --share-type default \
  my-group
```

### List and Show Share Groups

```bash
openstack share group list
openstack share group show my-group
```

### Create a Share in a Group

```bash
openstack share create \
  --share-type default \
  --share-group my-group \
  --name grouped-share \
  NFS 10
```

### Create a Share Group Snapshot

```bash
openstack share group snapshot create my-group --name my-group-snap
openstack share group snapshot list
openstack share group snapshot show my-group-snap
openstack share group snapshot delete my-group-snap
```

### Delete a Share Group

```bash
openstack share group delete my-group
```

All member shares must be deleted first.

## Share Migration

Migration moves a share from one backend to another (or one availability zone to another).

### Start Migration

```bash
openstack share migration start \
  my-share \
  <destination-host> \
  --new-share-type new-type \
  --new-share-network new-net \
  --writable \
  --preserve-metadata \
  --preserve-snapshots \
  --nondisruptive
```

- `<destination-host>`: the `manila-share` host in `host@backend#pool` format (e.g., `manila@cephfs#ceph`)
- `--new-share-type`: optional; use a different share type on the destination
- `--new-share-network`: optional; use a different share network (for DHSS=true destinations)
- `--writable`: keep the share writable during migration (driver support required)
- `--preserve-metadata`: copy all share metadata to the destination
- `--preserve-snapshots`: migrate snapshots too (driver support required)
- `--nondisruptive`: attempt to migrate without interrupting client I/O

### Check Migration Progress

```bash
openstack share migration get progress my-share
openstack share show my-share   # check task_state field
```

### Complete Migration (Cutover)

```bash
openstack share migration complete my-share
```

Triggers the final cutover. After this call, the share export location changes to the destination backend. Clients must remount.

### Cancel Migration

```bash
openstack share migration cancel my-share
```

Only valid before `migration_complete` is called.

## Share Replicas

Replication is available when the share type has `replication_type` set.

### Create a Replica

```bash
openstack share replica create my-share \
  --availability-zone az2 \
  --share-network share-net-az2
```

- `--availability-zone`: the AZ where the replica should reside
- `--share-network`: required for DHSS=true backends

### List Replicas

```bash
openstack share replica list --share-id my-share
```

Output includes `replica_state` (`active`, `in_sync`, `out_of_sync`) and `status`.

### Show a Replica

```bash
openstack share replica show <replica-id>
```

### Promote a Replica

```bash
openstack share replica promote <replica-id>
```

Makes the specified replica the new `active` replica. The previous active becomes a secondary. Useful for manual failover.

### Resync a Replica

```bash
openstack share replica resync <replica-id>
```

Triggers an immediate out-of-band sync from the active replica to the specified secondary. Normally sync is periodic (backend-controlled).

### Delete a Replica

```bash
openstack share replica delete <replica-id>
```

Cannot delete the `active` replica. Promote another replica first.

## Administration

### List Share Backends and Pools

```bash
openstack share pool list
openstack share pool list --long               # includes capacity and capabilities
openstack share pool list --host manila@cephfs
openstack share pool list --backend cephfs
openstack share pool list --pool ceph-pool-1
openstack share pool list --capabilities snapshot_support=True
```

### Manage an Existing Share

Bring an existing export (created outside of Manila) under Manila management:

```bash
openstack share manage \
  <host> \
  NFS \
  <export-path> \
  --name managed-share \
  --share-type default \
  --driver-options key=value
```

### Unmanage a Share

Remove a share from Manila without deleting the data:

```bash
openstack share unmanage my-share
```

### Quota Management

```bash
openstack quota show --project my-project
openstack quota set --shares 50 --gigabytes 10000 --snapshots 100 --project my-project
openstack quota delete --project my-project   # revert to default quota
```

Default quota defaults (configurable in `manila.conf`):
- `quota_shares = 50`
- `quota_gigabytes = 1000`
- `quota_snapshots = 50`
- `quota_share_networks = 10`
- `quota_share_groups = 10`
- `quota_share_group_snapshots = 10`
- `quota_share_replicas = 100`
- `quota_replica_gigabytes = 1000`

## Configuration Reference

### [DEFAULT] Section

```ini
[DEFAULT]
# Transport URL for oslo.messaging (RabbitMQ)
transport_url = rabbit://guest:guest@controller:5672/

# Hostname for this node (used in pool reporting)
host = manila@cephfs

# Enabled share backends (comma-separated; each must have its own [section])
enabled_share_backends = cephfs

# Default share type name (used when no type is specified)
default_share_type = default

# Share server cleanup interval (DHSS=true, in seconds)
unused_share_server_cleanup_interval = 10

# API rate limiting
api_rate_limit = true
```

### [database] Section

```ini
[database]
connection = mysql+pymysql://manila:MANILA_DBPASS@controller/manila
max_pool_size = 10
max_overflow = 20
pool_timeout = 30
```

### [keystone_authtoken] Section

```ini
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = manila
password = MANILA_PASS
```

### [oslo_messaging_rabbit] Section

```ini
[oslo_messaging_rabbit]
rabbit_ha_queues = true
amqp_durable_queues = true
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
rabbit_max_retries = 0
```

### Generic Driver (DHSS=true, NFS-Ganesha/Samba via VMs)

```ini
[generic]
share_driver = manila.share.drivers.generic.GenericShareDriver
driver_handles_share_servers = True

# Nova image used for share server VMs
service_image_name = manila-service-image

# Flavor for share server VMs
service_instance_flavor_id = 100

# Nova keypair injected into share server VMs
path_to_public_key = /etc/manila/manila.id_rsa.pub
path_to_private_key = /etc/manila/manila.id_rsa

# Neutron network for service (management) network
service_net_name = manila_service_net

# Connect share server to tenant network via this interface type
interface_driver = manila.network.linux.interface.OVSInterfaceDriver

# Protocol: nfs or smb
share_volume_fstype = ext4
```

### CephFS Driver (DHSS=false)

```ini
[cephfs]
share_driver = manila.share.drivers.cephfs.driver.CephFSDriver
driver_handles_share_servers = False

# CephFS backend name in manila.conf
share_backend_name = CEPHFS

# Ceph cluster name
cephfs_cluster_name = ceph

# Auth ID Manila uses to communicate with Ceph
cephfs_auth_id = manila

# Path to ceph.conf and keyring
cephfs_conf_path = /etc/ceph/ceph.conf
cephfs_keyring_path = /etc/ceph/ceph.client.manila.keyring

# Native CephFS (kernel or FUSE) or via NFS-Ganesha
cephfs_protocol_helper_type = CEPHFS

# Volume mode: either 'native' or 'nfs'
# For NFS-Ganesha: cephfs_protocol_helper_type = NFS
```

### NetApp ONTAP Driver

```ini
[netapp]
share_driver = manila.share.drivers.netapp.common.NetAppDriver
driver_handles_share_servers = False

share_backend_name = NETAPP_NAS

# ONTAP management LIF (cluster or SVM management IP)
netapp_server_hostname = 192.168.1.100
netapp_server_port = 443
netapp_transport_type = https

netapp_login = admin
netapp_password = NETAPP_PASS

# SVM (Vserver) to use
netapp_vserver = vs_manila

# Root aggregate for SVM (DHSS=true mode only)
# netapp_root_volume_aggregate = aggr1

# Aggregate for share volumes
netapp_aggregate_name_search_pattern = aggr.*

# Thin provisioning
netapp_thin_provisioned = true
```
