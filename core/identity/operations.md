# Keystone Operations

## Bootstrap Keystone

Run once after installation to create the initial admin user, project, role, service, and endpoint.

```bash
keystone-manage bootstrap \
  --bootstrap-password s3cur3AdminP@ss \
  --bootstrap-admin-url http://10.0.0.10:5000/v3/ \
  --bootstrap-internal-url http://10.0.0.11:5000/v3/ \
  --bootstrap-public-url http://203.0.113.10:5000/v3/ \
  --bootstrap-region-id RegionOne
```

This creates:
- User `admin` in domain `Default`
- Project `admin` in domain `Default`
- Role `admin`
- Role assignment: `admin` user → `admin` role → `admin` project
- Service: `keystone` (type: `identity`)
- Three endpoints (public, internal, admin) in `RegionOne`

## Sync the Database

Run after installation and after every upgrade before starting Keystone.

```bash
keystone-manage db_sync
```

Check for pending migrations:

```bash
keystone-manage db_sync --check
```

Output `0` means up to date. Non-zero means unapplied migrations exist.

## Source Admin Credentials

```bash
export OS_USERNAME=admin
export OS_PASSWORD=s3cur3AdminP@ss
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://10.0.0.10:5000/v3
export OS_IDENTITY_API_VERSION=3
```

Or use a `clouds.yaml`:

```yaml
# ~/.config/openstack/clouds.yaml
clouds:
  mycloud:
    auth:
      auth_url: http://10.0.0.10:5000/v3
      username: admin
      password: s3cur3AdminP@ss
      project_name: admin
      user_domain_name: Default
      project_domain_name: Default
    identity_api_version: 3
    region_name: RegionOne
```

```bash
export OS_CLOUD=mycloud
```

## Manage Domains

### Create a Domain

```bash
openstack domain create acme-corp \
  --description "ACME Corporation tenant domain"
```

### List Domains

```bash
openstack domain list
```

### Show a Domain

```bash
openstack domain show acme-corp
```

### Disable a Domain

Disabling a domain prevents all users and projects within it from authenticating.

```bash
openstack domain set --disable acme-corp
```

### Delete a Domain

Domains must be disabled before deletion.

```bash
openstack domain set --disable acme-corp
openstack domain delete acme-corp
```

## Manage Projects

### Create a Project

```bash
openstack project create engineering \
  --domain Default \
  --description "Engineering team project"
```

Create in a custom domain:

```bash
openstack project create frontend \
  --domain acme-corp \
  --description "Frontend team project"
```

Create a nested project (parent must exist):

```bash
openstack project create backend \
  --domain acme-corp \
  --parent frontend \
  --description "Backend sub-team of frontend"
```

### List Projects

```bash
openstack project list
# List all projects across all domains (requires system-scoped admin token)
openstack project list --domain acme-corp
```

### Show a Project

```bash
openstack project show engineering
```

### Set Project Properties

```bash
openstack project set engineering --description "Engineering team — updated"
openstack project set engineering --disable
openstack project set engineering --enable
```

### Delete a Project

```bash
openstack project delete engineering
```

**Warning**: This is irreversible and triggers cascading deletes across all services. Get explicit operator approval before running this command.

## Manage Users

### Create a User

```bash
openstack user create alice \
  --domain Default \
  --password s3cur3P@ss \
  --email alice@example.com \
  --description "Alice Smith — Engineering"
```

### List Users

```bash
openstack user list
openstack user list --domain acme-corp
```

### Show a User

```bash
openstack user show alice
```

### Set User Properties

```bash
openstack user set alice --email alice.smith@example.com
openstack user set alice --password n3wP@ssw0rd
openstack user set alice --disable
openstack user set alice --enable
```

### Delete a User

```bash
openstack user delete alice
```

### Change a User's Password (self-service)

```bash
openstack user password set --password n3wP@ssw0rd
```

## Manage Groups

### Create a Group

```bash
openstack group create engineers \
  --domain Default \
  --description "All engineering staff"
```

### Add Users to a Group

```bash
openstack group add user engineers alice
openstack group add user engineers bob
```

### List Group Members

```bash
openstack group contains user engineers alice
openstack user list --group engineers
```

### Remove a User from a Group

```bash
openstack group remove user engineers bob
```

## Manage Roles

### Create a Role

```bash
openstack role create project-admin \
  --description "Can manage project resources and members"
```

### List Roles

```bash
openstack role list
```

### Show a Role

```bash
openstack role show member
```

### Add a Role Assignment

```bash
# User on a project
openstack role add --project engineering --user alice member

# User on a domain
openstack role add --domain acme-corp --user bob admin

# User on the system
openstack role add --system all --user svcaccount reader

# Group on a project
openstack role add --project engineering --group engineers member

# Inherited: applies to all projects in the domain
openstack role add --domain Default --user ops-bot admin --inherited
```

