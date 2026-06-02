# OpenStack Security Hardening

This file covers cross-service hardening practices applicable to any OpenStack deployment. Apply these after deploying individual services.

## TLS Everywhere

All inter-service and client-to-service communication should use TLS. Plaintext HTTP is acceptable only in isolated lab environments.

### Generate a Self-Signed CA (for Internal Use)

```bash
# Create CA private key and certificate
openssl genrsa -out /etc/ssl/private/openstack-ca.key 4096
openssl req -new -x509 -days 3650 \
  -key /etc/ssl/private/openstack-ca.key \
  -out /etc/ssl/certs/openstack-ca.crt \
  -subj "/CN=OpenStack Internal CA/O=Example Corp/C=US"

# Generate a certificate for each service endpoint
# (repeat for nova, glance, neutron, cinder, barbican, etc.)
SERVICE=keystone
openssl genrsa -out /etc/ssl/private/${SERVICE}.key 2048
openssl req -new \
  -key /etc/ssl/private/${SERVICE}.key \
  -out /tmp/${SERVICE}.csr \
  -subj "/CN=${SERVICE}.example.com/O=Example Corp/C=US"

cat > /tmp/${SERVICE}.ext <<EOF
subjectAltName = DNS:${SERVICE}.example.com, IP:10.0.0.10
EOF

openssl x509 -req -days 825 \
  -in /tmp/${SERVICE}.csr \
  -CA /etc/ssl/certs/openstack-ca.crt \
  -CAkey /etc/ssl/private/openstack-ca.key \
  -CAcreateserial \
  -extfile /tmp/${SERVICE}.ext \
  -out /etc/ssl/certs/${SERVICE}.crt

# Distribute CA certificate to all nodes
cp /etc/ssl/certs/openstack-ca.crt /usr/local/share/ca-certificates/
update-ca-certificates
```

For production, use certificates from a trusted internal CA (e.g., HashiCorp Vault PKI, Dogtag, Microsoft AD CS) or a public CA for external endpoints.

### Configure TLS in Each Service

Every OpenStack service that runs a WSGI app uses Apache or Nginx. Add SSL to the VirtualHost:

```apache
# /etc/apache2/sites-available/keystone.conf (apply same pattern to all services)
<VirtualHost *:5000>
    SSLEngine On
    SSLCertificateFile    /etc/ssl/certs/keystone.crt
    SSLCertificateKeyFile /etc/ssl/private/keystone.key
    SSLCACertificateFile  /etc/ssl/certs/openstack-ca.crt
    SSLProtocol           all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite        ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:...
    SSLHonorCipherOrder   On

    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    ...
</VirtualHost>
```

### Configure the `[ssl]` Section in Service Config Files

Services that make outbound HTTPS calls (e.g., inter-service communication) must trust the CA:

```ini
# /etc/nova/nova.conf
[ssl]
ca_file = /etc/ssl/certs/openstack-ca.crt
cert_file = /etc/ssl/certs/nova.crt
key_file = /etc/ssl/private/nova.key

# /etc/neutron/neutron.conf
[ssl]
ca_file = /etc/ssl/certs/openstack-ca.crt

# /etc/cinder/cinder.conf
[ssl]
ca_file = /etc/ssl/certs/openstack-ca.crt
```

### Update Keystone Endpoints to Use HTTPS

After enabling TLS, update all service catalog endpoints:

```bash
# Find endpoint IDs
openstack endpoint list --service nova

# Update each endpoint URL to https://
openstack endpoint set <ENDPOINT_ID> --url https://nova.example.com:8774/v2.1
```

Repeat for all public, internal, and admin endpoints.

### Configure keystonemiddleware to Verify TLS

Each service's `[keystone_authtoken]` section must trust the CA:

```ini
# /etc/nova/nova.conf
[keystone_authtoken]
www_authenticate_uri = https://keystone.example.com:5000
auth_url = https://keystone.example.com:5000
cafile = /etc/ssl/certs/openstack-ca.crt
```

