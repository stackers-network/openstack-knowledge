# Horizon Operations

## Install Horizon

```bash
# Ubuntu/Debian
apt install openstack-dashboard

# The package installs:
# - /etc/openstack-dashboard/local_settings.py  (main config)
# - /usr/share/openstack-dashboard/              (Django project root)
# - /etc/apache2/conf-available/openstack-dashboard.conf  (Apache config)
```

Enable the Apache config and restart:

```bash
a2enconf openstack-dashboard
systemctl restart apache2
```

## Configure local_settings.py

The primary configuration file is `/etc/openstack-dashboard/local_settings.py`. Edit it and then run:

```bash
# After any change to local_settings.py:
python3 /usr/share/openstack-dashboard/manage.py compress
python3 /usr/share/openstack-dashboard/manage.py collectstatic --noinput
systemctl restart apache2
```

### Point Horizon at Keystone

```python
# /etc/openstack-dashboard/local_settings.py

# Hostname or IP of the controller (used to build API URLs)
OPENSTACK_HOST = "10.0.0.10"

# Keystone v3 endpoint
OPENSTACK_KEYSTONE_URL = "https://keystone.example.com:5000/v3"

# Default Keystone domain for user authentication
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

# Default role assigned to new users created via the dashboard
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "member"
```

### Set API Versions

Force specific API microversions for each service:

```python
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
    "compute": 2,
    "network": 2,
}
```

### Enable Multi-Domain Support

When multi-domain support is enabled, the Horizon login page shows a "Domain" field, allowing users from non-Default domains to authenticate:

```python
# Show the Domain field on the login page
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

# If True, the Domain field defaults to OPENSTACK_KEYSTONE_DEFAULT_DOMAIN
# If False, users must always type their domain
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
```

### Configure Session Backend (Memcached)

Using Memcached for sessions prevents session loss on Apache restarts and enables session sharing across multiple Horizon nodes:

```python
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.PyMemcacheCache',
        'LOCATION': [
            '10.0.0.10:11211',
            '10.0.0.11:11211',
        ],
        'OPTIONS': {
            'no_delay': True,
            'ignore_exc': True,
        },
        'TIMEOUT': 3600,
    }
}

# Session cookie lifetime (seconds). 0 = browser session only.
SESSION_TIMEOUT = 3600
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SECURE = True   # Set True when using HTTPS
```

### Configure Allowed Hosts

```python
# List of hostnames or IPs that can be used to access Horizon.
# Prevents HTTP Host header injection attacks.
ALLOWED_HOSTS = [
    'dashboard.example.com',
    '10.0.0.10',
]
```

### Configure HTTPS Settings

```python
# Set to True when Horizon is served over HTTPS
CSRF_COOKIE_SECURE = True
SESSION_COOKIE_SECURE = True

# Tell Django that HTTPS is being used (when behind a TLS-terminating proxy)
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

### Enable or Disable Panels and Services

Disable panels for services that are not deployed:

```python
# Disable services that are not installed
OPENSTACK_HYPERVISOR_FEATURES = {
    'can_set_mount_point': False,
    'can_set_password': True,
    'requires_keypair': False,
    'enable_quotas': True,
}

# Disable the DNS (Designate) panel
OPENSTACK_ENABLED_PANELS = {
    'project': ['compute', 'volumes', 'network', 'object_store'],
    'admin': ['compute', 'volumes', 'network', 'system'],
    'identity': ['domains', 'projects', 'users', 'groups', 'roles'],
}
```

Alternatively, disable individual `_enable_<panel>.py` files in `/etc/openstack-dashboard/enabled/`:

```bash
# Disable the DNS panel
mv /etc/openstack-dashboard/enabled/_70_dns.py \
   /etc/openstack-dashboard/enabled/_70_dns.py.disabled
