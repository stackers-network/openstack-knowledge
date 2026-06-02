# Skyline Operations

## Configure skyline.yaml

The primary configuration file is `/etc/skyline/skyline.yaml`. Create the directory and file before starting Skyline:

```bash
mkdir -p /etc/skyline
```

Full production-ready `skyline.yaml`:

```yaml
# /etc/skyline/skyline.yaml

default:
  # Database URL for Skyline's own session/cache database
  # Create the database and user first:
  #   CREATE DATABASE skyline;
  #   CREATE USER 'skyline'@'10.0.0.10' IDENTIFIED BY 'skyline_db_pass';
  #   GRANT ALL PRIVILEGES ON skyline.* TO 'skyline'@'10.0.0.10';
  database_url: "mysql+aiomysql://skyline:skyline_db_pass@10.0.0.10:3306/skyline"

  # Secret key for signing session tokens — generate with:
  # python3 -c "import secrets; print(secrets.token_hex(32))"
  secret_key: "a9f8b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1"

  # Log level: DEBUG, INFO, WARNING, ERROR
  log_level: "INFO"

  # Log file path (empty = stdout, useful for containers)
  log_dir: ""

  # Enable/disable access logging
  access_log: true

  # Number of API worker processes (default: number of CPUs)
  workers: 4

  # Bind address for the skyline-apiserver internal HTTP server
  # (proxied by Nginx; do not expose this port directly)
  bind_address: "127.0.0.1"
  bind_port: 9999

openstack:
  # Keystone v3 endpoint (internal or public)
  keystone_url: "https://keystone.example.com:5000/v3"

  # Default Keystone domain for user authentication
  default_region: "RegionOne"

  # Roles that grant system-admin access in Skyline
  # Users with these roles in the system scope see all-tenant resources
  system_admin_roles:
    - "admin"

  # Roles that grant read access in Skyline
  system_reader_roles:
    - "reader"

  # Default Keystone domain (shown on the login page)
  keystone_default_domain: "Default"

  # If true, shows the Domain field on the login page
  keystone_multi_region_map:
    - default_region: RegionOne
      keystone_url: "https://keystone.example.com:5000/v3"

  # SSL/TLS settings for outbound calls from skyline-apiserver to OpenStack APIs
  ssl_verify: true
  cafile: "/etc/ssl/certs/openstack-ca.crt"

  # List of reclaim tasks (optional — cleanup expired sessions)
  reclaim_instance_interval: 604800   # 7 days in seconds

setting:
  # UI settings
  base_settings:
    # Page title shown in the browser tab
    base_title: "ACME Cloud"
    # Logo URL (relative to the static file base, or an absolute URL)
    base_logo: "/static/img/logo.svg"
    # Whether to display the Skyline API console panel
    enable_host_aggregates_management: false
    enable_compute_service_management: true
    enable_network_service_management: true
    enable_volume_service_management: true
```

Initialize the database:

```bash
# Sync database schema
skyline-apiserver db-sync
```

## Deploy with Docker

### Pull the Images

```bash
# skyline-apiserver (backend)
docker pull quay.io/skyline/skyline-apiserver:latest

# skyline-console (pre-built frontend static files)
docker pull quay.io/skyline/skyline-console:latest
```

### Run the Backend

```bash
docker run -d \
  --name skyline-apiserver \
  --restart unless-stopped \
  -p 127.0.0.1:9999:9999 \
  -v /etc/skyline/skyline.yaml:/etc/skyline/skyline.yaml:ro \
  -v /etc/ssl/certs/openstack-ca.crt:/etc/ssl/certs/openstack-ca.crt:ro \
  quay.io/skyline/skyline-apiserver:latest \
  gunicorn -b 0.0.0.0:9999 --workers 4 --worker-class uvicorn.workers.UvicornWorker \
  "skyline_apiserver.main:app"
```

### Extract the Static Frontend Files

```bash
# Copy the pre-built Vue.js assets out of the console image
docker create --name skyline-console-tmp quay.io/skyline/skyline-console:latest
docker cp skyline-console-tmp:/skyline/static /var/www/skyline
docker rm skyline-console-tmp
```

## Deploy with Docker Compose

Create `/opt/skyline/docker-compose.yaml`:

```yaml
version: "3.8"

services:
  skyline-apiserver:
    image: quay.io/skyline/skyline-apiserver:latest
    restart: unless-stopped
    ports:
      - "127.0.0.1:9999:9999"
    volumes:
      - /etc/skyline/skyline.yaml:/etc/skyline/skyline.yaml:ro
      - /etc/ssl/certs/openstack-ca.crt:/etc/ssl/certs/openstack-ca.crt:ro
    command: >
      gunicorn -b 0.0.0.0:9999
      --workers 4
      --worker-class uvicorn.workers.UvicornWorker
      "skyline_apiserver.main:app"
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost:9999/api/openstack/skyline/api/v1/"]
      interval: 30s
      timeout: 10s
      retries: 3
```

Start:

```bash
docker compose -f /opt/skyline/docker-compose.yaml up -d
```

## Deploy with Podman and systemd

```bash
# Create the container
podman create \
  --name skyline-apiserver \
  -p 127.0.0.1:9999:9999 \
  -v /etc/skyline/skyline.yaml:/etc/skyline/skyline.yaml:ro \
  quay.io/skyline/skyline-apiserver:latest

# Generate systemd unit
podman generate systemd --new --name skyline-apiserver \
  > /etc/systemd/system/skyline-apiserver.service

systemctl daemon-reload
systemctl enable --now skyline-apiserver
```

## Deploy on Bare Metal (systemd)

Install directly with pip:

```bash
pip3 install skyline-apiserver

# Initialize the database
skyline-apiserver db-sync

# Create the systemd unit
cat > /etc/systemd/system/skyline-apiserver.service <<'EOF'
[Unit]
Description=Skyline API Server
After=network.target

[Service]
User=skyline
Group=skyline
ExecStart=/usr/local/bin/gunicorn \
  -b 127.0.0.1:9999 \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --access-logfile /var/log/skyline/access.log \
  --error-logfile /var/log/skyline/error.log \
  "skyline_apiserver.main:app"
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

useradd -r -s /sbin/nologin skyline
mkdir -p /var/log/skyline && chown skyline:skyline /var/log/skyline

systemctl daemon-reload
systemctl enable --now skyline-apiserver
```

## Configure Nginx Reverse Proxy

Nginx serves the static Vue.js files and proxies API calls to skyline-apiserver:

```nginx
# /etc/nginx/sites-available/skyline

# HTTP → HTTPS redirect
server {
    listen 80;
    server_name skyline.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name skyline.example.com;

    ssl_certificate     /etc/ssl/certs/skyline.crt;
    ssl_certificate_key /etc/ssl/private/skyline.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;
    ssl_session_cache   shared:SSL:10m;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options SAMEORIGIN always;
    add_header X-Content-Type-Options nosniff always;

    # Serve the pre-built Vue.js SPA static files
    root /var/www/skyline;
    index index.html;

    # Proxy API calls to skyline-apiserver
    location /api/ {
        proxy_pass http://127.0.0.1:9999;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
        proxy_buffering off;
    }

    # Skyline API health check
    location /api/openstack/skyline/ {
        proxy_pass http://127.0.0.1:9999;
        proxy_set_header Host $host;
    }

    # Serve the SPA — all non-API routes return index.html (client-side routing)
    location / {
        try_files $uri $uri/ /index.html;
        expires 1h;
        add_header Cache-Control "public, no-transform";
    }

    # Cache static assets aggressively (content-hash filenames from Vite build)
    location ~* \.(js|css|woff2?|ttf|eot|svg|png|jpg|ico)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    access_log /var/log/nginx/skyline-access.log;
    error_log  /var/log/nginx/skyline-error.log;
}
```

Enable and test:

```bash
nginx -t
ln -s /etc/nginx/sites-available/skyline /etc/nginx/sites-enabled/
systemctl reload nginx
```

## Configure SSL/TLS for Skyline Outbound Calls

Skyline-apiserver makes HTTPS calls to OpenStack service APIs. Configure the CA trust in `skyline.yaml`:

```yaml
openstack:
  ssl_verify: true
  cafile: "/etc/ssl/certs/openstack-ca.crt"
```

If running in a container, mount the CA certificate:

```bash
docker run -d \
  -v /etc/ssl/certs/openstack-ca.crt:/etc/ssl/certs/openstack-ca.crt:ro \
  ...
```

Disable TLS verification only for testing:

```yaml
openstack:
  ssl_verify: false  # Never use in production
```

## Custom Logo and Branding

### Replace the Default Logo

Place your custom logo at the path referenced in `skyline.yaml`:

```bash
# Copy logo to the static files directory
cp /tmp/acme-logo.svg /var/www/skyline/static/img/logo.svg
```

Or use an absolute URL in `skyline.yaml` if the logo is hosted externally:

```yaml
setting:
  base_settings:
    base_title: "ACME Cloud Dashboard"
    base_logo: "https://cdn.acme.example.com/logo.svg"
```

### Color Theme

The color palette is defined in the built Vue.js bundle. To customize colors, you must rebuild the frontend from source (see [internals.md](internals.md)).

For a quick custom CSS override, inject a custom stylesheet via Nginx:

```nginx
# In the server block, after the root directive:
location = /custom.css {
    alias /var/www/skyline/custom.css;
}
```

Add a custom.css file:

```css
/* /var/www/skyline/custom.css */
:root {
  --primary-color: #1a73e8;
  --header-bg: #003580;
}
```

## Enable the API Console / Logs Panel

The Skyline admin panel provides a console for direct API call exploration. Enable it for system-admin users:

```yaml
# skyline.yaml
setting:
  base_settings:
    enable_compute_service_management: true
    enable_network_service_management: true
    enable_volume_service_management: true
    enable_host_aggregates_management: true
```

## Verify Skyline Health

```bash
# Check the apiserver health endpoint
curl -s http://127.0.0.1:9999/api/openstack/skyline/api/v1/ | python3 -m json.tool

# Check Nginx is serving the SPA
curl -s -I https://skyline.example.com/ | head -5

# Check database connectivity (container)
docker exec skyline-apiserver skyline-apiserver db-sync --check

# View apiserver logs (container)
docker logs -f skyline-apiserver

# View Nginx logs
tail -f /var/log/nginx/skyline-error.log
```

## Upgrade Skyline

For container deployments:

```bash
# Pull the new image
docker pull quay.io/skyline/skyline-apiserver:latest
docker pull quay.io/skyline/skyline-console:latest

# Stop the running container
docker stop skyline-apiserver
docker rm skyline-apiserver

# Run database migrations before starting the new version
docker run --rm \
  -v /etc/skyline/skyline.yaml:/etc/skyline/skyline.yaml:ro \
  quay.io/skyline/skyline-apiserver:latest \
  skyline-apiserver db-sync

# Extract the new static files
docker create --name skyline-console-tmp quay.io/skyline/skyline-console:latest
docker cp skyline-console-tmp:/skyline/static /var/www/skyline
docker rm skyline-console-tmp

# Start the new apiserver
docker run -d --name skyline-apiserver ...  # same as initial deploy
```