## oslo.policy: RBAC Hardening

### Enable Secure Defaults and Scope Enforcement

Apply this to every OpenStack service that uses oslo.policy:

```ini
# /etc/nova/nova.conf
[oslo_policy]
enforce_scope = true
enforce_new_defaults = true
policy_file = /etc/nova/policy.yaml

# /etc/neutron/neutron.conf
[oslo_policy]
enforce_scope = true
enforce_new_defaults = true
policy_file = /etc/neutron/policy.yaml

# /etc/cinder/cinder.conf
[oslo_policy]
enforce_scope = true
enforce_new_defaults = true
policy_file = /etc/cinder/policy.yaml

# Repeat for glance, barbican, heat, octavia, etc.
```

- `enforce_scope = true`: tokens must have the correct scope (system/domain/project) for the operation
- `enforce_new_defaults = true`: uses the `admin`/`member`/`reader` role hierarchy; removes legacy `is_admin_project` behavior

### Test a Policy Rule

Use `oslopolicy-checker` to verify a rule before deploying:

```bash
# Check if a project-member user can create a server
oslopolicy-checker \
  --config-file /etc/nova/nova.conf \
  --policy /etc/nova/policy.yaml \
  --rule "os_compute_api:servers:create" \
  --access '{"roles": ["member"], "system_scope": null, "project_id": "proj-123", "user_id": "user-456"}'

# Check if a system-reader can list all servers (admin-only operation)
oslopolicy-checker \
  --config-file /etc/nova/nova.conf \
  --policy /etc/nova/policy.yaml \
  --rule "os_compute_api:servers:detail:get_all_tenants" \
  --access '{"roles": ["reader"], "system_scope": "all", "project_id": null, "user_id": "user-789"}'
```

### Generate a Policy File from Defaults

Generate the current effective policy (in-code defaults merged with any overrides) to understand what rules are in place before making changes:

```bash
oslopolicy-policy-generator \
  --config-file /etc/nova/nova.conf \
  --output-file /tmp/nova-effective-policy.yaml

oslopolicy-policy-generator \
  --config-file /etc/neutron/neutron.conf \
  --output-file /tmp/neutron-effective-policy.yaml
```

### Minimal policy.yaml Overrides

Only place rules in `policy.yaml` that deviate from defaults. An empty file (or absent file) means all defaults apply. Example override to restrict a specific rule:

```yaml
# /etc/nova/policy.yaml
# Restrict flavor creation to system admin only (tighter than default)
"os_compute_api:flavors:create": "role:admin and system_scope:all"
```

## Service User Accounts: Least Privilege

Each service should have its own Keystone user with minimal required roles:

```bash
# Create service project (if it doesn't exist)
openstack project create --domain Default service

# Create per-service users
for SERVICE in nova neutron glance cinder barbican heat octavia; do
  openstack user create ${SERVICE} \
    --domain Default \
    --password "${SERVICE}_service_pass_$(openssl rand -hex 8)" \
    --description "${SERVICE} service user"
  # Most services need 'service' role in the service project
  openstack role add --project service --user ${SERVICE} service
done
```

The `service` role grants elevated inter-service API access (e.g., nova calling neutron for port binding) without granting full `admin` access. If a service requires system-scoped access, use the `admin` role only for that service's specific need.

Store service passwords in Vault or Barbican, not in configuration management repos.

## Keystone Federation for SSO

Integrating with an external Identity Provider (IdP) centralizes authentication:

```bash
# Create an identity provider
openstack identity provider create --remote-id https://sso.example.com/saml/metadata corp-idp

# Create a mapping (map IdP groups to Keystone roles)
cat > /tmp/idp-mapping.json <<'EOF'
[
  {
    "local": [
      {"user": {"name": "{0}"}},
      {"group": {"id": "engineers-group-id"}}
    ],
    "remote": [
      {"type": "MELLON_NAME_ID"},
      {"type": "MELLON_groups", "any_one_of": ["engineering"]}
    ]
  }
]
EOF

openstack mapping create --rules /tmp/idp-mapping.json corp-idp-mapping

# Create a federation protocol
openstack federation protocol create \
  --identity-provider corp-idp \
  --mapping corp-idp-mapping \
  saml2
```

