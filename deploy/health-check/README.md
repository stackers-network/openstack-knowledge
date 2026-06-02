# Health Check

How to validate that an OpenStack deployment is healthy.

## When to Check

- After any deploy or reconfigure
- Before an upgrade (establish baseline)
- When an operator asks "is it working?"
- On a regular schedule

## Quick Health Check

```bash
source /etc/kolla/admin-openrc.sh

# 1. Can we authenticate?
openstack token issue

# 2. Are all compute services up?
openstack compute service list
# All should show State: up, Status: enabled

# 3. Are all network agents up?
openstack network agent list
# All should show Alive: :-)

# 4. Are all volume services up?
openstack volume service list
# All should show State: up, Status: enabled

# 5. Are all endpoints registered?
openstack endpoint list --interface internal
```

## Container Health

Check on each host:

```bash
# All kolla containers running?
ssh <host> docker ps --filter "label=kolla_service" --format "{{.Names}}: {{.Status}}"

# Any restarting containers? (bad sign)
ssh <host> docker ps --filter "label=kolla_service" --filter "status=restarting"

# Any exited containers? (may be fine — some are one-shot)
ssh <host> docker ps -a --filter "label=kolla_service" --filter "status=exited" --format "{{.Names}}: {{.Status}}"
```

## API Endpoint Checks

```bash
# Keystone (identity)
curl -sk https://<internal_vip>:5000/v3

# Nova (compute)
curl -sk https://<internal_vip>:8774/v2.1

# Neutron (network)
curl -sk https://<internal_vip>:9696/v2.0

# Glance (image)
curl -sk https://<internal_vip>:9292/v2/images

# Cinder (block storage)
curl -sk https://<internal_vip>:8776/v3
```

If TLS is not enabled, use `http://` and the appropriate ports.

## Network Validation

```bash
# Check OVN (if using OVN)
ssh <control_host> docker exec ovn_northd ovn-nbctl show
ssh <control_host> docker exec ovn_northd ovn-sbctl show

# Check Open vSwitch (if using OVS)
ssh <network_host> docker exec openvswitch_vswitchd ovs-vsctl show
```

## Database Health (MariaDB Galera)

```bash
ssh <control_host> docker exec mariadb mysql -u root -p$(grep database_password /etc/kolla/passwords.yml | awk '{print $2}') \
  -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
# Should show number equal to control node count

ssh <control_host> docker exec mariadb mysql -u root -p<password> \
  -e "SHOW STATUS LIKE 'wsrep_ready'"
# Should show ON
```

## Message Queue Health (RabbitMQ)

```bash
ssh <control_host> docker exec rabbitmq rabbitmqctl cluster_status
# Should show all control nodes in the cluster

ssh <control_host> docker exec rabbitmq rabbitmqctl list_queues name messages consumers | head -20
# Watch for queues with growing message count and 0 consumers
```

## Smoke Test (Full Functional)

```bash
source /etc/kolla/admin-openrc.sh

# Create a test network
openstack network create test-net
openstack subnet create test-subnet --network test-net --subnet-range 192.168.100.0/24

# Create a test instance
openstack server create --flavor m1.small --image cirros --network test-net test-vm

# Wait for it to become ACTIVE
openstack server show test-vm -f value -c status
# Should show ACTIVE

# Clean up
openstack server delete test-vm
openstack subnet delete test-subnet
openstack network delete test-net
```

## Health Report Format

When reporting health, use this structure:

```
Status: HEALTHY / DEGRADED / UNHEALTHY

Services:
  Keystone:     OK
  Nova:         OK (3/3 compute agents)
  Neutron:      OK (5/5 agents)
  Cinder:       OK
  Glance:       OK

Containers:
  Total:        47
  Running:      47
  Unhealthy:    0

Issues:
  (none)
```

## Severity Levels

- **HEALTHY** — all checks pass, all services running
- **DEGRADED** — some non-critical services down but cloud is functional (e.g., one compute agent down out of 4)
- **UNHEALTHY** — critical services down, cloud is non-functional (e.g., Keystone down, all compute down)