```

Or use the `enabled/` directory pattern: Horizon loads files matching `_*.py` sorted alphabetically. Create a file that sets `DISABLED = True`:

```python
# /etc/openstack-dashboard/enabled/_70_dns_disabled.py
DISABLED = True
```

### Configure Network Topology

```python
OPENSTACK_NEUTRON_NETWORK = {
    'enable_router': True,
    'enable_quotas': True,
    'enable_ipv6': True,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_fip_topology_check': True,
    'enable_rbac_policy': True,
    # Default DNS nameservers for new subnets
    'default_dns_nameservers': ["8.8.8.8", "8.8.4.4"],
}
```

### Configure Available Instance Metadata

```python
CREATE_INSTANCE_FLAVOR_SORT = {
    'key': 'ram',
    'reverse': False,
}

# Show/hide certain fields in the launch instance wizard
LAUNCH_INSTANCE_DEFAULTS = {
    'config_drive': False,
    'enable_scheduler_hints': True,
    'disable_image': False,
    'disable_instance_snapshot': False,
    'disable_volume': False,
    'disable_volume_snapshot': False,
    'create_volume': True,
}
```

## Custom Branding

### Set the Site Name and Logo

```python
# /etc/openstack-dashboard/local_settings.py

# Browser tab title and header
SITE_BRANDING = "ACME Cloud"

# Welcome message on the login page
SITE_BRANDING_LINK = "https://cloud.acme.example.com"
```

### Override with a Custom Theme

Horizon supports switchable themes. Custom themes live in a directory containing SCSS and template overrides:

```bash
# Create a custom theme directory
mkdir -p /usr/share/openstack-dashboard/openstack_dashboard/themes/acme/{static,templates}

