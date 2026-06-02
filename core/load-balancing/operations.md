# Octavia Operations

## Prerequisites

Confirm the Octavia endpoint is reachable and the amphora image is registered:

```bash
openstack loadbalancer provider list
openstack image list --tag amphora
```

All `openstack loadbalancer` commands require the `python-openstackclient` package with the `python-octaviaclient` plugin. Operations that change state (create, update, delete, failover) are asynchronous — poll `provisioning_status` until `ACTIVE`.

## Load Balancers

### Create

```bash
# Minimal: specify a subnet for the VIP
openstack loadbalancer create \
  --name web-lb \
  --vip-subnet-id private-subnet

# Specify a fixed VIP address on the subnet
openstack loadbalancer create \
  --name web-lb \
  --vip-subnet-id private-subnet \
  --vip-address 192.168.1.100

# Use an HA topology via a flavor
openstack loadbalancer create \
  --name web-lb-ha \
  --vip-subnet-id private-subnet \
  --flavor ha-medium

# Use the OVN provider for lightweight internal LB
openstack loadbalancer create \
  --name internal-lb \
  --vip-subnet-id private-subnet \
  --provider ovn
```

### List and Show

```bash
openstack loadbalancer list
openstack loadbalancer list --name web-lb
openstack loadbalancer show web-lb

# Watch until ACTIVE (poll every 5 s)
watch -n 5 openstack loadbalancer show web-lb
```

### Update

```bash
# Rename
openstack loadbalancer set web-lb --name prod-lb

# Add a description
openstack loadbalancer set prod-lb --description "Production web tier LB"
```

### Delete

```bash
# Delete LB and all its child resources (listeners, pools, members)
openstack loadbalancer delete web-lb --cascade

# Without --cascade, delete fails if child resources exist
openstack loadbalancer delete web-lb
```

### Assign a Floating IP to the VIP

```bash
# Get the Neutron port ID of the VIP
VIP_PORT=$(openstack loadbalancer show web-lb -f value -c vip_port_id)

# Allocate a floating IP
FIP=$(openstack floating ip create provider-net -f value -c floating_ip_address)

# Associate with the VIP port
openstack floating ip set --port $VIP_PORT $FIP

echo "Load balancer reachable at $FIP"
```

## Listeners

Listeners define the front-end protocol and port that the load balancer accepts traffic on.

### Create

```bash
# HTTP listener on port 80
openstack loadbalancer listener create \
  --name http-listener \
  --protocol HTTP \
  --protocol-port 80 \
  web-lb

# HTTPS passthrough (TCP) on port 443 — back-end handles TLS
openstack loadbalancer listener create \
  --name https-passthrough \
  --protocol TCP \
  --protocol-port 443 \
  web-lb

# TLS termination at the load balancer (Barbican secret required)
openstack loadbalancer listener create \
  --name https-listener \
  --protocol TERMINATED_HTTPS \
  --protocol-port 443 \
  --default-tls-container-ref $(openstack secret list --name web-cert -f value -c "Secret href") \
  web-lb

# UDP listener (e.g., for DNS)
openstack loadbalancer listener create \
  --name dns-listener \
  --protocol UDP \
  --protocol-port 53 \
  dns-lb

# Set connection limit (-1 = unlimited)
openstack loadbalancer listener create \
  --name http-listener \
  --protocol HTTP \
  --protocol-port 80 \
  --connection-limit 10000 \
  web-lb
```

### List and Show

```bash
openstack loadbalancer listener list
openstack loadbalancer listener list --loadbalancer web-lb
openstack loadbalancer listener show http-listener
```

### Update and Delete

```bash
openstack loadbalancer listener set http-listener --connection-limit 5000
openstack loadbalancer listener delete http-listener
```

## Pools

Pools hold the back-end members and define the load-balancing algorithm.

### Create

```bash
# Attach directly to a listener
openstack loadbalancer pool create \
  --name web-pool \
  --protocol HTTP \
  --lb-algorithm ROUND_ROBIN \
  --listener http-listener

# Attach to the load balancer (not a listener) — for use as a default pool
openstack loadbalancer pool create \
  --name generic-pool \
  --protocol TCP \
  --lb-algorithm LEAST_CONNECTIONS \
  --loadbalancer web-lb

# Session persistence: SOURCE_IP
openstack loadbalancer pool create \
  --name sticky-pool \
  --protocol HTTP \
  --lb-algorithm ROUND_ROBIN \
  --session-persistence type=SOURCE_IP \
  --listener http-listener

# Session persistence: HTTP cookie
openstack loadbalancer pool create \
  --name cookie-pool \
  --protocol HTTP \
  --lb-algorithm ROUND_ROBIN \
  --session-persistence type=HTTP_COOKIE,cookie_name=SERVERID \
  --listener http-listener
```

Supported algorithms:

| Algorithm | Behaviour |
|---|---|
| `ROUND_ROBIN` | Requests distributed in turn |
| `LEAST_CONNECTIONS` | Requests sent to member with fewest active connections |
| `SOURCE_IP` | Requests from same client IP always go to same member |
| `SOURCE_IP_PORT` | Requests from same client IP+port always go to same member |

