# Backup and Restore

This skill covers database backups, configuration backups, data backups, and full disaster recovery procedures for an OpenStack cloud. Follow these procedures before any upgrade, major maintenance, or on a regular schedule.

## When to Read This Skill

- Backing up the cloud before an upgrade
- Implementing a routine backup schedule
- Recovering from a partial or full infrastructure failure
- Recovering a specific service (database, configuration, image, or volume)
- Planning disaster recovery procedures

---

## What to Back Up

| Component | Type | Consequence of Loss |
|---|---|---|
| Service databases | MySQL/MariaDB | Loss of all resource state (instances, networks, volumes, etc.) |
| Service configurations | `/etc/<service>/` | Services fail to start; manual reconfiguration required |
| Keystone Fernet keys | `/etc/keystone/fernet-keys/` | All existing tokens invalidated; users must re-authenticate |
| Keystone credential keys | `/etc/keystone/credential-keys/` | Encrypted application credentials and EC2 credentials become unreadable |
| Glance images | Depends on backend | Loss of images; new instances cannot be launched |
| Cinder volumes | Depends on backend | Loss of tenant data |
| Swift rings | `/etc/swift/*.ring.gz` | Loss of object-to-node mapping; all object data becomes inaccessible |
| Barbican master KEK | Barbican config | All secrets become permanently unreadable |

---

## Database Backup

### Full Database Backup Per Service

Run on the MariaDB controller node. Replace the password placeholder with your actual credential.

```bash
# Keystone
mysqldump --single-transaction --routines --triggers keystone > keystone-backup.sql

# Nova (three databases)
mysqldump --single-transaction --routines --triggers nova > nova-backup.sql
mysqldump --single-transaction --routines --triggers nova_api > nova_api-backup.sql
mysqldump --single-transaction --routines --triggers nova_cell0 > nova_cell0-backup.sql

# Neutron
mysqldump --single-transaction --routines --triggers neutron > neutron-backup.sql

# Cinder
mysqldump --single-transaction --routines --triggers cinder > cinder-backup.sql

# Glance
mysqldump --single-transaction --routines --triggers glance > glance-backup.sql

# Heat
mysqldump --single-transaction --routines --triggers heat > heat-backup.sql

# Barbican
mysqldump --single-transaction --routines --triggers barbican > barbican-backup.sql

# Designate
mysqldump --single-transaction --routines --triggers designate > designate-backup.sql

# Ironic
mysqldump --single-transaction --routines --triggers ironic > ironic-backup.sql

# Magnum
mysqldump --single-transaction --routines --triggers magnum > magnum-backup.sql

# Octavia
mysqldump --single-transaction --routines --triggers octavia > octavia-backup.sql

# Manila
mysqldump --single-transaction --routines --triggers manila > manila-backup.sql
```

### Full Database Backup Script

```bash
#!/bin/bash
# backup-openstack-dbs.sh
# Run from the MariaDB controller node with DB admin access

BACKUP_DIR="/backup/openstack-db/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

DATABASES=(
  keystone
  nova
  nova_api
  nova_cell0
  neutron
  cinder
  glance
  heat
  barbican
  designate
  ironic
  magnum
  octavia
  manila
)

echo "Starting database backups to $BACKUP_DIR"

for db in "${DATABASES[@]}"; do
  echo "Backing up $db..."
  mysqldump \
    --single-transaction \
    --routines \
    --triggers \
    --add-drop-database \
    --databases "$db" \
    > "$BACKUP_DIR/${db}.sql"
  gzip "$BACKUP_DIR/${db}.sql"
  echo "  → $BACKUP_DIR/${db}.sql.gz ($(du -sh "$BACKUP_DIR/${db}.sql.gz" | cut -f1))"
done

echo "Database backup complete: $BACKUP_DIR"
ls -lh "$BACKUP_DIR"
```

### Backup with Credentials

```bash
# Using ~/.my.cnf to avoid password in command history
cat > ~/.my.cnf <<EOF
[client]
user = root
password = YOUR_DB_ROOT_PASSWORD
EOF
chmod 600 ~/.my.cnf

# Now mysqldump uses ~/.my.cnf automatically
mysqldump --single-transaction --routines --triggers keystone > keystone-backup.sql
```

---

## Configuration Backup

### Back Up All Service Configurations

