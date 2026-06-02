# Diagnose — Infrastructure Troubleshooting

Troubleshooting decision trees for infrastructure-level issues: slow APIs, service startup failures, RabbitMQ, and database problems. For instance, network, API auth, and volume issues, see the [main diagnose skill](README.md).

**Before any diagnosis**: confirm which release is running (`openstack versions show` or `openstack --version`) and collect the output of `openstack endpoint list` so you know the service topology.

---

## Slow API Responses

**Symptom**: API calls take several seconds or time out. Performance has degraded from baseline.

### Step 1 — Check database query performance

```bash
# Check for slow queries and active connections
mysqladmin -u root -p processlist
mysql -u root -p -e "SHOW FULL PROCESSLIST;"

# Check slow query log
mysql -u root -p -e "SHOW VARIABLES LIKE 'slow_query_log%';"
mysql -u root -p -e "SHOW VARIABLES LIKE 'long_query_time';"

# Check for table locks
mysql -u root -p -e "SHOW OPEN TABLES WHERE In_use > 0;"

# Check InnoDB status for deadlocks
mysql -u root -p -e "SHOW ENGINE INNODB STATUS\G" | grep -A 30 "LATEST DETECTED DEADLOCK"
```

Enable the slow query log if not already on:

```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```

### Step 2 — Check RabbitMQ queue depth

```bash
# List all queues with depth and consumer count
rabbitmqctl list_queues name messages consumers message_bytes

# Filter for deep queues
rabbitmqctl list_queues name messages consumers | awk '$2 > 100'

# Check if workers are consuming
rabbitmqctl list_queues name messages consumers | grep "^nova\." | sort -k2 -rn | head -20

# Check connections
rabbitmqctl list_connections name state | head -30

# Memory and disk alarms
rabbitmqctl status | grep -E "memory|disk|alarm"
```

Queue depth growing without consumers means the workers for that service are not running or are crashing.

### Step 3 — Check service worker count

```bash
# Nova API workers
grep "api_workers\|osapi_compute_workers\|metadata_workers" /etc/nova/nova.conf

# Neutron API workers
grep "api_workers" /etc/neutron/neutron.conf

# Check actual running processes
ps aux | grep nova-api | grep -v grep | wc -l
ps aux | grep neutron-server | grep -v grep | wc -l

# Check worker memory usage
ps aux | grep nova-api | sort -k6 -rn | head -5
```

Increase workers if fewer than `(CPU_COUNT / 2)`:

```ini
# /etc/nova/nova.conf
[DEFAULT]
osapi_compute_workers = 8
metadata_workers = 4
```

### Step 4 — Check Keystone token validation cache

Every API call validates a token against Keystone. Poor caching multiplies the load:

```bash
# Check if caching is enabled
grep -A 10 '\[cache\]' /etc/nova/nova.conf
grep -A 10 '\[keystone_authtoken\]' /etc/nova/nova.conf | grep "memcache\|cache"

# Check memcached
systemctl status memcached
echo "stats" | nc localhost 11211 | grep -E "curr_connections|cmd_get|get_hits|get_misses"

# Hit rate (should be > 80%)
echo "stats" | nc localhost 11211 | awk '/get_hits/{h=$2} /get_misses/{m=$2} END{print "hit_rate:", h/(h+m)*100"%"}'
```

Enable memcached-backed token cache:

```ini
# /etc/<service>/<service>.conf
[keystone_authtoken]
memcached_servers = localhost:11211
token_cache_time = 300
```

### Step 5 — Check service and OS metrics

```bash
# CPU saturation
top -b -n1 | head -20
vmstat 1 5

# Disk I/O
iostat -x 1 5

# Network
netstat -s | grep -i "retransmit\|error"
ss -s
```

### Common Resolutions

| Cause | Resolution |
|---|---|
| Slow DB queries | Add indexes, tune `innodb_buffer_pool_size`, kill long-running queries |
| RabbitMQ queue backlog | Scale up service workers, investigate crashing consumers |
| Too few API workers | Increase `api_workers` in service config, restart |
| No token cache | Enable memcached-backed token cache |
| Disk I/O bottleneck | Move DB to SSD, tune `innodb_flush_method` |
| Network congestion | Check MTU, check for packet loss between nodes |

---

## Service Won't Start

**Symptom**: A service fails to start or crashes immediately after starting.

### Step 1 — Check configuration syntax

```bash
# Nova
nova-manage config validate

# Cinder
cinder-manage config validate

# For services without config validate — try a dry run
nova-api --config-file /etc/nova/nova.conf --help > /dev/null 2>&1; echo $?

# Check for unknown config options (oslo.config)
grep -i "unknown option\|unrecognized section" /var/log/<service>/<service>.log | head -20
```

