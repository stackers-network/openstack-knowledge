# Skyline Internals

## Overall Architecture

Skyline consists of two separate repositories that are deployed together:

```
┌─────────────────────────────────────────────────────────┐
│  skyline-console (Vue.js 3 SPA)                         │
│  Built with Vite → static files (HTML/JS/CSS)           │
└───────────────────────┬─────────────────────────────────┘
                        │  Nginx serves static files
                        │  Nginx proxies /api/* requests
                        ▼
┌─────────────────────────────────────────────────────────┐
│  skyline-apiserver (FastAPI / Python / asyncio)         │
│  ┌────────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │  Auth Router   │  │  Proxy Layer │  │  Settings   │ │
│  │  /login /token │  │  /api/proxy/ │  │  /settings  │ │
│  └────────────────┘  └──────┬───────┘  └─────────────┘ │
│                             │ httpx (async HTTP client) │
└─────────────────────────────┼───────────────────────────┘
                              │
            ┌─────────────────┼──────────────────┐
            ▼                 ▼                  ▼
       Keystone            Nova API         Neutron API
       (auth)           (compute)          (networking)
```

Skyline does **not** have read or write access to any OpenStack service database (Nova DB, Neutron DB, etc.). All data access goes through the service REST APIs, just like any other OpenStack client.

## skyline-console (Vue.js 3 Frontend)

### Technology Stack

| Component | Technology | Notes |
|---|---|---|
| Framework | Vue.js 3 (Composition API) | `<script setup>` syntax throughout |
| Build tool | Vite | Fast HMR in dev; produces optimized bundles |
| State management | Pinia | Replaces Vuex; one store per domain (auth, project, settings) |
| Routing | Vue Router 4 | Client-side routing; all non-API paths return `index.html` |
| UI component library | Ant Design Vue | `ant-design-vue` package |
| HTTP client | Axios | Wraps all API calls; interceptors add auth headers |
| Internationalization | vue-i18n | English and Chinese (zh-CN) built in |

### Project Structure

```
skyline-console/
├── src/
│   ├── main.js                 # App entry point
│   ├── App.vue                 # Root component
│   ├── router/                 # Vue Router route definitions
│   │   └── index.js
│   ├── stores/                 # Pinia stores
│   │   ├── auth.js             # Keystone token, user info, project
│   │   ├── settings.js         # Cloud settings from skyline-apiserver
│   │   └── project.js          # Current project selection
│   ├── api/                    # Axios API clients
│   │   ├── keystone.js
│   │   ├── nova.js
│   │   ├── neutron.js
│   │   ├── cinder.js
│   │   └── ...
│   ├── pages/                  # Top-level page components
│   │   ├── login/
│   │   ├── compute/
│   │   │   ├── Instance/       # Instance list + detail pages
│   │   │   ├── Flavor/
│   │   │   └── Image/
│   │   ├── network/
│   │   ├── storage/
│   │   └── identity/
│   ├── components/             # Shared UI components
│   │   ├── Tables/
│   │   ├── Forms/
│   │   ├── Charts/
│   │   └── Modals/
│   └── locales/                # i18n translation files
│       ├── en.json
│       └── zh.json
├── vite.config.js
├── package.json
└── index.html
```

### Authentication Flow in the Frontend

1. User visits Skyline → Vue Router checks `auth` store for a valid token
2. If no token, redirect to `/login`
3. Login page POSTs credentials to `POST /api/openstack/skyline/api/v1/login`
4. skyline-apiserver exchanges credentials for a Keystone token, stores it server-side, returns a session cookie
5. Subsequent API requests from the frontend include the session cookie; skyline-apiserver injects the Keystone token into proxied OpenStack API calls
6. On logout, `DELETE /api/openstack/skyline/api/v1/logout` revokes the session

### API Call Pattern

All OpenStack API calls from the frontend go through the skyline-apiserver proxy:

```
Frontend: GET /api/openstack/proxy/nova/v2.1/{project_id}/servers
                         ↓
skyline-apiserver fetches: GET https://nova.example.com:8774/v2.1/{project_id}/servers
  (with X-Auth-Token header added from the server-side session)
```

The frontend never holds the Keystone token directly — it only holds the session cookie. The token lives in the skyline-apiserver session store (database or Redis).

### Build the Frontend

```bash
git clone https://opendev.org/openstack/skyline-console
cd skyline-console

# Install Node.js dependencies
npm install

# Development server (with hot module replacement)
npm run dev
# Opens at http://localhost:8080 (proxies /api to a real skyline-apiserver)

# Production build
npm run build
# Output: dist/ directory (copy to /var/www/skyline)

# Preview production build locally
npm run preview
```

