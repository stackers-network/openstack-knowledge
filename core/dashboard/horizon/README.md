# Horizon

Horizon is the official OpenStack dashboard. It is a Django-based web application that provides a browser interface for managing OpenStack resources. Horizon wraps the OpenStack REST APIs and presents them as panels organized by service (Project, Admin, Identity).

## When to Read This Skill

- Deploying Horizon for the first time
- Configuring `local_settings.py` for a production environment
- Enabling multi-domain authentication
- Setting up WebSSO with a federated identity provider
- Adding custom branding (logo, colors, site name)
- Enabling or disabling service panels
- Deploying behind Apache or Nginx
- Developing or installing third-party dashboard plugins

## Sub-Files

| File | What It Covers |
|---|---|
| [operations.md](operations.md) | `local_settings.py` settings, Apache/Nginx deployment, multi-domain, SSO, branding, panel management |
| [internals.md](internals.md) | Django architecture, Dashboard/PanelGroup/Panel hierarchy, plugin registration, custom panel development |

## Quick Reference

```bash
# Install Horizon
apt install openstack-dashboard

# After editing local_settings.py, compress static assets
python3 /usr/share/openstack-dashboard/manage.py compress
python3 /usr/share/openstack-dashboard/manage.py collectstatic --noinput

# Restart Apache
systemctl restart apache2

# Check for configuration errors
python3 /usr/share/openstack-dashboard/manage.py check

# Show enabled panels (lists all installed Django apps)
python3 /usr/share/openstack-dashboard/manage.py shell -c \
  "import horizon; print([d.get_dashboard(d) for d in horizon.base.Horizon._registry.keys()])"
```

## Key Configuration File

`/etc/openstack-dashboard/local_settings.py` — the primary configuration file. It is imported by the Django settings module at startup and overrides defaults from `openstack_dashboard/defaults.py`.

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| Keystone | Authentication and service catalog | Horizon authenticates users and fetches the catalog from Keystone |
| Apache httpd 2.4+ (mod_wsgi) | WSGI server | `openstack-dashboard` package installs the Apache config automatically |
| Memcached 1.5+ | Django session storage | Strongly recommended over database sessions; reduces DB load |
| Python 3.10+ | Runtime | 3.11 recommended |
| `python-openstackclient` (optional) | CLI access alongside dashboard | Not required but useful for operators |
| All OpenStack service clients | API access per panel | `python-novaclient`, `python-neutronclient`, etc. are pulled in as dependencies |

## URL

After installation, Horizon is available at:

```
http://<controller-ip>/dashboard
```

Or with SSL:

```
https://<controller-hostname>/dashboard
```

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

Horizon release notes: https://docs.openstack.org/releasenotes/horizon/