# Copy and modify the base theme
cp -r /usr/share/openstack-dashboard/openstack_dashboard/themes/material/* \
  /usr/share/openstack-dashboard/openstack_dashboard/themes/acme/

# Edit _variables.scss to change colors/logo
nano /usr/share/openstack-dashboard/openstack_dashboard/themes/acme/static/_variables.scss
```

Register the theme in `local_settings.py`:

```python
AVAILABLE_THEMES = [
    ('default', 'Default', 'themes/default'),
    ('material', 'Material', 'themes/material'),
    ('acme', 'ACME', 'themes/acme'),
]

# Set the default theme
DEFAULT_THEME = 'acme'
```

Place a custom logo at `themes/acme/static/img/logo.svg` and reference it in `_variables.scss`:

```scss
// themes/acme/static/_variables.scss
$brand-logo-url: url('../img/logo.svg');
$brand-logo-width: 200px;
$brand-logo-height: 48px;
```

Recompile after theme changes:

```bash
python3 /usr/share/openstack-dashboard/manage.py compress
python3 /usr/share/openstack-dashboard/manage.py collectstatic --noinput
systemctl restart apache2
```

## WebSSO Configuration

Enable Keystone-federated SSO on the Horizon login page:

```python
# /etc/openstack-dashboard/local_settings.py

# Enable WebSSO
WEBSSO_ENABLED = True

# List of SSO choices shown on the login page
# Format: (protocol_id, display_name)
WEBSSO_CHOICES = (
    ("credentials", _("Keystone Credentials")),
    ("saml2", _("Corporate SSO (SAML2)")),
    ("oidc", _("Google/OIDC")),
)

# Default to credentials (not SSO)
WEBSSO_DEFAULT_REDIRECT = False

# Keystone WebSSO callback URL — must match the URL in your IdP
# and in keystone.conf [federation] trusted_dashboard
WEBSSO_KEYSTONE_URL = "https://keystone.example.com:5000/v3"
```

Configure Keystone to trust the Horizon redirect URL:

```ini
# /etc/keystone/keystone.conf
[federation]
trusted_dashboard = https://dashboard.example.com/dashboard/auth/websso/
sso_callback_template = /etc/keystone/sso_callback_template.html
```

## Apache WSGI Deployment

The `openstack-dashboard` package installs a working Apache configuration at `/etc/apache2/conf-available/openstack-dashboard.conf`. For production, review and harden it:

```apache
# /etc/apache2/sites-available/openstack-dashboard.conf

<VirtualHost *:443>
    ServerName dashboard.example.com

    SSLEngine On
    SSLCertificateFile    /etc/ssl/certs/dashboard.crt
    SSLCertificateKeyFile /etc/ssl/private/dashboard.key
    SSLCACertificateFile  /etc/ssl/certs/openstack-ca.crt
    SSLProtocol           all -SSLv3 -TLSv1 -TLSv1.1

    WSGIScriptAlias /dashboard /usr/share/openstack-dashboard/openstack_dashboard/wsgi.py
    WSGIDaemonProcess horizon user=horizon group=horizon processes=4 threads=1
    WSGIProcessGroup horizon
    WSGIApplicationGroup %{GLOBAL}

    Alias /static /usr/share/openstack-dashboard/static

    <Directory /usr/share/openstack-dashboard/openstack_dashboard>
        Require all granted
    </Directory>

    <Directory /usr/share/openstack-dashboard/static>
        Require all granted
        ExpiresActive On
        ExpiresDefault "access plus 1 year"
    </Directory>

    # Security headers
    Header always set X-Frame-Options DENY
    Header always set X-Content-Type-Options nosniff
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
    Header always set Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'"

    ErrorLog /var/log/apache2/dashboard-error.log
    CustomLog /var/log/apache2/dashboard-access.log combined
</VirtualHost>

# Redirect HTTP to HTTPS
<VirtualHost *:80>
    ServerName dashboard.example.com
    Redirect permanent / https://dashboard.example.com/
</VirtualHost>
```

Enable and activate:

```bash
a2enmod wsgi ssl headers expires
a2ensite openstack-dashboard
a2disconf openstack-dashboard   # disable the conf-available version
systemctl restart apache2
```

## Nginx Reverse Proxy Deployment

Use Nginx as a reverse proxy in front of Gunicorn (serving Horizon):

```bash
# Start Horizon with Gunicorn
gunicorn --workers 4 --bind 127.0.0.1:8080 \
  openstack_dashboard.wsgi:application \
  --chdir /usr/share/openstack-dashboard
```

```nginx
# /etc/nginx/sites-available/horizon
server {
    listen 443 ssl;
    server_name dashboard.example.com;

    ssl_certificate     /etc/ssl/certs/dashboard.crt;
    ssl_certificate_key /etc/ssl/private/dashboard.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location /dashboard/static/ {
        alias /usr/share/openstack-dashboard/static/;
        expires 1y;
        add_header Cache-Control "public";
    }

    location /dashboard/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Multi-Node Horizon (High Availability)

Run multiple Horizon nodes behind a load balancer. All nodes must:

1. Share the same `SECRET_KEY` in `local_settings.py` (required for CSRF protection and session signing)
2. Use Memcached for session storage (so sessions survive node failover)
3. Share the same `local_settings.py` (deploy via configuration management)

```python
# /etc/openstack-dashboard/local_settings.py

# Must be identical on all Horizon nodes
# Generate once: python3 -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
SECRET_KEY = 'xK9mV2pQ7rL3nZ8wA1sD4tF6gH0jU5eIyB2cN3oP4qR5sT6uW7vX8yZ9'
```

## Database Sync (First Run)

```bash
python3 /usr/share/openstack-dashboard/manage.py migrate
```

This creates the Django session, admin, and auth tables used by Horizon itself (not the OpenStack service databases).

## Troubleshoot Horizon

```bash
# Check for Django configuration errors
python3 /usr/share/openstack-dashboard/manage.py check

# Check Apache error log for WSGI startup failures
tail -50 /var/log/apache2/error.log

# Test that Keystone is reachable from the Horizon server
curl -s https://keystone.example.com:5000/v3 | python3 -m json.tool

# Verify static files are compressed and collected
ls /usr/share/openstack-dashboard/static/dashboard/

# Check session backend (Memcached) is reachable
python3 -c "from pymemcache.client.base import Client; c = Client(('10.0.0.10', 11211)); c.set('test', 'ok'); print(c.get('test'))"

# Enable Django debug mode temporarily (never leave enabled in production)
# In local_settings.py: DEBUG = True
# Then check /dashboard/ in browser for detailed tracebacks
```
