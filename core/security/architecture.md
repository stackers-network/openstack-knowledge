# Barbican Architecture

## Component Overview

Barbican consists of three processes and a shared database:

```
  Clients (openstack CLI, Nova, Octavia, Swift)
          │
          ▼  HTTPS :9311
  ┌───────────────────┐
  │   barbican-api    │  WSGI app (Apache/Nginx)
  │  (REST API layer) │  Validates Keystone tokens
  └─────────┬─────────┘  Enforces oslo.policy RBAC
            │  oslo.messaging (RabbitMQ)
            ▼
  ┌───────────────────┐
  │  barbican-worker  │  Async task processor
  │  (async tasks)    │  Key generation, cert signing
  └─────────┬─────────┘
            │
  ┌─────────┴──────────────────────────┐
  │           Plugin Layer             │
  │  ┌──────────┐  ┌─────────────────┐ │
  │  │  Crypto  │  │  Secret Store   │ │
  │  │  Plugin  │  │  Plugin         │ │
  │  └──────────┘  └─────────────────┘ │
  └────────────────────────────────────┘
            │
  ┌─────────┴─────────┐
  │  MariaDB / PgSQL  │  Metadata + encrypted blobs
  └───────────────────┘

  ┌───────────────────────────┐
  │ barbican-keystone-listener│  oslo.messaging consumer
  │ (project delete events)   │  Purges project secrets on
  └───────────────────────────┘  project delete
```

### barbican-api

The WSGI application that exposes the Barbican REST API on port 9311. It:

- Authenticates requests via `keystonemiddleware.auth_token`
- Enforces oslo.policy RBAC (`/etc/barbican/policy.yaml`)
- Validates request payloads and dispatches to the appropriate plugin
- Handles synchronous operations directly (secret store/retrieve/delete)
- Queues asynchronous operations (orders) to RabbitMQ for the worker

Key configuration file: `/etc/barbican/barbican.conf`

Typical Apache virtual host snippet:

```apache
<VirtualHost *:9311>
    WSGIDaemonProcess barbican-api processes=4 threads=1 user=barbican group=barbican
    WSGIProcessGroup barbican-api
    WSGIScriptAlias / /usr/bin/barbican-wsgi-api
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLog /var/log/barbican/barbican-api.log
    CustomLog /var/log/barbican/barbican-api-access.log combined
</VirtualHost>
```

### barbican-worker

A long-running daemon that consumes tasks from the RabbitMQ queue. It handles:

- **Key generation orders**: Requests the crypto plugin to generate a new symmetric or asymmetric key, then stores the result as a secret
- **Certificate signing requests (CSR)**: Dispatches to the configured certificate plugin (Dogtag CA, DigiCert, Let's Encrypt via ACME, or a custom CA)
- **Certificate orders**: Full certificate lifecycle (generate key pair, create CSR, submit to CA, retrieve signed certificate)

Start the worker:

```bash
barbican-worker --config-file /etc/barbican/barbican.conf
```

### barbican-keystone-listener

A separate Oslo messaging consumer that listens for Keystone notifications on the `notifications` exchange. When a project (tenant) is deleted in Keystone, the listener triggers cleanup of all secrets and containers owned by that project.

This process is optional but strongly recommended in production to avoid orphaned secrets accumulating in the database.

Start the listener:

```bash
barbican-keystone-listener --config-file /etc/barbican/barbican.conf
```

## Secret Types

Barbican stores secrets as **opaque blobs** with a declared content type. The `secret_type` field categorizes what is stored, enabling consumers to interpret the payload correctly.

| Secret Type | Content | Typical Use |
|---|---|---|
| `symmetric` | Raw bytes of a symmetric key | AES volume encryption, HMAC keys |
| `public` | PEM-encoded public key | RSA/EC public key distribution |
| `private` | PEM-encoded private key | RSA/EC private key storage |
| `certificate` | PEM-encoded X.509 certificate | TLS/HTTPS certificates |
| `passphrase` | Arbitrary text (UTF-8) | Passwords, tokens, API keys |
| `opaque` | Any binary blob | Default when type is not specified |

All payloads are encrypted at rest using the configured crypto plugin before being written to the database. The `secret_type` is stored as plaintext metadata.

## Containers

A **container** groups related secrets into a logical unit. Containers do not encrypt anything — they only hold references (HREFs) to individual secrets that must already exist.

| Container Type | Required Secrets | Optional Secrets | Use Case |
|---|---|---|---|
| `generic` | None (free-form) | Any named secrets | Arbitrary groupings |
| `rsa` | `public_key`, `private_key` | `private_key_passphrase` | RSA key pair management |
| `certificate` | `certificate` | `private_key`, `private_key_passphrase`, `intermediates` | TLS certificate bundles for Octavia/load balancers |

Creating a container validates that referenced secrets exist and that the required secret names are present for typed containers.

## Orders

An **order** is an asynchronous request to Barbican to generate or retrieve a secret on behalf of the caller. The caller does not supply the key material — Barbican generates it.

| Order Type | What It Does | Resulting Secret Type |
|---|---|---|
| `key` | Generates a new symmetric key (AES, HMAC) | `symmetric` |
| `asymmetric` | Generates a new key pair (RSA, EC, DSA) | Container with `public_key` + `private_key` |
| `certificate` | Signs a CSR or generates a full key+cert | Container with `certificate` + `private_key` |

Order state machine:

```
PENDING → PROCESSING → ACTIVE    (success)
                     → ERROR     (failure — inspect order for error detail)
```

Poll order status with `openstack secret order get <ORDER_HREF>` until the status is `ACTIVE`.

## Transport Keys

Transport keys enable **client-side encryption of secret payloads during upload** — the secret is encrypted with the server's public transport key before being sent over the wire. This prevents the API server from ever seeing the plaintext payload in memory (useful in untrusted network paths or for compliance).

Flow:
1. Client retrieves the barbican-api's transport public key: `GET /v1/transport_keys`
2. Client wraps (encrypts) the secret payload with the transport public key using RSA-OAEP
3. Client submits the wrapped payload with `Content-Type: application/octet-stream` and the `trans_wrapped_key` field set to the transport key UUID
4. barbican-api unwraps using its transport private key before passing to the crypto plugin

Transport keys are rarely used in standard deployments but are available for high-security environments.

## Database Schema (Logical)

| Table | Purpose |
|---|---|
| `secrets` | Secret metadata (name, algorithm, bit_length, expiration, secret_type) |
| `secret_store_metadata` | Encrypted blobs or references to external store |
| `containers` | Container metadata (name, type) |
| `container_secrets` | Junction: container → secret (with role name) |
| `orders` | Order requests and current state |
| `order_metadata` | Key/value pairs for order parameters |
| `project_quotas` | Per-project quota overrides |
| `kek_data` | Key Encryption Key metadata per project (used by simple_crypto) |
| `encrypted_data` | AES-GCM encrypted ciphertext for each secret |

## Configuration: Key Sections

```ini
# /etc/barbican/barbican.conf

[DEFAULT]
log_file = /var/log/barbican/barbican.log
bind_host = 0.0.0.0
bind_port = 9311

[database]
connection = mysql+pymysql://barbican:barbicandb_pass@10.0.0.10/barbican
max_pool_size = 10
max_overflow = 20

[keystone_authtoken]
www_authenticate_uri = https://keystone.example.com:5000
auth_url = https://keystone.example.com:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = barbican
password = barbican_service_pass
memcached_servers = 10.0.0.10:11211

[oslo_messaging_rabbit]
rabbit_host = 10.0.0.10
rabbit_port = 5672
rabbit_userid = barbican
rabbit_password = rabbit_barbican_pass
rabbit_virtual_host = /barbican

[secretstore]
# Comma-separated list of enabled secret store plugins
enabled_secretstore_plugins = store_crypto

[crypto]
# Comma-separated list of enabled crypto plugins
enabled_crypto_plugins = simple_crypto

[simple_crypto_plugin]
# Master KEK — must be a 32-byte URL-safe base64 string
# Generate with: python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
kek = dGhpcyBpcyBhIHRlc3Qga2V5Zm9yYmFyYmljYW4hCg==

[oslo_policy]
enforce_scope = true
enforce_new_defaults = true
policy_file = /etc/barbican/policy.yaml
```