Configure the dev server proxy in `vite.config.js`:

```javascript
// vite.config.js
export default {
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:9999',
        changeOrigin: true,
      },
    },
  },
}
```

## skyline-apiserver (FastAPI Backend)

### Technology Stack

| Component | Technology | Notes |
|---|---|---|
| Web framework | FastAPI 0.100+ | Async; uses Pydantic v2 for request/response models |
| ASGI server | Uvicorn (via Gunicorn workers) | `uvicorn.workers.UvicornWorker` for multi-process |
| Async HTTP client | httpx (async) | Used to proxy requests to OpenStack service APIs |
| ORM | SQLAlchemy 2.0 (async) | `asyncmy` driver for MySQL; `asyncpg` for PostgreSQL |
| Database migrations | Alembic | `skyline-apiserver db-sync` runs `alembic upgrade head` |
| Configuration | Pydantic Settings | Reads from `skyline.yaml` via a custom loader |

### Project Structure

```
skyline_apiserver/
├── main.py                     # FastAPI app factory; mounts routers
├── config.py                   # Pydantic settings model (skyline.yaml → Python)
├── routers/
│   ├── login.py                # POST /login, DELETE /logout
│   ├── openstack_proxy.py      # GET|POST|PUT|DELETE /proxy/* (the API proxy)
│   ├── settings.py             # GET /settings (returns skyline.yaml UI settings)
│   ├── contrib.py              # Admin-only endpoints (service management)
│   └── utils.py                # Health check, version endpoint
├── core/
│   ├── keystone.py             # Keystone token exchange, catalog parsing
│   ├── session.py              # Server-side session management (DB or Redis)
│   └── policy.py               # Skyline-level policy checks (system admin vs member)
├── models/
│   ├── session.py              # SQLAlchemy ORM models for sessions
│   └── ...
├── schemas/
│   ├── login.py                # Pydantic request/response schemas
│   └── settings.py
└── db.py                       # Async SQLAlchemy engine and session factory
```

### API Proxy Implementation

The core of skyline-apiserver is the transparent proxy. When the frontend calls:

```
GET /api/openstack/proxy/nova/v2.1/{project_id}/servers
```

The proxy router:

1. Extracts the service name (`nova`) from the URL path
2. Looks up the Nova endpoint URL from the Keystone service catalog (cached in the session)
3. Retrieves the Keystone token from the server-side session
4. Forwards the request using `httpx.AsyncClient`:

```python
# Simplified proxy logic from skyline_apiserver/routers/openstack_proxy.py
async def proxy_request(service: str, path: str, request: Request, session: Session):
    token = session.keystone_token
    endpoint = get_endpoint_from_catalog(session.service_catalog, service)

    async with httpx.AsyncClient(verify=settings.openstack.cafile) as client:
        response = await client.request(
            method=request.method,
            url=f"{endpoint}/{path}",
            headers={
                **{k: v for k, v in request.headers.items()
                   if k.lower() not in ("host", "x-auth-token")},
                "X-Auth-Token": token,
            },
            content=await request.body(),
            params=request.query_params,
            timeout=300.0,
        )

    return Response(
        content=response.content,
        status_code=response.status_code,
        headers=dict(response.headers),
        media_type=response.headers.get("content-type"),
    )
```

5. Streams the response back to the frontend unchanged

This means skyline-apiserver is essentially a token-injecting proxy — it adds no OpenStack business logic of its own for data plane operations.

### Session Management

Sessions are stored in the Skyline database (default) or Redis. The session record holds:

| Field | Content |
|---|---|
| `session_id` | UUID (stored in browser cookie `session`) |
| `user_id` | Keystone user UUID |
| `project_id` | Currently selected project UUID |
| `keystone_token` | The Keystone token string |
| `token_expiry` | Token expiration timestamp |
| `service_catalog` | JSON-serialized Keystone service catalog |
| `created_at` | Session creation time |
| `updated_at` | Last activity time |

Sessions expire when the Keystone token expires. Skyline automatically refreshes the token if the user is active within the last `token_expiry - 300` seconds.

### Configuration Loading

`skyline.yaml` is parsed by a Pydantic Settings model at startup:

```python
# skyline_apiserver/config.py (simplified)
from pydantic import BaseSettings
from typing import List

class OpenStackSettings(BaseSettings):
    keystone_url: str
    default_region: str = "RegionOne"
    system_admin_roles: List[str] = ["admin"]
    ssl_verify: bool = True
    cafile: str = ""

class DefaultSettings(BaseSettings):
    database_url: str
    secret_key: str
    log_level: str = "INFO"
    workers: int = 4
    bind_address: str = "127.0.0.1"
    bind_port: int = 9999

class Settings(BaseSettings):
    default: DefaultSettings
    openstack: OpenStackSettings

    class Config:
        env_file = None  # Only reads from skyline.yaml

def load_settings(path="/etc/skyline/skyline.yaml") -> Settings:
    import yaml
    with open(path) as f:
        data = yaml.safe_load(f)
    return Settings(**data)
```

### Database Migrations

Skyline uses Alembic for schema management. The `db-sync` command runs pending migrations:

```bash
# Run all pending migrations (upgrade to head)
skyline-apiserver db-sync

# Show current migration revision
skyline-apiserver db-sync --check

# Generate a new migration (for Skyline developers)
cd skyline-apiserver/
alembic revision --autogenerate -m "add_custom_field"
```

## Extension Mechanism

Skyline does not have a plugin system comparable to Horizon's `enabled/` directory. Extensions are achieved by:

### 1. Adding New API Routes (Backend)

Fork `skyline-apiserver`, add a new router module, and mount it in `main.py`:

```python
# skyline_apiserver/routers/custom.py
from fastapi import APIRouter
router = APIRouter(prefix="/api/openstack/skyline/api/v1/custom")

@router.get("/my-data")
async def get_my_data():
    return {"custom": "data"}

# skyline_apiserver/main.py
from .routers import custom
app.include_router(custom.router)
```

### 2. Adding New UI Pages (Frontend)

Add a new page component in `src/pages/`, register a route in `src/router/index.js`, and add a menu entry:

```javascript
// src/router/index.js
{
  path: '/custom',
  name: 'CustomPage',
  component: () => import('@/pages/custom/index.vue'),
  meta: { title: 'Custom Resource', icon: 'icon-custom' }
}
```

### 3. Injecting Custom Branding Without Rebuilding

Place a custom CSS file at the Nginx-served root and inject it via Nginx's `sub_filter` module:

```nginx
# In the Nginx server block
sub_filter '</head>' '<link rel="stylesheet" href="/custom.css"></head>';
sub_filter_once on;
```

## Local Development Setup

```bash
# Clone both repos
git clone https://opendev.org/openstack/skyline-apiserver
git clone https://opendev.org/openstack/skyline-console

# --- Backend ---
cd skyline-apiserver
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"

# Create development config
cp etc/skyline.yaml.sample /etc/skyline/skyline.yaml
# Edit /etc/skyline/skyline.yaml with your dev cloud's settings

# Create a local DB
mysql -u root -e "CREATE DATABASE skyline_dev; GRANT ALL ON skyline_dev.* TO 'skyline'@'localhost' IDENTIFIED BY 'dev';"
# Update database_url in skyline.yaml accordingly

skyline-apiserver db-sync

# Start dev server (auto-reload on code changes)
uvicorn skyline_apiserver.main:app --host 127.0.0.1 --port 9999 --reload

# --- Frontend ---
cd ../skyline-console
node --version   # requires Node.js 18+
npm install

# Configure the dev proxy to point at the local apiserver
# (default vite.config.js already proxies /api to localhost:9999)

npm run dev
# Dashboard available at http://localhost:8080
```

## Observability

### Structured Logging

skyline-apiserver logs in JSON format when `log_level = INFO` or higher. Example log entry:

```json
{
  "timestamp": "2025-04-02T10:23:44.123Z",
  "level": "INFO",
  "message": "Proxied request",
  "method": "GET",
  "path": "/api/openstack/proxy/nova/v2.1/proj-123/servers",
  "status_code": 200,
  "duration_ms": 45,
  "user_id": "user-abc",
  "project_id": "proj-123"
}
```

### Metrics

FastAPI exposes Prometheus metrics when the `prometheus-fastapi-instrumentator` package is installed:

```python
# In main.py (requires: pip install prometheus-fastapi-instrumentator)
from prometheus_fastapi_instrumentator import Instrumentator
Instrumentator().instrument(app).expose(app)
# Metrics available at /metrics
```

Scrape from Prometheus:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'skyline-apiserver'
    static_configs:
      - targets: ['127.0.0.1:9999']
    metrics_path: /metrics
```
