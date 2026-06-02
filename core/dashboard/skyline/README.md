# Skyline

Skyline is the newer OpenStack dashboard, introduced as an official project in 2021. It replaces the traditional server-side-rendered Django approach with a **Single Page Application** architecture: a **Vue.js 3** frontend (skyline-console) communicating with a **FastAPI** backend (skyline-apiserver) which proxies requests to the underlying OpenStack service APIs.

Skyline is lighter, faster in perceived user experience, and easier to containerize than Horizon. Its tradeoff is a smaller plugin ecosystem and narrower service coverage.

## When to Read This Skill

- Deploying Skyline as a Horizon alternative on a new cloud
- Configuring `skyline.yaml` for production
- Running Skyline in containers (Docker/Podman) or on bare metal
- Setting up Nginx as a reverse proxy in front of Skyline
- Customizing branding (logo, title, colors)
- Understanding the API proxy pattern and how Skyline routes requests
- Extending or developing Skyline features

## Sub-Files

| File | What It Covers |
|---|---|
| [operations.md](operations.md) | `skyline.yaml` settings, container and systemd deployment, Nginx configuration, TLS, branding |
| [internals.md](internals.md) | Vue.js 3 frontend, FastAPI backend, API proxy pattern, extension mechanism, development setup |

## Quick Reference

```bash
# Pull and run Skyline (Docker)
docker pull quay.io/skyline/skyline-apiserver:latest

docker run -d \
  --name skyline \
  -p 9999:9999 \
  -v /etc/skyline/skyline.yaml:/etc/skyline/skyline.yaml:ro \
  quay.io/skyline/skyline-apiserver:latest

# Check Skyline API health
curl -s http://localhost:9999/api/openstack/skyline/api/v1/

# View logs
docker logs -f skyline
```

## Architecture at a Glance

```
Browser (Vue.js 3 SPA)
      │  HTTPS
      ▼
  Nginx (static files + reverse proxy)
      │
      ▼
skyline-apiserver (FastAPI / Python)
      │  HTTP (internal)
      ├──► Keystone (auth)
      ├──► Nova (compute API)
      ├──► Neutron (networking API)
      ├──► Cinder (volumes API)
      ├──► Glance (images API)
      └──► ... (other services)
```

The Vue.js frontend is served as static files by Nginx. All API calls from the frontend go to the skyline-apiserver, which acts as an authenticated proxy — it holds the user's Keystone token in a server-side session and forwards requests to the appropriate OpenStack service.

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| Keystone | Authentication and service catalog | All user sessions start with a Keystone token |
| Nginx | Serve static frontend files; reverse proxy to skyline-apiserver | Port 443 typically |
| MariaDB 10.6+ or PostgreSQL 14+ | Skyline session/cache database | Skyline uses its own DB, not OpenStack service DBs |
| Python 3.10+ | Runtime for skyline-apiserver | 3.11 recommended |
| Redis (optional) | Alternative session/cache backend | Can substitute for the database cache |

## Ports

| Port | Service | Notes |
|---|---|---|
| 9999 | skyline-apiserver (internal) | FastAPI app; proxied by Nginx |
| 443 | Nginx (external) | HTTPS for frontend + API |

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

- skyline-apiserver: https://opendev.org/openstack/skyline-apiserver
- skyline-console: https://opendev.org/openstack/skyline-console