Configure Horizon to use WebSSO:

```python
# /etc/openstack-dashboard/local_settings.py
WEBSSO_ENABLED = True
WEBSSO_CHOICES = (
    ("credentials", _("Keystone Credentials")),
    ("saml2", _("Corporate SSO (SAML2)")),
)
WEBSSO_DEFAULT_REDIRECT = False
```

## Barbican Integration with Other Services

### Nova: Volume Encryption

Configure Nova to use Barbican for managing volume encryption keys:

```ini
# /etc/nova/nova.conf
[key_manager]
backend = barbican
```

Configure Cinder for encrypted volume types:

```ini
# /etc/cinder/cinder.conf
[key_manager]
backend = barbican
```

Create an encrypted volume type:

```bash
# Create the volume type
openstack volume type create --encryption-provider nova.volume.encryptors.luks.LuksEncryptor \
  --encryption-cipher aes-xts-plain64 \
  --encryption-key-size 256 \
  --encryption-control-location front-end \
  LUKS-encrypted
```

When a volume of this type is created, Cinder requests a new AES-256 key from Barbican via castellan and stores the key reference with the volume. The key is fetched at attach time.

### Octavia: TLS Termination

Configure Octavia to retrieve TLS certificates from Barbican:

```ini
# /etc/octavia/octavia.conf
[certificates]
cert_manager = barbican_cert_manager

[key_manager]
auth_endpoint = https://keystone.example.com:5000
```

Create a TLS-terminated listener using a Barbican container:

```bash
# Store certificate and key
CERT_REF=$(openstack secret store \
  --name lb-cert \
  --payload "$(cat /etc/ssl/certs/lb.crt)" \
  --payload-content-type 'application/pkix-cert' \
  --secret-type certificate \
  -f value -c "Secret href")

KEY_REF=$(openstack secret store \
  --name lb-key \
  --payload "$(cat /etc/ssl/private/lb.key)" \
  --payload-content-type 'application/pkcs8' \
  --secret-type private \
  -f value -c "Secret href")

CONTAINER_REF=$(openstack secret container create \
  --name lb-tls \
  --type certificate \
  --secret "certificate=$CERT_REF" \
  --secret "private_key=$KEY_REF" \
  -f value -c "Container href")

# Create HTTPS listener using the container
openstack loadbalancer listener create \
  --name https-listener \
  --protocol TERMINATED_HTTPS \
  --protocol-port 443 \
  --default-tls-container-ref "$CONTAINER_REF" \
  my-loadbalancer
```

### Swift: At-Rest Encryption

Configure Swift proxy to use Barbican for encryption key management:

```ini
# /etc/swift/proxy-server.conf
[filter:keymaster]
use = egg:swift#keymaster
encryption_root_secret = /etc/swift/encryption.secret
# Or use Barbican:
key_manager_backend = barbican
```

The Swift encryption middleware (`[filter:encryption]`) must appear in the pipeline:

```ini
[pipeline:main]
pipeline = catch_errors ... keymaster encryption ... proxy-server
```

## Security Groups Best Practices

Apply a default-deny security posture:

```bash
# Remove the default "allow all egress" rule from the default security group
DEFAULT_SG=$(openstack security group list --project my-project -f value -c ID | head -1)
openstack security group rule list $DEFAULT_SG -f value -c ID | while read RULE_ID; do
  openstack security group rule delete "$RULE_ID"
done

# Create purpose-specific security groups
# Web tier: allow HTTP/HTTPS from anywhere, SSH from bastion only
openstack security group create web-tier --description "Web servers"
openstack security group rule create web-tier --protocol tcp --dst-port 80 --remote-ip 0.0.0.0/0
openstack security group rule create web-tier --protocol tcp --dst-port 443 --remote-ip 0.0.0.0/0
openstack security group rule create web-tier --protocol tcp --dst-port 22 --remote-ip 10.0.0.100/32

# App tier: allow only from web tier
openstack security group create app-tier --description "Application servers"
WEB_SG_ID=$(openstack security group show web-tier -f value -c id)
openstack security group rule create app-tier --protocol tcp --dst-port 8080 --remote-group $WEB_SG_ID

# DB tier: allow only from app tier
openstack security group create db-tier --description "Database servers"
APP_SG_ID=$(openstack security group show app-tier -f value -c id)
openstack security group rule create db-tier --protocol tcp --dst-port 3306 --remote-group $APP_SG_ID

# Allow return traffic (stateful firewall handles this, but explicit is clearer)
# Egress: allow established connections back; drop unsolicited outbound
openstack security group rule create web-tier --direction egress --protocol tcp --dst-port 1:65535
```

## Network Segmentation

Use separate networks for different traffic types:

| Network | Purpose | Typical CIDR | Routed? |
|---|---|---|---|
| Management | SSH, API endpoints, IPMI | 10.0.0.0/24 | Yes (restricted) |
| Tenant/Overlay | VM-to-VM traffic (VXLAN/GRE) | 10.1.0.0/16 | No (tunneled) |
| Storage | Ceph cluster traffic, iSCSI | 10.2.0.0/24 | No (isolated) |
| External/Provider | Floating IPs, public access | 203.0.113.0/24 | Yes (public) |

Configure service APIs to bind only on management network interfaces:

```ini
# /etc/nova/nova.conf — bind to management IP only
[DEFAULT]
my_ip = 10.0.0.11

# /etc/glance/glance-api.conf
[DEFAULT]
bind_host = 10.0.0.12
bind_port = 9292
```

## Audit Logging with CADF

OpenStack services emit notifications in CADF (Cloud Audit Data Federation) format for security-relevant events (API calls, authentication, resource creation/deletion).

### Enable Notifications in Each Service

```ini
# /etc/nova/nova.conf
[oslo_messaging_notifications]
driver = messagingv2
transport_url = rabbit://nova:nova_rabbit_pass@10.0.0.10:5672/nova
topics = notifications

# /etc/keystone/keystone.conf
[oslo_messaging_notifications]
driver = messagingv2
transport_url = rabbit://keystone:ks_rabbit_pass@10.0.0.10:5672/keystone
topics = notifications
```

### Process Notifications with Panko or Custom Consumer

```bash
# Example: consume notifications with a simple Python consumer
python3 - <<'EOF'
import oslo_messaging as messaging
transport = messaging.get_notification_transport(cfg.CONF)
listener = messaging.get_notification_listener(
    transport,
    targets=[messaging.Target(topic='notifications')],
    endpoints=[MyAuditEndpoint()],
    executor='threading'
)
listener.start()
EOF
```

For centralized audit logging, forward CADF events to a SIEM (Splunk, Elasticsearch, or Wazuh) using a custom oslo.messaging consumer or by configuring the `log` notification driver to write structured JSON logs.

## Database Security

### Use Dedicated Users Per Service

```sql
-- On the MariaDB server
CREATE USER 'nova'@'10.0.0.11' IDENTIFIED BY 'nova_db_strong_pass';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'10.0.0.11';

CREATE USER 'neutron'@'10.0.0.12' IDENTIFIED BY 'neutron_db_strong_pass';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'10.0.0.12';

-- Do NOT use GRANT ALL ON *.* — each service gets only its own database
FLUSH PRIVILEGES;
```

### Enable TLS for Database Connections

```bash
# Generate MariaDB server certificate
openssl req -new -x509 -days 825 \
  -key /etc/mysql/ssl/server.key \
  -out /etc/mysql/ssl/server.crt \
  -subj "/CN=db.example.com"
```