### Remove a Role Assignment

```bash
openstack role remove --project engineering --user alice member
```

### List Role Assignments

```bash
openstack role assignment list --project engineering
openstack role assignment list --user alice --names
openstack role assignment list --effective --user alice --names
```

`--effective` expands group memberships and inheritance to show the complete set of access.

### Create an Implied Role

Implied roles enable automatic role hierarchy: assigning the "parent" automatically grants the "child".

```bash
# admin implies member
openstack implied role create admin --implied-role member
# member implies reader
openstack implied role create member --implied-role reader
```

List implied roles:

```bash
openstack implied role list
```

## Manage Services

### Create a Service

```bash
openstack service create --name nova --description "Compute Service" compute
openstack service create --name neutron --description "Networking Service" network
openstack service create --name glance --description "Image Service" image
openstack service create --name cinder --description "Block Storage Service" volume
openstack service create --name cinderv3 --description "Block Storage Service V3" volumev3
openstack service create --name swift --description "Object Storage Service" object-store
openstack service create --name placement --description "Placement Service" placement
openstack service create --name heat --description "Orchestration Service" orchestration
openstack service create --name heat-cfn --description "Orchestration CloudFormation" cloudformation
```

### List Services

```bash
openstack service list
```

### Show a Service

```bash
openstack service show nova
```

### Delete a Service

Deleting a service also deletes all its endpoints.

```bash
openstack service delete nova
```

## Manage Endpoints

### Create Endpoints

```bash
# Nova — public, internal, admin
openstack endpoint create --region RegionOne \
  nova public http://203.0.113.10:8774/v2.1
openstack endpoint create --region RegionOne \
  nova internal http://10.0.0.11:8774/v2.1
openstack endpoint create --region RegionOne \
  nova admin http://10.0.0.11:8774/v2.1

# Keystone
openstack endpoint create --region RegionOne \
  keystone public http://203.0.113.10:5000/v3
openstack endpoint create --region RegionOne \
  keystone internal http://10.0.0.11:5000/v3
openstack endpoint create --region RegionOne \
  keystone admin http://10.0.0.11:5000/v3
```

### List Endpoints

```bash
openstack endpoint list
openstack endpoint list --service nova
openstack endpoint list --region RegionOne --interface public
```

### Show an Endpoint

```bash
openstack endpoint show <endpoint-id>
```

### Update an Endpoint

```bash
openstack endpoint set <endpoint-id> --url http://10.0.0.12:8774/v2.1
openstack endpoint set <endpoint-id> --disable
```

### Delete an Endpoint

```bash
openstack endpoint delete <endpoint-id>
```

## Issue and Inspect Tokens

### Issue a Token

```bash
openstack token issue
```

Output includes token ID, expiry, user, project, and catalog. The token value is stored in `OS_AUTH_TOKEN`.

Issue a domain-scoped token:

```bash
openstack token issue --os-domain-name Default
```

Issue a system-scoped token:

```bash
openstack --os-system-scope all token issue
```

### Revoke a Token

```bash
openstack token revoke <token-id>
```

## Manage Application Credentials

### Create an Application Credential

```bash
openstack application credential create deploy-robot \
  --description "CI/CD pipeline for engineering project" \
  --role member \
  --expiration "2025-12-31T00:00:00"
```

The `secret` field is shown once. Store it securely (e.g., in Vault or Barbican).

Create with restricted access rules (allow only specific API calls):

```bash
openstack application credential create nova-reader \
  --role reader \
  --access-rules '[{"service": "compute", "method": "GET", "path": "/v2.1/servers"}]'
```

Use in `clouds.yaml`:

```yaml
clouds:
  ci-cloud:
    auth:
      auth_url: http://10.0.0.10:5000/v3
      application_credential_id: abc123def456
      application_credential_secret: xyzSecretValue
    auth_type: v3applicationcredential
    identity_api_version: 3
```

### List Application Credentials

```bash
openstack application credential list
```

### Show an Application Credential

```bash
openstack application credential show deploy-robot
```

### Delete an Application Credential

```bash
openstack application credential delete deploy-robot
```

## Fernet Key Management

### Initial Setup

Run once on the primary Keystone node. Then distribute keys to all Keystone nodes.

```bash
keystone-manage fernet_setup \
  --keystone-user keystone \
  --keystone-group keystone
```

Inspect generated keys:

```bash
ls -la /etc/keystone/fernet-keys/
# 0  (staged key)
# 1  (primary key — used to sign new tokens)
```

### Rotate Fernet Keys

Run periodically (daily or weekly). Must run on all Keystone nodes, or synchronize keys after rotation.

