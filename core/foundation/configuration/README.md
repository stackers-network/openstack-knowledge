# OpenStack Configuration

Comprehensive reference for OpenStack configuration: oslo.config format, paste.ini WSGI pipelines, policy.yaml RBAC, logging, per-service config file locations, and validation.

---

## oslo.config

Every OpenStack service uses **oslo.config** to parse configuration files and CLI options. oslo.config enforces a declared schema — each option is registered by the service code with a type, default value, and help text. Unknown options in config files produce a warning but do not cause failures.

### INI Format

Config files use Python-style INI format with `[SECTION]` headings and `key = value` pairs:

```ini
[DEFAULT]
debug = false
log_file = /var/log/nova/nova-api.log

[database]
connection = mysql+pymysql://nova:secret@10.0.0.5/nova

[keystone_authtoken]
www_authenticate_uri = http://10.0.0.5:5000
auth_url = http://10.0.0.5:5000
memcached_servers = 10.0.0.5:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova_service_password
```

Rules:
- Section names are case-sensitive: `[DEFAULT]` not `[default]`.
- Option names are case-insensitive but lowercase by convention.
- Inline comments use `#` or `;` — but do not put a comment on the same line as a value.
- Multi-value options use repeated keys or comma-separated values depending on the option type.
- Boolean values: `true` / `false` (oslo.config also accepts `1`/`0`, `yes`/`no`, `on`/`off`).

### `--config-file` Flag

All oslo.config services accept `--config-file` on the command line. Multiple files are merged, with later files taking priority:

```bash
nova-api --config-file /etc/nova/nova.conf \
         --config-file /etc/nova/nova-api-extra.conf
```

The default config file path per service is `/etc/<service>/<service>.conf` (e.g., `/etc/nova/nova.conf`). Services also read `/etc/<service>/<service>.conf.d/*.conf` if that directory exists, in lexicographic order.

### Generate a Sample Config

Use `oslo-config-generator` to produce a sample config showing every option with its default value and docstring:

```bash
# Install the generator
pip install oslo.config

# Generate for nova (uses the config-generator entry point registered by nova)
oslo-config-generator --config-file /usr/share/nova/nova-config-generator.conf \
  --output-file /tmp/nova-sample.conf

# Or generate for a single namespace:
oslo-config-generator --namespace nova.conf \
  --output-file /tmp/nova-sample.conf

# Show all options for a namespace in the terminal:
oslo-config-generator --namespace oslo.messaging --format ini
```

Generated samples are annotated:

```ini
#
# From nova.conf
#

# Enable verbose output (boolean value)
# Deprecated group/name - [DEFAULT]/verbose
#verbose = false

# Default log levels for nova (list value)
#default_log_levels = amqp=WARN,amqplib=WARN,boto=WARN,...
```

Uncommented options are required or have non-trivial defaults. Commented-out options show the default.

---

## Common Config Sections

### `[DEFAULT]`

Global options not belonging to a specific subsystem.

```ini
[DEFAULT]
# Logging
debug = false
log_file = /var/log/nova/nova-api.log
log_dir = /var/log/nova
use_journal = false
use_syslog = false

# Transport URL for oslo.messaging (RabbitMQ)
transport_url = rabbit://openstack:rabbit_password@10.0.0.5:5672/

# Service-specific: Nova API workers
osapi_compute_workers = 4

# Service binding
my_ip = 10.0.0.20
```

### `[database]`

SQLAlchemy connection string and pool settings. Used by every service.

```ini
[database]
connection = mysql+pymysql://nova:nova_db_password@10.0.0.5/nova
connection_recycle_time = 3600
max_pool_size = 10
max_overflow = 20
pool_timeout = 30
# For MariaDB Galera — use all three nodes:
# connection = mysql+pymysql://nova:nova_db_password@10.0.0.5,10.0.0.6,10.0.0.7/nova
```

Driver prefixes:
- `mysql+pymysql://` — MariaDB or MySQL via PyMySQL (recommended; no native C dependency)
- `mysql+mysqldb://` — MariaDB/MySQL via MySQLdb (requires `python3-mysqldb`)
- `postgresql+psycopg2://` — PostgreSQL via psycopg2

