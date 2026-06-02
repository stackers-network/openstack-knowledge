# Choose a Web Dashboard

OpenStack provides two supported web dashboards: Horizon (the original Django-based dashboard) and Skyline (a newer Vue.js + FastAPI dashboard). Both interact with OpenStack APIs — they do not add capabilities, only UI.

## Feature Comparison

| Feature | Horizon | Skyline |
|---|---|---|
| Maturity | Very mature — production since 2012, every OpenStack release | Newer — contributed to OpenStack in Wallaby (2021), production-ready since Zed |
| Frontend framework | Django + Mako templates (server-side rendering) | Vue.js 3 (single-page application) |
| Backend framework | Django (Python) | FastAPI (Python, async) |
| Architecture | Monolithic (all-in-one Django app) | Decoupled (Skyline-apiserver + Skyline-console) |
| Plugin ecosystem | Large — Nova, Neutron, Cinder, Glance, Heat, Magnum, Octavia, Barbican, Masakari, and more via horizon plugins | Growing — core services (Nova, Neutron, Cinder, Glance) plus expanding plugins |
| Service coverage | Complete — all OpenStack services with mature plugins | Core services well covered; some advanced services (Barbican, Masakari, Heat complex flows) still maturing |
| Performance | Heavier — full page loads for most actions | Lighter — SPA with async API calls, faster perceived performance |
| Customization | Extensive — custom panels, branding, Django settings, SCSS | Moderate — component-level theming, config-driven branding |
| Mobile responsiveness | Partial (not designed for mobile) | Better (SPA architecture handles responsive design more naturally) |
| Multi-domain support | Yes | Yes |
| Multi-region support | Yes (switchable endpoints) | Yes |
| i18n / localization | Yes (many languages) | Yes (growing) |
| Accessibility (a11y) | Partial | Partial |
| Container-friendly | Yes (runs in Docker/K8s) | Yes (designed container-first) |
| Deployment complexity | Simple (wsgi behind Apache/Nginx) | Simple (Docker or systemd for both components) |
| Community direction | Actively maintained, stable | Active development, growing community focus |

## Recommendation by Use Case

### Established Production Cloud, Full Service Coverage Needed
**Use Horizon.** If your cloud runs Heat, Magnum, Octavia, Barbican, or other services beyond the core, Horizon's plugin ecosystem covers them all. It is battle-tested across every OpenStack release.

### New Cloud, Core Services Only
**Use Skyline.** For clouds running Nova, Neutron, Cinder, and Glance, Skyline offers a better user experience — faster, more modern UI — with lower frontend rendering overhead. It is the recommended choice for new deployments where service coverage is not a constraint.

### Public or Self-Service Cloud with Many End Users
**Consider Skyline.** The SPA architecture gives faster navigation and a more app-like feel, which matters for self-service portals used by non-technical users.

### Clouds with Custom Branding or White-labeling
**Use Horizon.** Django's settings.py and the `local_settings.py` override mechanism provide deep customization of themes, logos, default panels, and custom panels per project/domain.

### Both Dashboards in Parallel
Both can coexist. They talk to the same Keystone endpoint and the same APIs. Running both on different URLs is a valid way to evaluate Skyline before committing to migration.

## Horizon Deployment

### Package Installation

```bash
# Ubuntu
apt install openstack-dashboard

# Rocky Linux / RHEL
dnf install openstack-dashboard
```

### Key Configuration File

```python
# /etc/openstack-dashboard/local_settings.py (Ubuntu)
# /etc/openstack-dashboard/local_settings   (RHEL)

OPENSTACK_HOST = "controller"
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "member"
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.memcached.PyMemcacheCache",
        "LOCATION": "controller:11211",
    }
}
```

### Start / Restart

```bash
systemctl restart apache2   # Ubuntu
systemctl restart httpd     # RHEL
```

### Verify

```bash
# Check the wsgi process
ps aux | grep wsgi
curl -s -o /dev/null -w "%{http_code}" http://localhost/dashboard/
# Should return 200 or 302
```

## Skyline Deployment

Skyline consists of two components: `skyline-apiserver` and `skyline-console`.

### Docker Compose (Recommended for Evaluation)

```yaml
# docker-compose.yml
services:
  skyline:
    image: quay.io/openstack.kolla/skyline-apiserver:2025.1-ubuntu-jammy
    volumes:
      - ./skyline.yaml:/etc/skyline/skyline.yaml:ro
    ports:
      - "9999:9999"
    restart: unless-stopped
```

### Configuration File

```yaml
# /etc/skyline/skyline.yaml
default:
  database_url: sqlite:////var/lib/skyline/skyline.db  # or MySQL
  debug: false
  log_dir: /var/log/skyline

openstack:
  keystone_url: http://controller:5000/v3
  default_region: RegionOne
  system_user_name: skyline
  system_user_password: <PASSWORD>
  system_user_domain: Default
  system_project: service
  system_project_domain: Default
  interface_type: public  # or internal

nginx:
  # Skyline-console is served by nginx inside the container
```

### Database Initialization

```bash
# Initialize the Skyline database
skyline-db-init

# Or inside the container
docker exec skyline skyline-db-init
```

### Keystone Setup for Skyline

```bash
# Create Skyline system user
openstack user create --domain Default --password <PASSWORD> skyline
openstack role add --project service --user skyline admin

# Skyline needs system-scoped reader for some admin views
openstack role add --user skyline --system all reader
```

### Verify

```bash
# Health check
curl -s http://localhost:9999/api/openstack/skyline/api/v1/login \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"region": "RegionOne", "username": "admin", "password": "<PASS>", "domain": "Default"}'
```

## Common Operations

```bash
# Horizon — collect static files after config change
cd /usr/share/openstack-dashboard
python manage.py collectstatic --noinput
python manage.py compress --force
systemctl restart apache2

# Horizon — list installed plugins
pip show openstack-dashboard | grep -i location
ls /usr/lib/python3/dist-packages/openstack_dashboard/contrib/

# Skyline — check API server logs
journalctl -u skyline-apiserver --since "10 minutes ago"
# or
docker logs skyline 2>&1 | tail -50
```