```bash
tar czf openstack-config-backup.tar.gz \
  /etc/keystone \
  /etc/nova \
  /etc/neutron \
  /etc/cinder \
  /etc/glance \
  /etc/heat \
  /etc/horizon \
  /etc/octavia \
  /etc/designate \
  /etc/barbican \
  /etc/ironic \
  /etc/magnum \
  /etc/manila \
  /etc/placement \
  /etc/swift
```

### Include HAProxy and Systemd Unit Overrides

```bash
tar czf openstack-infra-config-backup.tar.gz \
  /etc/haproxy/haproxy.cfg \
  /etc/keepalived/ \
  /etc/rabbitmq/ \
  /etc/mysql/ \
  /etc/memcached.conf \
  /etc/systemd/system/nova-*.service.d \
  /etc/systemd/system/neutron-*.service.d \
  /etc/systemd/system/cinder-*.service.d
```

---

## Keystone Fernet and Credential Keys

Fernet and credential keys are critical. Without them:
- Existing tokens issued with the lost fernet keys cannot be validated.
- Application credentials and EC2 credentials encrypted with the lost credential keys are permanently unreadable.

```bash
# Back up both key repositories
tar czf fernet-keys-backup.tar.gz \
  /etc/keystone/fernet-keys/ \
  /etc/keystone/credential-keys/

# Verify the backup is not empty
tar tzf fernet-keys-backup.tar.gz
```

Back up the fernet key archive off-node immediately — storing it only on the Keystone controller defeats the purpose.

### Key Rotation Schedule

Keystone fernet keys must be rotated on a schedule shorter than the token expiry. The rotation creates a new primary key and promotes the current primary to secondary. Backed-up tokens remain valid until they expire if the secondary key is present.

```bash
# Rotate fernet keys (run on all Keystone controllers with shared key repository)
keystone-manage fernet_rotate --keystone-user keystone --keystone-group keystone

# Rotate credential keys
keystone-manage credential_rotate --keystone-user keystone --keystone-group keystone
```

After each rotation, back up the updated key repositories.

---

## Glance Image Backup

The backup strategy depends on the Glance storage backend.

### File Backend (default)

Images are stored as files under `/var/lib/glance/images/`. Each file is named by its UUID.

```bash
# List images with UUIDs and their backing files
openstack image list -f value -c ID -c Name

# Back up all image files
rsync -avz /var/lib/glance/images/ /backup/glance-images/

# Or with tar
tar czf glance-images-backup.tar.gz /var/lib/glance/images/
```

### Ceph Backend (rbd)

Images are stored as RBD objects in a Ceph pool (typically `images`).

```bash
# List all images in the pool
rbd ls images

# Export a specific image
rbd export images/<image-uuid> /backup/glance/<image-uuid>.img

# Export all images
for img in $(rbd ls images); do
  rbd export "images/$img" "/backup/glance/$img.img"
  echo "Exported $img"
done

# Alternative: RBD snapshot and export-diff for incremental backups
rbd snap create images/<image-uuid>@backup-$(date +%Y%m%d)
rbd export-diff images/<image-uuid>@backup-$(date +%Y%m%d) /backup/glance/<image-uuid>-delta.img
```

### Swift Backend

Images are stored in Swift object containers. Swift's built-in replication provides redundancy. For a true off-cluster backup:

```bash
# Download all images from Swift to a local backup location
for image_id in $(openstack image list -f value -c ID); do
  openstack image save --file "/backup/glance/${image_id}.img" "$image_id"
  echo "Saved $image_id"
done
```

---

## Cinder Volume Backup

### Using the Cinder Backup Service

Cinder has a built-in backup mechanism (`cinder-backup`) that can target Swift or Ceph.

```bash
# Create a backup of a volume (volume must be in 'available' or 'in-use' state)
openstack volume backup create --name vol-backup my-volume

# Create an incremental backup
openstack volume backup create --name vol-backup-incr --incremental my-volume

# List backups
openstack volume backup list

# Show backup details
openstack volume backup show vol-backup

# Restore a backup to a new volume
openstack volume backup restore vol-backup

# Restore to an existing volume (overwrites data)
openstack volume backup restore --volume my-restored-volume vol-backup
```

### LVM Backend — Direct Snapshot