### `[keystone_authtoken]`

Configures `keystonemiddleware.auth_token` for validating inbound API tokens. Present in every service except Keystone itself.

```ini
[keystone_authtoken]
www_authenticate_uri = http://10.0.0.5:5000
auth_url = http://10.0.0.5:5000
memcached_servers = 10.0.0.5:11211,10.0.0.6:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova_service_password
service_token_roles = service
service_token_roles_required = true
```

Key options:
- `www_authenticate_uri`: Public Keystone endpoint, returned in `WWW-Authenticate` headers on 401.
- `auth_url`: Keystone endpoint used by the middleware itself to fetch tokens.
- `memcached_servers`: Token cache. Critical for performance; prevents per-request Keystone calls.
- `service_token_roles_required`: Enforce that service-to-service calls carry a valid service role token. Set to `true` in production.

### `[oslo_messaging_rabbit]`

RabbitMQ-specific settings for oslo.messaging (used when `transport_url` is a `rabbit://` URI).

```ini
[oslo_messaging_rabbit]
rabbit_ha_queues = true
heartbeat_timeout_threshold = 60
heartbeat_rate = 2
# TLS for RabbitMQ
# ssl = true
# ssl_ca_file = /etc/ssl/certs/ca.pem
# ssl_certfile = /etc/ssl/certs/client.pem
# ssl_keyfile = /etc/ssl/private/client.key
kombu_reconnect_delay = 1.0
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
rabbit_interval_max = 30
```

Note: `transport_url` in `[DEFAULT]` is the primary setting. The `[oslo_messaging_rabbit]` section provides tuning, not the connection URL.

### `[oslo_concurrency]`

Controls locking behavior for concurrent operations.

```ini
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

`lock_path` must be a writable directory. Used by Nova for file-based locks during concurrent disk operations. If unset, oslo.concurrency uses `tempfile.gettempdir()`, which may be cleaned up by the OS.

### `[cache]`

Configures oslo.cache (dogpile.cache) for service-level caching (separate from keystonemiddleware's Memcached config).

```ini
[cache]
enabled = true
backend = oslo_cache.memcache_pool
memcache_servers = 10.0.0.5:11211,10.0.0.6:11211
expiration_time = 600
tls_enabled = false
```

Common backends:
- `oslo_cache.memcache_pool` — Memcached with connection pooling (production).
- `dogpile.cache.memory` — In-process memory (single-process testing only).
- `dogpile.cache.null` — Disables caching (debugging).

---

## paste.ini / WSGI Pipelines

OpenStack API services are WSGI applications deployed behind a web server (Apache + `mod_wsgi`, `uwsgi`, or `gunicorn`). The WSGI pipeline is defined in a `paste.ini` file using **PasteDeploy** syntax.

### What paste.ini Does

`paste.ini` defines a chain of WSGI middleware and the application endpoint. Each request passes through the middleware chain before reaching the service's application object.

### Example — Nova `api-paste.ini`

```ini
[composite:osapi_compute]
use = call:nova.api.openstack.urlmap:urlmap_factory
/                        oscomputeversions
/v2                      osapi_compute_app_v21
/v2.1                    osapi_compute_app_v21

[app:osapi_compute_app_v21]
paste.app_factory = nova.api.openstack.compute:APIRouterV21.factory

[pipeline:osapi_compute_app_v21]
pipeline = cors http_proxy_to_wsgi compute_req_id faultwrap request_log sizelimit osprofiler authtoken keystonecontext osapi_compute_app_v21