```bash
keystone-manage fernet_rotate \
  --keystone-user keystone \
  --keystone-group keystone
```

After rotation:
- Previous primary becomes secondary
- Staged key becomes new primary
- New staged key is generated
- Oldest secondary (beyond `max_active_keys` count) is deleted

Distribute updated keys to all other Keystone nodes via a secure mechanism (e.g., Ansible vault, rsync over SSH).

### Credential Key Setup

Credential encryption keys are separate from Fernet keys.

```bash
keystone-manage credential_setup \
  --keystone-user keystone \
  --keystone-group keystone
```

### Rotate Credential Keys

```bash
keystone-manage credential_rotate \
  --keystone-user keystone \
  --keystone-group keystone
```

After rotation, re-encrypt stored credentials with the new primary key:

```bash
keystone-manage credential_migrate \
  --keystone-user keystone \
  --keystone-group keystone
```

## Configuration Reference

### `[DEFAULT]`

```ini
[DEFAULT]
log_file = /var/log/keystone/keystone.log
log_dir = /var/log/keystone
debug = false
```

### `[database]`

```ini
[database]
connection = mysql+pymysql://keystone:keystonedb_pass@10.0.0.10/keystone
max_pool_size = 10
max_overflow = 20
pool_timeout = 30
```

### `[identity]`

```ini
[identity]
# Default identity driver (SQL or LDAP)
driver = sql
# List of supported password hashing algorithms (bcrypt recommended)
password_hash_algorithm = bcrypt
# Work factor for bcrypt (4-31, default 12)
password_hash_rounds = 12
# Max domains with per-domain backends before requiring identity_providers override
max_password_length = 4096
```

### `[token]`

```ini
[token]
# Token provider: fernet (default) or jws
provider = fernet
# Token lifetime in seconds (default: 3600 = 1 hour)
expiration = 3600
# Allow re-use of revoked token IDs (security: keep false)
allow_rescope_scoped_token = false
# Cache token validation results
cache_on_issue = true
```

### `[fernet_tokens]`

```ini
[fernet_tokens]
key_repository = /etc/keystone/fernet-keys
# Max number of active Fernet keys (staged + primary + secondaries)
# Minimum: 3 (1 staged + 1 primary + 1 secondary during rotation)
max_active_keys = 5
```

### `[credential]`

```ini
[credential]
# Driver for credential storage
driver = sql
# Encryption key repository (separate from fernet keys)
key_repository = /etc/keystone/credential-keys
```

### `[cache]`

```ini
[cache]
enabled = true
# dogpile.cache backend — memcached recommended for production
backend = dogpile.cache.pymemcache
# Memcached servers
backend_argument = url:10.0.0.10:11211 10.0.0.11:11211
# Cache expiry seconds (should be less than token expiry)
expiration_time = 600
```

### `[oslo_policy]`

```ini
[oslo_policy]
enforce_scope = true
enforce_new_defaults = true
policy_file = policy.yaml
```

### `[catalog]`

```ini
[catalog]
# Catalog driver: sql (default) or templated
driver = sql
```

### `[assignment]`

```ini
[assignment]
# Assignment driver — only sql is supported in modern Keystone
driver = sql
```

### `[resource]`

```ini
[resource]
# Enable caching of project/domain data
cache_time = 600
list_limit = 1000
```

### `[federation]`

```ini
[federation]
# Trusted dashboard origins for WebSSO
trusted_dashboard = http://203.0.113.10/dashboard/auth/websso/
# SSO callback template
sso_callback_template = /etc/keystone/sso_callback_template.html
# Mapping purge interval
assertion_prefix = MELLON_
```

### `[security_compliance]`

```ini
[security_compliance]
# Disable inactive users after N days (0 = disabled)
disable_user_account_days_inactive = 90
# Lock account after N failed login attempts (0 = disabled)
lockout_failure_attempts = 5
# Duration to lock account (seconds) after exceeding lockout_failure_attempts
lockout_duration = 1800
# Number of previous passwords to check against on password change
unique_last_password_count = 5
# Minimum days between password changes
minimum_password_age = 1
# Password expiry in days (0 = never)
password_expires_days = 180
# Regex pattern for password strength
password_regex = ^(?=.*\d)(?=.*[a-zA-Z]).{8,}$
password_regex_description = "Must be 8+ chars with letters and numbers"
```

## Verify Keystone Health

```bash
# Check the service catalog
openstack catalog list

# Check all endpoints are reachable
openstack endpoint list --interface public

# Verify token issuance
openstack token issue

# Check service status (if using systemd)
systemctl status httpd   # or apache2
systemctl status memcached

# Run Keystone doctor checks
keystone-manage doctor
```