### Step 2 — Check database connectivity

```bash
# Verify credentials in service config
grep "^connection" /etc/<service>/<service>.conf 2>/dev/null || \
  grep "connection" /etc/<service>/<service>.conf | grep -v "^#"

# Test connection
mysql -u <USER> -p<PASS> -h <HOST> <DB> -e "SELECT 1"

# Check MySQL is running and accessible
systemctl status mysql
systemctl status mariadb
ss -tlnp | grep 3306
```

### Step 3 — Check database migration version

```bash
nova-manage db version
nova-manage api_db version
neutron-db-manage current
cinder-manage db version
glance-manage db_version
keystone-manage db_version
```

If the DB schema is ahead of the code (downgrade scenario) or behind (migration not run), the service will refuse to start.

```bash
# Run pending migrations (always backup DB first)
mysqldump -u root -p <SERVICE_DB> > /tmp/<service>-db-backup-$(date +%Y%m%d).sql
nova-manage db sync
nova-manage api_db sync
neutron-db-manage upgrade heads
cinder-manage db sync
glance-manage db_sync
```

### Step 4 — Check log file permissions

```bash
# Check log directory ownership
ls -la /var/log/<service>/

# Fix ownership
chown -R <service_user>:<service_group> /var/log/<service>/

# Check /run directory for PID files
ls -la /var/run/<service>/ 2>/dev/null || ls -la /run/<service>/ 2>/dev/null
chown -R <service_user>:<service_group> /run/<service>/
```

### Step 5 — Check port conflicts

```bash
# Nova API default: 8774, Neutron: 9696, Cinder: 8776, Glance: 9292, Keystone: 5000
ss -tlnp | grep "8774\|9696\|8776\|9292\|5000"

# Find what is using a port
ss -tlnp sport = :8774
fuser 8774/tcp
```

### Step 6 — Read the service log directly

```bash
# Start the service manually in foreground for immediate output
sudo -u nova nova-api --config-file /etc/nova/nova.conf 2>&1 | head -50
sudo -u neutron neutron-server --config-file /etc/neutron/neutron.conf 2>&1 | head -50

# Or read the log right after attempted start
journalctl -u nova-api -n 100 --no-pager
```

### Common Resolutions

| Cause | Resolution |
|---|---|
| Config syntax error | Fix the offending option (check log for the option name) |
| DB connection refused | Start MySQL/MariaDB, fix connection string |
| DB migration mismatch | Run `<service>-manage db sync` |
| Port already in use | Kill conflicting process or change service port |
| Log dir permissions | `chown -R <user>:<group> /var/log/<service>/` |
| Missing Python package | Install missing dependency (`pip install <package>`) |

---

## RabbitMQ Issues

**Symptom**: Services report connection errors, timeouts, or queue-related failures. `journalctl` shows `AMQP connection error` or `ConnectionClosed`.

### Step 1 — Check RabbitMQ status

```bash
rabbitmqctl status
# Look for: running applications, Erlang version, memory, disk, alarms

rabbitmqctl cluster_status
# Check all nodes are running, no network partitions
# Look for: [{running_nodes,...}] and [{partitions,[]}]  -- partitions must be empty
```

### Step 2 — Check memory and disk alarms

```bash
rabbitmqctl status | grep -A 10 "memory\|disk\|alarm"

# Memory alarm blocks all publishers when vm_memory_high_watermark is exceeded
# Disk alarm blocks all publishers when free disk < disk_free_limit

# Current usage
rabbitmqctl status | grep "memory,"
df -h /var/lib/rabbitmq
```

Resolve memory alarm:

```bash
# Temporarily raise the watermark (edit /etc/rabbitmq/rabbitmq.conf for permanent)
rabbitmqctl set_vm_memory_high_watermark 0.6

# Or purge old messages from dead queues
rabbitmqctl list_queues name messages | awk '$2 > 10000 {print $1}' | \
  xargs -I{} rabbitmqctl purge_queue {}
```

### Step 3 — Check queue depth and consumers

```bash
# Full queue listing
rabbitmqctl list_queues name messages consumers durable auto_delete

# Queues with messages and no consumers (orphaned)
rabbitmqctl list_queues name messages consumers | awk '$3 == 0 && $2 > 0'

# Connection count per service
rabbitmqctl list_connections client_properties state | head -40
```

### Step 4 — Check for network partitions

```bash
rabbitmqctl cluster_status | grep partition
# Output must show: {partitions,[]}
# If it shows nodes, there is an active partition
```

Recover from a split-brain partition (choose the side that has the most current data):

```bash
# Stop RabbitMQ on the minority side
rabbitmqctl stop_app
# Reset it
rabbitmqctl reset
# Re-join the cluster
rabbitmqctl join_cluster rabbit@<MAJORITY_NODE>
# Start
rabbitmqctl start_app
```

