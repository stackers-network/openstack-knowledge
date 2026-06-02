# Diagnose

Systematic troubleshooting for OpenStack issues. Given a symptom, investigate to find the root cause.

## Diagnostic Process

### 1. Gather Context

```bash
# What was the last action taken?
git log --oneline -5

# What's the current state?
source /etc/kolla/admin-openrc.sh
openstack compute service list
openstack network agent list
```

### 2. Check Service Health

```bash
# Are all services reporting in?
openstack endpoint list --interface internal
openstack compute service list
openstack network agent list
openstack volume service list

# Any containers down?
ssh <host> docker ps --filter "label=kolla_service" --format "{{.Names}}: {{.Status}}"
ssh <host> docker ps --filter "label=kolla_service" --filter "status=restarting"
```

### 3. Read Logs

Most service logs are in `/var/log/kolla/<service>/`:

```bash
# Find the log volume
ssh <host> readlink -f /var/log/kolla
# Usually: /var/lib/docker/volumes/kolla_logs/_data

# Read logs for a specific service
ssh <host> tail -100 /var/log/kolla/<service>/<service>.log | grep -i error

# Container stdout (most containers don't log here, but some do)
ssh <host> docker logs --tail 100 kolla_<service>
```

### 4. Check Configuration

```bash
# What config is the service actually running with?
ssh <host> docker exec kolla_<service> cat /etc/<service>/<config_file>

# Generate configs without deploying (useful for comparing)
kolla-ansible genconfig -i inventory/hosts.yml
```

### 5. Check Infrastructure

```bash
# MariaDB cluster
ssh <control> docker exec mariadb mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
ssh <control> docker exec mariadb mysql -u root -e "SHOW STATUS LIKE 'wsrep_ready'"

# RabbitMQ cluster
ssh <control> docker exec rabbitmq rabbitmqctl cluster_status

# OVN (if using OVN)
ssh <control> docker exec ovn_northd ovn-nbctl show
```

## Common Issues and Solutions

### Service Won't Start

1. Check container logs: `docker logs kolla_<service>`
2. Check config for invalid options: `docker exec kolla_<service> cat /etc/<service>/<config>.conf`
3. Check dependencies: is the database reachable? is RabbitMQ up?
4. Check ports: `ss -tlnp | grep <port>` — is another process binding?
5. Try restarting just that container: `docker restart kolla_<service>`

### API Returns 500

1. Check the specific service logs (not the proxy/haproxy)
2. Look for database connection errors — MariaDB might be down
3. Check Keystone token validity — token may be expired or Keystone may be down
4. Check RabbitMQ connectivity — service may not be able to reach the message queue

### VMs Can't Reach Network

1. Check OVN/OVS agent status: `openstack network agent list`
2. Check bridge configuration: `ovs-vsctl show` on the host
3. Check security groups: default group may block traffic
4. Check floating IP association and external network setup
5. Check that `neutron_external_interface` is UP and has no IP

### Deploy or Reconfigure Failed

1. **Read the Ansible error** — the failed task name tells you which service and which step
2. Check host reachability: `ansible -i inventory/hosts.yml all -m ping`
3. Check disk space: `df -h` on failed host
4. Check Docker status: `systemctl status docker` on failed host
5. Try running just the failed service: `kolla-ansible deploy -i inventory/hosts.yml --tags <service>`
6. **Do NOT run destroy** — fix the issue and re-run deploy

### MariaDB / Galera Issues

1. Check cluster status: `SHOW STATUS LIKE 'wsrep%'`
2. `wsrep_cluster_size` should equal number of control nodes
3. `wsrep_ready` should be ON
4. If split-brain: identify the node with most recent data, bootstrap from it
5. **Never force bootstrap** unless you understand the cluster state
6. Recovery: `kolla-ansible mariadb-recovery -i inventory/hosts.yml`

### RabbitMQ Issues

1. Check cluster status: `rabbitmqctl cluster_status`
2. Check for network partitions in the output
3. Check queue depths: `rabbitmqctl list_queues name messages consumers`
4. Growing queues with 0 consumers = service consuming the queue is down
5. If cluster is broken: may need to reset one node and rejoin

### HAProxy / VIP Issues

1. Check keepalived: `docker logs kolla_keepalived`
2. Check VIP is assigned: `ip addr show` on control nodes — one should have the VIP
3. Check HAProxy backends: `curl -s http://<vip>:1984/stats` (HAProxy stats if enabled)
4. If VIP is floating between nodes: check for network issues between control nodes

### Certificate / TLS Issues

1. Check certificate expiry: `openssl s_client -connect <vip>:5000 -servername <vip> </dev/null 2>/dev/null | openssl x509 -noout -dates`
2. Check CA trust: `curl -v https://<vip>:5000/v3` — look for certificate errors
3. Regenerate certificates: `kolla-ansible certificates -i inventory/hosts.yml`

## Safety Rules

- **Never take remediation action without operator approval** — present findings and recommended fix, let them decide
- **Log all diagnostic steps** — if the fix doesn't work, you need to know what you already tried
- **If uncertain about root cause, say so** — don't guess and don't apply random fixes
- **Don't restart services during a deploy or upgrade** — let the operation complete or fail first