[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory

[filter:keystonecontext]
paste.filter_factory = nova.api.auth:NovaKeystoneContext.factory

[filter:cors]
paste.filter_factory = oslo_middleware.cors:filter_factory
oslo_config_config = nova.conf.CONF

[filter:http_proxy_to_wsgi]
paste.filter_factory = oslo_middleware.http_proxy_to_wsgi:HTTPProxyToWSGI.factory
```

Key pipeline elements:
- `authtoken` — `keystonemiddleware.auth_token`: validates the `X-Auth-Token` header.
- `keystonecontext` — populates the service's request context from the token data injected by `authtoken`.
- `cors` — adds CORS headers for browser-based clients (Horizon).
- `faultwrap` — catches unhandled exceptions and returns proper JSON error responses.
- `sizelimit` — rejects requests with bodies exceeding `max_request_body_size`.

### File Locations

| Service | paste.ini path |
|---|---|
| Nova | `/etc/nova/api-paste.ini` |
| Neutron | `/etc/neutron/api-paste.ini` |
| Glance | `/etc/glance/glance-api-paste.ini` |
| Cinder | `/etc/cinder/api-paste.ini` |
| Keystone | `/etc/keystone/keystone-paste.ini` |

Do not modify `paste.ini` files unless adding or removing middleware. The default pipelines are correct for standard deployments.

---

## policy.yaml

oslo.policy implements Role-Based Access Control (RBAC) for every OpenStack API action. Rules are read from `policy.yaml` (or the legacy `policy.json`) at service startup and can be reloaded without restart.

### Rule Syntax

```yaml
# policy.yaml

# Simple role check
"admin_required": "role:admin"

# OR logic
"admin_or_owner": "rule:admin_required or project_id:%(project_id)s"

# Always allow
"allow_all": "@"

# Always deny
"deny_all": "!"

# Compound rule
"compute:create": "rule:admin_or_owner"

# Scope-based rule (system scope)
"compute:create_flavor": "role:admin and system_scope:all"
```

### Scope Concepts

OpenStack introduced scope types in Queens / Rocky to distinguish between cloud-wide and project-scoped operations:

| Scope | `system_scope` value | Use case |
|---|---|---|
| System | `all` | Cloud-wide admin operations (create flavors, manage hypervisors) |
| Project | `%(project_id)s` | Tenant operations (create instances, manage own networks) |
| Domain | `%(domain_id)s` | Domain admin operations (manage users within a domain) |

Enable scope enforcement and new defaults in service config:

```ini
[oslo_policy]
enforce_scope = true
enforce_new_defaults = true
```

Setting `enforce_scope = true` causes requests without a matching scope to be rejected. Migrate to new defaults before enabling in production — use the `oslopolicy-checker` to audit current policies first.

### Generate Sample Policies

```bash
# Generate sample policy for nova
oslopolicy-sample-generator --config-file /usr/share/nova/nova-policy-generator.conf \
  --output-file /etc/nova/policy.yaml.sample

# Or via the registered entry point:
oslopolicy-sample-generator --namespace nova \
  --output-file /tmp/nova-policy.yaml
```

The generated file contains every policy rule with its default value, a docstring, and the scope type.

### Check a Policy Rule

```bash
# Check if user with role 'member' in project 'abc123' can create an instance
oslopolicy-checker \
  --config-file /etc/nova/nova.conf \
  --policy /etc/nova/policy.yaml \
  --rule "os_compute_api:servers:create" \
  --credentials '{"roles": ["member"], "project_id": "abc123ef-..."}'
```

### Override Only What You Need

`policy.yaml` is a delta — you only need to include rules you want to override. Empty file = all defaults. Example: allow members to create flavors (normally admin-only):

```yaml
# /etc/nova/policy.yaml
"compute:create_flavor": "role:member and system_scope:all"
```

### Reload Policies Without Restart

oslo.policy watches `policy.yaml` for changes. Most services reload on the next request after a file modification. Some services require a SIGHUP:

```bash
kill -HUP $(pgrep -f nova-api)
```

---

## Logging

### oslo.log Configuration

Logging is configured in the `[DEFAULT]` section of `<service>.conf`:

```ini
[DEFAULT]
# Log to file (mutually exclusive with log_dir for log file name generation)
log_file = /var/log/nova/nova-api.log

# OR: log to directory (oslo.log names the file by service binary)
log_dir = /var/log/nova

# OR: log to syslog
use_syslog = true
syslog_log_facility = LOG_LOCAL0

# OR: log to systemd journal
use_journal = true

# Log level
debug = false          # If true, sets level to DEBUG; otherwise INFO

# Verbose was removed in Pike — debug=true replaces it

# Include timestamps in log output (default: true when not using journal)
log_date_format = %Y-%m-%d %H:%M:%S
```

### Log Levels

oslo.log supports standard Python log levels. Set per-module level overrides:

```ini
[DEFAULT]
# Override log levels for specific loggers (comma-separated key=value)
default_log_levels = amqp=WARN,amqplib=WARN,boto=WARN,qpid=WARN,sqlalchemy=WARN,suds=INFO,oslo.messaging=INFO,iso8601=WARN,requests.packages.urllib3.connectionpool=WARN,urllib3.connectionpool=WARN,websocket=WARN,requests.packages.urllib3.util.retry=WARN,urllib3.util.retry=WARN,keystonemiddleware=WARN,routes.middleware=WARN,stevedore=WARN,taskflow=WARN,keystoneauth=WARN,oslo.cache=INFO,oslo_policy=WARN
```

### Log Format

oslo.log uses a structured format that includes request IDs:

```
2025-10-15 14:32:01.456 12345 INFO nova.api.openstack.compute.servers [req-abc123 user_id project_id - - -] Create called for server web-01
```

Fields: `timestamp PID LEVEL module [req-id user_id project_id domain_id user_domain_id project_domain_id]`

Custom format:
```ini
[DEFAULT]
logging_context_format_string = %(asctime)s.%(msecs)03d %(process)d %(levelname)s %(name)s [%(request_id)s %(user_identity)s] %(instance)s%(message)s
logging_default_format_string = %(asctime)s.%(msecs)03d %(process)d %(levelname)s %(name)s [-] %(instance)s%(message)s
```

---

## Service Config File Locations

Standard paths for per-service configuration files:

| Service | Main Config | Additional Configs |
|---|---|---|
| Keystone | `/etc/keystone/keystone.conf` | `/etc/keystone/keystone-paste.ini`, `/etc/keystone/policy.yaml`, `/etc/keystone/logging.conf` |
| Nova | `/etc/nova/nova.conf` | `/etc/nova/api-paste.ini`, `/etc/nova/policy.yaml` |
| Neutron | `/etc/neutron/neutron.conf` | `/etc/neutron/api-paste.ini`, `/etc/neutron/policy.yaml`, `/etc/neutron/plugins/ml2/ml2_conf.ini`, `/etc/neutron/l3_agent.ini`, `/etc/neutron/dhcp_agent.ini`, `/etc/neutron/metadata_agent.ini` |
| Glance | `/etc/glance/glance-api.conf` | `/etc/glance/glance-api-paste.ini`, `/etc/glance/policy.yaml` |
| Cinder | `/etc/cinder/cinder.conf` | `/etc/cinder/api-paste.ini`, `/etc/cinder/policy.yaml` |
| Placement | `/etc/placement/placement.conf` | `/etc/placement/policy.yaml` |
| Heat | `/etc/heat/heat.conf` | `/etc/heat/api-paste.ini`, `/etc/heat/policy.yaml` |
| Octavia | `/etc/octavia/octavia.conf` | `/etc/octavia/api-paste.ini`, `/etc/octavia/policy.yaml` |
| Barbican | `/etc/barbican/barbican.conf` | `/etc/barbican/api-paste.ini`, `/etc/barbican/policy.yaml` |
| Designate | `/etc/designate/designate.conf` | `/etc/designate/api-paste.ini`, `/etc/designate/policy.yaml` |
| Ironic | `/etc/ironic/ironic.conf` | `/etc/ironic/api-paste.ini`, `/etc/ironic/policy.yaml` |
| Magnum | `/etc/magnum/magnum.conf` | `/etc/magnum/api-paste.ini`, `/etc/magnum/policy.yaml` |
| Manila | `/etc/manila/manila.conf` | `/etc/manila/api-paste.ini`, `/etc/manila/policy.yaml` |
| Swift | `/etc/swift/swift.conf` | `/etc/swift/proxy-server.conf`, `/etc/swift/account-server.conf`, `/etc/swift/container-server.conf`, `/etc/swift/object-server.conf` |
| Horizon | `/etc/openstack-dashboard/local_settings.py` | (Django settings, not INI) |

### Kolla-Ansible Config Override Directory

In kolla-ansible deployments, per-service config overrides go in `/etc/kolla/config/<service>/`. These are merged into the container at deploy time:

```
/etc/kolla/config/
├── nova/
│   └── nova.conf          # Merged into container's /etc/nova/nova.conf
├── neutron/
│   └── ml2_conf.ini
└── keystone/
    └── keystone.conf
```

---

## Validate Configs Before Restarting Services

### 1. oslo-config-validator

```bash
# Validate nova.conf against the nova option schema
oslo-config-validator --config-file /etc/nova/nova.conf \
  --namespace nova.conf \
  --namespace oslo.messaging \
  --namespace oslo.db \
  --namespace keystonemiddleware.auth_token
```

Reports: unknown options, options with wrong types, deprecated options.

### 2. Service-Specific Validation Commands

Most services have a `--config-file` check or a `verify` subcommand:

```bash
# Nova: run nova-status upgrade check to catch config issues
nova-manage config verify
nova-status upgrade check

# Neutron: validate neutron.conf before restart
neutron-db-manage --config-file /etc/neutron/neutron.conf check_migration

# Keystone: validate and check token signing keys
keystone-manage doctor

# Glance: check config
glance-manage config validate

# Cinder: DB and config check
cinder-manage config list
```

### 3. DB Migration Status Check

Before restarting after a config change that touches the database section:

```bash
nova-manage db version           # Current migration version
nova-manage db sync              # Apply pending migrations (safe to re-run)

neutron-db-manage current        # Show current migration head
neutron-db-manage check_migration

keystone-manage db_sync
```

### 4. Syntax Check Without Starting the Service

Run the service binary with `--help` — oslo.config parses and validates the config file during option processing:

```bash
nova-api --config-file /etc/nova/nova.conf --help > /dev/null
```

Any unknown options or type errors print to stderr before the help text. Exit code is 0 on success.

### 5. Policy Validation

```bash
# Check all rules in policy.yaml parse correctly
oslopolicy-checker --config-file /etc/nova/nova.conf \
  --policy /etc/nova/policy.yaml --all-rules

# List all effective rules (default + overrides)
oslopolicy-list-redundant --config-file /etc/nova/nova.conf \
  --policy /etc/nova/policy.yaml
```

`oslopolicy-list-redundant` identifies rules in `policy.yaml` that exactly match the compiled-in defaults — safe to remove.

---

## Realistic Complete Config Example — Nova

```ini
[DEFAULT]
debug = false
log_file = /var/log/nova/nova-api.log
transport_url = rabbit://openstack:rabbit_pass@10.0.0.5:5672/
my_ip = 10.0.0.20
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
osapi_compute_workers = 4
metadata_workers = 2

[api]
auth_strategy = keystone

[api_database]
connection = mysql+pymysql://nova_api:nova_api_pass@10.0.0.5/nova_api

[database]
connection = mysql+pymysql://nova:nova_pass@10.0.0.5/nova
connection_recycle_time = 3600

[keystone_authtoken]
www_authenticate_uri = http://10.0.0.5:5000
auth_url = http://10.0.0.5:5000
memcached_servers = 10.0.0.5:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova_service_password
service_token_roles = service
service_token_roles_required = true

[service_user]
send_service_user_token = true
auth_url = http://10.0.0.5:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova_service_password

[placement]
auth_url = http://10.0.0.5:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = placement_service_password
region_name = RegionOne

[glance]
api_servers = http://10.0.0.5:9292

[neutron]
auth_url = http://10.0.0.5:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = neutron_service_password
region_name = RegionOne
service_metadata_proxy = true
metadata_proxy_shared_secret = metadata_proxy_secret

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[oslo_messaging_rabbit]
rabbit_ha_queues = true
heartbeat_timeout_threshold = 60

[cache]
enabled = true
backend = oslo_cache.memcache_pool
memcache_servers = 10.0.0.5:11211

[libvirt]
virt_type = kvm
cpu_mode = host-model

[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = 10.0.0.20
novncproxy_base_url = http://10.0.0.5:6080/vnc_auto.html
```

---

## Cross-Reference

- Architecture and service relationships → `core/foundation/architecture/`
- Deployment method config specifics → `core/foundation/deployment-overview/`
- Keystone-specific config (federation, LDAP) → `core/identity/`
- Neutron ML2 config → `core/networking/`
- Cinder backend config → `core/storage/block/`