### List, Show, Update, Delete

```bash
openstack loadbalancer pool list
openstack loadbalancer pool show web-pool
openstack loadbalancer pool set web-pool --lb-algorithm LEAST_CONNECTIONS
openstack loadbalancer pool delete web-pool
```

## Members

Members are the back-end servers that receive traffic from the pool.

### Add Members

```bash
# Add a member by IP address and port
openstack loadbalancer member create \
  --name web-1 \
  --address 192.168.1.10 \
  --protocol-port 8080 \
  web-pool

openstack loadbalancer member create \
  --name web-2 \
  --address 192.168.1.11 \
  --protocol-port 8080 \
  web-pool

# Add with a custom weight (higher = more traffic)
openstack loadbalancer member create \
  --name web-heavy \
  --address 192.168.1.12 \
  --protocol-port 8080 \
  --weight 5 \
  web-pool

# Add a backup member (only used when all primary members are down)
openstack loadbalancer member create \
  --name web-backup \
  --address 192.168.1.20 \
  --protocol-port 8080 \
  --backup \
  web-pool

# Specify the subnet for members on a different subnet than the VIP
openstack loadbalancer member create \
  --name web-3 \
  --address 10.0.2.15 \
  --protocol-port 8080 \
  --subnet-id backend-subnet \
  web-pool
```

### Batch Update (replace entire member list)

```bash
# Replace all members in one API call (PUT /lbaas/pools/{pool_id}/members)
openstack loadbalancer member batch update web-pool \
  --members '[{"address":"192.168.1.10","protocol_port":8080},{"address":"192.168.1.11","protocol_port":8080}]'
```

### List, Show, Update, Delete

```bash
openstack loadbalancer member list web-pool
openstack loadbalancer member show web-pool web-1
openstack loadbalancer member set web-pool web-1 --weight 2
openstack loadbalancer member delete web-pool web-1
```

### Drain a Member

Set weight to 0 to stop sending new connections while existing ones complete:

```bash
openstack loadbalancer member set web-pool web-1 --weight 0
# Wait for connections to drain, then remove
openstack loadbalancer member delete web-pool web-1
```

## Health Monitors

Health monitors probe back-end members and mark them `ONLINE` or `ERROR` based on responses.

### Create

```bash
# HTTP health check — GET /healthz, expect 200
openstack loadbalancer healthmonitor create \
  --name hm-http \
  --type HTTP \
  --delay 5 \
  --timeout 3 \
  --max-retries 3 \
  --url-path /healthz \
  --expected-codes 200 \
  web-pool

# HTTPS health check
openstack loadbalancer healthmonitor create \
  --name hm-https \
  --type HTTPS \
  --delay 5 \
  --timeout 3 \
  --max-retries 3 \
  --url-path /health \
  --expected-codes 200,204 \
  web-pool

# TCP health check (just checks TCP connect)
openstack loadbalancer healthmonitor create \
  --name hm-tcp \
  --type TCP \
  --delay 10 \
  --timeout 5 \
  --max-retries 3 \
  web-pool

# UDP-CONNECT health check
openstack loadbalancer healthmonitor create \
  --name hm-udp \
  --type UDP-CONNECT \
  --delay 5 \
  --timeout 3 \
  --max-retries 3 \
  dns-pool

# Set max-retries-down separately (failures before marking ERROR)
openstack loadbalancer healthmonitor create \
  --name hm-strict \
  --type HTTP \
  --delay 5 \
  --timeout 3 \
  --max-retries 3 \
  --max-retries-down 2 \
  --url-path / \
  --expected-codes 200 \
  web-pool
```

Health monitor parameters:

| Parameter | Meaning |
|---|---|
| `--delay` | Seconds between probes |
| `--timeout` | Seconds to wait for a probe response |
| `--max-retries` | Consecutive successes to mark member `ONLINE` |
| `--max-retries-down` | Consecutive failures to mark member `ERROR` (default = `--max-retries`) |

### List, Show, Update, Delete

```bash
openstack loadbalancer healthmonitor list
openstack loadbalancer healthmonitor show hm-http
openstack loadbalancer healthmonitor set hm-http --delay 10
openstack loadbalancer healthmonitor delete hm-http
```

## L7 Policies and Rules

L7 policies allow content-based routing based on HTTP headers, URLs, or cookies.

### Create an L7 Policy

```bash
# Redirect /api/** to a different pool
openstack loadbalancer l7policy create \
  --name api-policy \
  --listener http-listener \
  --action REDIRECT_TO_POOL \
  --redirect-pool api-pool \
  --position 1

# Redirect a path to a URL
openstack loadbalancer l7policy create \
  --name old-path-redirect \
  --listener http-listener \
  --action REDIRECT_TO_URL \
  --redirect-url https://new.example.com/ \
  --position 2

# Reject requests matching a rule
openstack loadbalancer l7policy create \
  --name block-policy \
  --listener http-listener \
  --action REJECT \
  --position 3
```

### Create L7 Rules

Each L7 policy requires one or more rules. All rules in a policy must match (AND logic) for the policy to trigger.