```ini
# /etc/mysql/mariadb.conf.d/50-server.cnf
[mysqld]
ssl-ca   = /etc/mysql/ssl/ca.crt
ssl-cert = /etc/mysql/ssl/server.crt
ssl-key  = /etc/mysql/ssl/server.key
require_secure_transport = ON
```

Configure services to use TLS for DB connections:

```ini
# /etc/nova/nova.conf
[database]
connection = mysql+pymysql://nova:nova_db_pass@db.example.com/nova?ssl_ca=/etc/ssl/certs/openstack-ca.crt
```

## RabbitMQ Security

### Enable TLS for RabbitMQ

```bash
# /etc/rabbitmq/rabbitmq.conf
listeners.ssl.default = 5671
ssl_options.cacertfile = /etc/rabbitmq/ssl/ca.crt
ssl_options.certfile   = /etc/rabbitmq/ssl/server.crt
ssl_options.keyfile    = /etc/rabbitmq/ssl/server.key
ssl_options.verify     = verify_peer
ssl_options.fail_if_no_peer_cert = true
```

### Create Per-Service VHosts and Users

```bash
# Create virtual hosts per service
rabbitmqctl add_vhost /nova
rabbitmqctl add_vhost /neutron
rabbitmqctl add_vhost /cinder
rabbitmqctl add_vhost /barbican

# Create per-service users
rabbitmqctl add_user nova nova_rabbit_pass
rabbitmqctl add_user neutron neutron_rabbit_pass

# Grant permissions only to the service's own vhost
rabbitmqctl set_permissions -p /nova nova ".*" ".*" ".*"
rabbitmqctl set_permissions -p /neutron neutron ".*" ".*" ".*"

# Delete the default guest user
rabbitmqctl delete_user guest
```

Configure services to use TLS and their dedicated vhost:

```ini
# /etc/nova/nova.conf
[oslo_messaging_rabbit]
rabbit_use_ssl = true
kombu_ssl_ca_certs = /etc/ssl/certs/openstack-ca.crt
transport_url = rabbit://nova:nova_rabbit_pass@10.0.0.10:5671/nova
```

## Keystone Security Compliance Settings

Enable account lockout and password policies in Keystone:

```ini
# /etc/keystone/keystone.conf
[security_compliance]
# Lock account after 5 failed attempts
lockout_failure_attempts = 5
# Lock for 30 minutes
lockout_duration = 1800
# Require password change every 180 days
password_expires_days = 180
# Prevent reuse of last 5 passwords
unique_last_password_count = 5
# Enforce password complexity
password_regex = ^(?=.*\d)(?=.*[a-zA-Z]).{12,}$
password_regex_description = "Must be 12+ chars with letters and numbers"
# Disable users inactive for 90 days
disable_user_account_days_inactive = 90
```

## Security Audit Checklist

Run these checks periodically:

```bash
# 1. Verify no services have debug mode enabled in production
grep -r "^debug = true" /etc/nova /etc/neutron /etc/cinder /etc/glance /etc/keystone

# 2. Verify all endpoints use HTTPS
openstack endpoint list --interface public | grep http://

# 3. Verify enforce_scope is enabled on all services
grep -r "enforce_scope" /etc/nova /etc/neutron /etc/cinder /etc/glance /etc/keystone

# 4. Check for overly permissive security group rules
openstack security group rule list --all-projects | grep '0.0.0.0/0' | grep -v '80\|443'

# 5. Verify Barbican secrets have appropriate ACLs
openstack secret list | while read SECRET; do
  openstack acl get "$SECRET"
done

# 6. Check for users with system-admin role (should be minimal)
openstack role assignment list --role admin --system all --names

# 7. Verify RabbitMQ guest user is deleted
rabbitmqctl list_users | grep guest

# 8. Check for services running as root
ps aux | grep -E "(nova|neutron|glance|cinder|barbican)" | grep "^root"
```