```bash
# Create an LVM snapshot of a Cinder volume
# Volume LV name format: volume-<uuid>
lvdisplay | grep cinder-volumes

lvcreate -L 1G -s -n volume-backup /dev/cinder-volumes/volume-<uuid>

# Copy the snapshot to backup storage
dd if=/dev/cinder-volumes/volume-backup bs=1M | gzip > /backup/cinder/volume-<uuid>.img.gz

# Remove the snapshot when done
lvremove -f /dev/cinder-volumes/volume-backup
```

### Ceph RBD Backend — Direct Export

```bash
# List volumes in the Ceph pool
rbd ls volumes

# Export a volume
rbd export volumes/volume-<uuid> /backup/cinder/volume-<uuid>.img

# Create a point-in-time snapshot before export
rbd snap create volumes/volume-<uuid>@backup-$(date +%Y%m%d)
rbd export volumes/volume-<uuid>@backup-$(date +%Y%m%d) /backup/cinder/volume-<uuid>.img
```

---

## Swift Object Storage Backup

### Ring Files — Critical

Swift rings define which nodes hold which objects. **If the rings are lost, access to all stored objects is lost, even if the raw data is intact.** Back up ring files after every ring change.

```bash
# Back up all ring files
cp /etc/swift/*.ring.gz /backup/swift-rings/
cp /etc/swift/*.builder /backup/swift-rings/  # builder files are needed for ring changes

# Create a dated archive
tar czf /backup/swift-rings-$(date +%Y%m%d).tar.gz \
  /etc/swift/*.ring.gz \
  /etc/swift/*.builder \
  /etc/swift/swift.conf

# Verify ring file integrity
swift-ring-builder /etc/swift/account.builder
swift-ring-builder /etc/swift/container.builder
swift-ring-builder /etc/swift/object.builder
```

### Object Data

Swift replicates objects across nodes (default 3 replicas). For off-cluster backup of critical data:

```bash
# Download all objects from a container
openstack object list <container-name> -f value -c Name | \
  while read obj; do
    openstack object save <container-name> "$obj"
  done

# Or use swift CLI for bulk download
swift download <container-name> --all -D /backup/swift-objects/
```

---

## Barbican Backup

Barbican stores secrets encrypted in the database. The database backup contains ciphertext only. To recover, both the database and the master KEK (Key Encryption Key) must be available.

```bash
# Back up Barbican database (same as above)
mysqldump --single-transaction --routines --triggers barbican > barbican-backup.sql

# Back up the master KEK — stored in barbican.conf
grep -A 5 '\[simple_crypto_plugin\]' /etc/barbican/barbican.conf
# The kek_of_the_day or fixed_crypto_kek value must be backed up securely

# Back up Barbican config (includes PKCS#11 or KMIP connection details if used)
cp /etc/barbican/barbican.conf /backup/barbican-barbican.conf.bak
```

Store the KEK backup separately from the database backup and the config backup — all three together would allow an attacker to read all secrets.

---

## Restore Procedures

Restore in this order to satisfy service dependencies.

### 1. Restore Databases

```bash
# For each service database:
mysql keystone < keystone-backup.sql
mysql nova < nova-backup.sql
mysql nova_api < nova_api-backup.sql
mysql nova_cell0 < nova_cell0-backup.sql
mysql neutron < neutron-backup.sql
mysql cinder < cinder-backup.sql
mysql glance < glance-backup.sql
mysql heat < heat-backup.sql
mysql barbican < barbican-backup.sql
mysql designate < designate-backup.sql
mysql ironic < ironic-backup.sql
mysql magnum < magnum-backup.sql
mysql octavia < octavia-backup.sql
mysql manila < manila-backup.sql
```

If the backup was created with `--add-drop-database`, it includes `DROP DATABASE IF EXISTS` statements. Review before running if the DB already has data.

### 2. Restore Configuration

```bash
# Restore all service configurations
tar xzf openstack-config-backup.tar.gz -C /

# Fix ownership
chown -R keystone:keystone /etc/keystone
chown -R nova:nova /etc/nova
chown -R neutron:neutron /etc/neutron
chown -R cinder:cinder /etc/cinder
chown -R glance:glance /etc/glance
```

### 3. Restore Keystone Keys