```bash
# Match requests where path starts with /api
openstack loadbalancer l7rule create \
  --compare-type STARTS_WITH \
  --type PATH \
  --value /api \
  api-policy

# Match on HTTP header
openstack loadbalancer l7rule create \
  --compare-type EQUAL_TO \
  --type HEADER \
  --key X-API-Version \
  --value v2 \
  api-policy

# Match on cookie value
openstack loadbalancer l7rule create \
  --compare-type EQUAL_TO \
  --type COOKIE \
  --key canary \
  --value true \
  canary-policy

# Invert a rule (NOT match)
openstack loadbalancer l7rule create \
  --compare-type STARTS_WITH \
  --type PATH \
  --value /public \
  --invert \
  private-policy
```

L7 rule types: `HOST_NAME`, `PATH`, `FILE_TYPE`, `HEADER`, `COOKIE`
L7 compare types: `REGEX`, `STARTS_WITH`, `ENDS_WITH`, `CONTAINS`, `EQUAL_TO`

### List and Delete L7 Resources

```bash
openstack loadbalancer l7policy list --listener http-listener
openstack loadbalancer l7policy show api-policy
openstack loadbalancer l7rule list api-policy
openstack loadbalancer l7policy delete api-policy
```

## Stats and Status

### Load Balancer Stats

```bash
# Traffic statistics for the load balancer
openstack loadbalancer stats show web-lb

# Statistics for a specific listener
openstack loadbalancer listener stats show http-listener

# Statistics for a pool
openstack loadbalancer pool stats show web-pool
```

Stats fields returned:

| Field | Meaning |
|---|---|
| `active_connections` | Current open connections |
| `bytes_in` | Total bytes received from clients |
| `bytes_out` | Total bytes sent to clients |
| `request_errors` | HTTP requests that resulted in errors |
| `total_connections` | Total connections since last reset |

### Operational Status

```bash
# Check all resources in one command (shows tree)
openstack loadbalancer status show web-lb
```

## Amphora Failover

Trigger a manual failover (replaces the amphora VM with a new one):

```bash
# Fail over the amphora backing a specific load balancer
openstack loadbalancer failover web-lb

# List amphorae to find specific IDs
openstack loadbalancer amphora list
openstack loadbalancer amphora show <amphora-id>

# Fail over a specific amphora by ID
openstack loadbalancer amphora failover <amphora-id>
```

Failover workflow: Octavia boots a new amphora, configures it identically to the failed one, then deletes the old VM. During failover the load balancer's `provisioning_status` is `PENDING_UPDATE`.

## Quotas

```bash
# Show quotas for the current project
openstack loadbalancer quota show

# Show quotas for a specific project (admin)
openstack loadbalancer quota show --project my-project

# Set quotas (admin)
openstack loadbalancer quota set \
  --loadbalancer 10 \
  --listener 50 \
  --pool 50 \
  --member 200 \
  --healthmonitor 50 \
  my-project

# Reset to defaults
openstack loadbalancer quota delete my-project
```

## Flavors

```bash
# List available load balancer flavors
openstack loadbalancer flavor list

# Show flavor details
openstack loadbalancer flavor show large

# Create a flavor profile (admin)
openstack loadbalancer flavorprofile create \
  --name ha-large-profile \
  --provider amphora \
  --flavor-data '{"compute_flavor": "m1.xlarge", "loadbalancer_topology": "ACTIVE_STANDBY"}'

# Create a flavor (admin)
openstack loadbalancer flavor create \
  --name ha-large \
  --flavorprofile ha-large-profile \
  --description "HA load balancer on xlarge compute"

# Use the flavor
openstack loadbalancer create \
  --name prod-lb \
  --vip-subnet-id private-subnet \
  --flavor ha-large
```

## Availability Zones

```bash
# List availability zones
openstack loadbalancer availabilityzone list

# Create an availability zone profile (admin)
openstack loadbalancer availabilityzoneprofile create \
  --name az1-profile \
  --provider amphora \
  --availability-zone-data '{"compute_zone": "nova-az1"}'

# Create an availability zone (admin)
openstack loadbalancer availabilityzone create \
  --name az1 \
  --availabilityzoneprofile az1-profile

# Pin a load balancer to an availability zone
openstack loadbalancer create \
  --name az1-lb \
  --vip-subnet-id private-subnet \
  --availability-zone az1
```

## Common Troubleshooting

```bash
# Check overall service health
openstack loadbalancer amphora list

# Find amphorae in ERROR state
openstack loadbalancer amphora list --status ERROR

# Check logs on the amphora (requires SSH access to management network)
ssh -i /etc/octavia/ssh/octavia_ssh_key ubuntu@<amphora-mgmt-ip>
sudo journalctl -u amphora-agent
sudo haproxy -c -f /var/lib/octavia/<lb-uuid>/haproxy.cfg

# Check health-manager is receiving heartbeats
# (on health-manager host)
sudo tcpdump -i any udp port 5555

# Check octavia-worker logs
journalctl -u octavia-worker --since "10 minutes ago"

# Reset a stuck load balancer (admin)
openstack loadbalancer set web-lb --provisioning-status ACTIVE
```
