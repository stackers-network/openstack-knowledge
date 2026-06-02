# Dashboard — Horizon and Skyline

OpenStack ships two web dashboards: **Horizon** (the original, Django-based dashboard, in service since 2012) and **Skyline** (a newer alternative with a Vue.js 3 frontend and a FastAPI backend, introduced in 2021). Both provide a graphical interface to OpenStack APIs, but they differ substantially in architecture, maturity, and service coverage.

## When to Read This Skill

- Deploying a web UI for an OpenStack cloud
- Choosing between Horizon and Skyline for a new deployment
- Configuring multi-domain support, SSO, or custom branding
- Adding custom panels or extending the dashboard
- Troubleshooting session management or API proxy issues
- Understanding Django plugin architecture or Vue.js SPA patterns

## Horizon vs Skyline Comparison

| Feature | Horizon | Skyline |
|---|---|---|
| Maturity | Very mature (since 2012) | Newer (since 2021) |
| Framework | Django + Python (server-side rendering) | Vue.js 3 (SPA) + FastAPI (async Python backend) |
| Plugin ecosystem | Large — many third-party panels (Trove, Manila, Designate, Watcher, etc.) | Growing — fewer third-party integrations |
| Performance | Heavier (synchronous server-side rendering) | Lighter (SPA with async API calls, faster perceived load) |
| Service coverage | Complete — panels for every major OpenStack service | Core services (Nova, Neutron, Cinder, Glance, Keystone, Swift) |
| Customization | Extensive — panels, themes, custom tables/forms/workflows | Moderate — custom logos, color themes |
| API versions | Supports legacy and current API versions | Current API versions only |
| Multi-domain support | Full (configurable per deployment) | Basic |
| SSO / WebSSO | Full Keystone federation + WebSSO | Via Keystone token; no direct WebSSO UI |
| Packaging | Python package, runs with Apache/mod_wsgi | Container-native (Docker/Podman), also systemd |
| Upstream status | Official OpenStack project | Official OpenStack project |

## Sub-Files

| File | What It Covers |
|---|---|
| [horizon/README.md](horizon/README.md) | Horizon overview, dependencies, quick reference |
| [horizon/operations.md](horizon/operations.md) | `local_settings.py` configuration, multi-domain, SSO, branding, deployment |
| [horizon/internals.md](horizon/internals.md) | Django architecture, Dashboard/Panel hierarchy, plugin system, custom panel development |
| [skyline/README.md](skyline/README.md) | Skyline overview, dependencies, quick reference |
| [skyline/operations.md](skyline/operations.md) | `skyline.yaml` configuration, container/systemd deployment, Nginx reverse proxy, TLS, branding |
| [skyline/internals.md](skyline/internals.md) | Vue.js 3 frontend, FastAPI backend, API proxy pattern, extension mechanism, development setup |

## Choosing a Dashboard

**Choose Horizon when:**
- You need panels for less-common OpenStack services (Trove, Manila, Designate, Watcher, Ironic)
- You need a large existing plugin ecosystem or third-party vendor panels
- You need full multi-domain and WebSSO support
- You are upgrading an existing Horizon deployment
- Your team has Django/Python expertise for customization

**Choose Skyline when:**
- You are building a new deployment and want a modern, fast UI
- Your service scope is limited to core services
- You prefer container-native deployment
- You want a lighter-weight dashboard with a better perceived end-user experience
- Your team has frontend (Vue.js) expertise

Both dashboards can coexist on the same cloud — point them at the same Keystone endpoint.

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

- Horizon release notes: https://docs.openstack.org/releasenotes/horizon/
- Skyline API server: https://opendev.org/openstack/skyline-apiserver
- Skyline Console: https://opendev.org/openstack/skyline-console