```bash
tar xzf fernet-keys-backup.tar.gz -C /

# Fix ownership and permissions
chown -R keystone:keystone /etc/keystone/fernet-keys /etc/keystone/credential-keys
chmod 600 /etc/keystone/fernet-keys/*
chmod 600 /etc/keystone/credential-keys/*
```

### 4. Start Keystone and Verify

```bash
systemctl start apache2  # or nginx
openstack token issue
openstack endpoint list
```

### 5. Start Glance and Restore Images

```bash
systemctl start glance-api

# If file backend: restore image files
rsync -avz /backup/glance-images/ /var/lib/glance/images/
chown -R glance:glance /var/lib/glance/images/

openstack image list
```

### 6. Start Cinder and Restore Volumes

```bash
systemctl start cinder-api cinder-scheduler cinder-volume cinder-backup

# If using Cinder backup service:
openstack volume backup list
openstack volume backup restore <backup-id>

openstack volume list
```

### 7. Start Neutron

```bash
systemctl start neutron-server
# Start agents on network/compute nodes
systemctl start neutron-openvswitch-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent

openstack network agent list
```

### 8. Start Nova

```bash
systemctl start nova-api nova-conductor nova-scheduler nova-novncproxy
# On each compute node:
systemctl start nova-compute

openstack compute service list
```

### 9. Start Remaining Services

Start Heat, Octavia, Designate, Barbican, Ironic, Magnum, Manila, and Placement in any order after Nova is healthy.

---

## Disaster Recovery — Full Cloud Recovery

Use this procedure when recovering the entire cloud from scratch (e.g., complete controller failure).

### Phase 1: Infrastructure Recovery

1. Provision or recover controller nodes (OS installation, networking).
2. Restore MariaDB Galera cluster:
   - Bootstrap the first node: `galera_new_cluster`
   - Join remaining nodes: `systemctl start mysql`
   - Verify cluster: `mysql -e "SHOW STATUS LIKE 'wsrep_cluster_size';"`
3. Restore RabbitMQ cluster:
   - Start first node, join others: `rabbitmqctl join_cluster rabbit@<first-node>`
   - Verify: `rabbitmqctl cluster_status`
4. Restore Memcached on each controller.

### Phase 2: Restore Databases

```bash
# After Galera is healthy, restore all service databases
for db in keystone nova nova_api nova_cell0 neutron cinder glance heat barbican designate ironic magnum octavia manila; do
  mysql "$db" < "/backup/${db}.sql"
  echo "Restored $db"
done
```

### Phase 3: Restore Configuration

```bash
tar xzf openstack-config-backup.tar.gz -C /
# Fix ownership as above
```

### Phase 4: Start Services in Order

Follow the service startup order:
1. Keystone (verify: `openstack token issue`)
2. Glance (verify: `openstack image list`)
3. Placement (`openstack resource provider list`)
4. Cinder (verify: `openstack volume list`)
5. Neutron (verify: `openstack network agent list`)
6. Nova control plane (verify: `openstack compute service list`)
7. Nova compute nodes
8. Remaining services

### Phase 5: Verify Compute Node Connectivity

```bash
# Verify compute nodes reconnected to the message bus
openstack compute service list

# Verify network agents on compute nodes
openstack network agent list

# Verify hypervisors are reporting correctly
openstack hypervisor list
```

### Phase 6: Instance State Recovery

Instances running on compute nodes during a controller failure are typically still running. After controller recovery:

```bash
# Verify instance states match actual running states on compute nodes
openstack server list --all-projects

# For instances stuck in 'error' or incorrect state after controller recovery:
# Reset state to allow operations (use with care — verify the actual state on the hypervisor first)
openstack server set --state active <instance-id>
```

---

## Backup Schedule Recommendations

| Component | Recommended Frequency |
|---|---|
| All service databases | Daily (before business hours), retained for 30 days |
| Service configurations | After any config change; weekly at minimum |
| Keystone fernet and credential keys | After every rotation; rotation should be daily or weekly |
| Glance images | Weekly for full backup; incremental on file changes |
| Swift ring files | After any ring change (topology change, node add/remove) |
| Cinder volume backups | Per-volume, per-tenant SLA; snapshots before major changes |
| Barbican master KEK | Offline copy; change only triggers re-encryption of all secrets |

Store backups off-node. A backup stored only on the same node being backed up provides no protection against node failure.