### Step 5 — Check OpenStack service connection config

```bash
grep "transport_url" /etc/nova/nova.conf
grep "transport_url" /etc/neutron/neutron.conf
grep "transport_url" /etc/cinder/cinder.conf
# Format: rabbit://<user>:<password>@<host>:5672/<vhost>

# Test connectivity from service host
nc -zv <RABBITMQ_HOST> 5672

# Verify the OpenStack vhost exists
rabbitmqctl list_vhosts
rabbitmqctl list_permissions -p /openstack
```

### Common Resolutions

| Cause | Resolution |
|---|---|
| Memory alarm | Reduce memory use, purge dead queues, increase watermark |
| Disk alarm | Free disk space on RabbitMQ node, purge old messages |
| Network partition | Reset minority node, re-join cluster |
| Wrong vhost/credentials | Fix `transport_url` in service config |
| Too many connections | Tune `vm_memory_high_watermark`, add connection pooling |
| Erlang cookie mismatch | Ensure all cluster nodes share the same `.erlang.cookie` |

---

## Database Issues

**Symptom**: Services log `OperationalError`, `Lost connection to MySQL`, connection pool errors, or very slow queries.

### Step 1 — Check connection pool and active connections

```bash
# Show all active connections
mysql -u root -p -e "SHOW PROCESSLIST;"
mysql -u root -p -e "SHOW FULL PROCESSLIST;"

# Count connections by user/DB
mysql -u root -p -e "
  SELECT user, db, COUNT(*) AS connections, state
  FROM information_schema.processlist
  GROUP BY user, db, state
  ORDER BY connections DESC;"

# Current and max connections
mysql -u root -p -e "SHOW STATUS LIKE 'Threads_connected';"
mysql -u root -p -e "SHOW VARIABLES LIKE 'max_connections';"

# Check connection errors
mysql -u root -p -e "SHOW STATUS LIKE 'Connection_errors%';"
```

### Step 2 — Identify slow queries

```bash
# Active slow queries
mysql -u root -p -e "SHOW FULL PROCESSLIST;" | awk '$6 > 5'  # queries running > 5s

# Kill a long-running query
mysql -u root -p -e "KILL QUERY <PROCESS_ID>;"
mysql -u root -p -e "KILL <PROCESS_ID>;"  # kill the connection

# Enable slow query log temporarily
mysql -u root -p -e "SET GLOBAL slow_query_log = 'ON';"
mysql -u root -p -e "SET GLOBAL long_query_time = 2;"
tail -f /var/log/mysql/mysql-slow.log
```

### Step 3 — Check for table locks

```bash
mysql -u root -p -e "SHOW OPEN TABLES WHERE In_use > 0;"
mysql -u root -p -e "SHOW ENGINE INNODB STATUS\G" | grep -A 50 "TRANSACTIONS"
mysql -u root -p -e "SELECT * FROM information_schema.INNODB_TRX\G"
```

### Step 4 — Check replication lag (HA deployments)

```bash
# On the replica
mysql -u root -p -e "SHOW SLAVE STATUS\G"
# Check: Seconds_Behind_Master, Slave_IO_Running, Slave_SQL_Running
# Slave_IO_Running and Slave_SQL_Running must both be 'Yes'
# Seconds_Behind_Master should be < 30

# Galera cluster status
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep%';"
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"
# wsrep_local_state_comment should be 'Synced'
```

### Step 5 — Check InnoDB buffer pool usage

```bash
mysql -u root -p -e "SHOW STATUS LIKE 'Innodb_buffer_pool%';"
# Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests gives disk read ratio
# Should be < 1%

# Current buffer pool size
mysql -u root -p -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size';"
# Typically set to 70-80% of available RAM on dedicated DB servers
```

### Step 6 — Check service-side connection pool config

```bash
# Nova DB pool settings
grep -E "max_pool_size|max_overflow|pool_timeout|connection_recycle_time" /etc/nova/nova.conf

# Typical tuning
# [database]
# max_pool_size = 20
# max_overflow = 10
# pool_timeout = 30
# connection_recycle_time = 3600
```

### Common Resolutions

| Cause | Resolution |
|---|---|
| Too many connections | Increase `max_connections` in my.cnf, tune service pool size |
| Slow queries | Add missing indexes, kill runaway queries, tune buffer pool |
| Table locks / deadlocks | Identify and kill blocking transactions, review application retry logic |
| Galera node desynced | Restart the desynced node, allow SST/IST to resync |
| Replication lag | Reduce write load on primary, check replica I/O |
| Connection pool exhausted | Increase `max_pool_size`, reduce `max_overflow` |
