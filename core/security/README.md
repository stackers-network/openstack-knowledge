# Security — Barbican + Hardening

This skill covers two related concerns: **Barbican** (the OpenStack Key Manager service, which stores and manages secrets, certificates, and cryptographic keys) and **cross-service security hardening** (TLS, RBAC, audit logging, and network segmentation for the entire OpenStack stack).

## When to Read This Skill

- Deploying Barbican to store secrets, TLS certificates, or symmetric keys
- Integrating Barbican with Nova (volume encryption), Octavia (TLS termination), or Swift (at-rest encryption)
- Generating keys via orders (rather than importing existing secrets)
- Managing certificate authorities and signing requests
- Configuring an HSM backend (PKCS#11 or KMIP)
- Hardening TLS for internal and external OpenStack API endpoints
- Tightening oslo.policy RBAC and enabling scope enforcement
- Setting up audit logging and CADF notifications
- Securing database connections and RabbitMQ message bus

## Sub-Files

| File | What It Covers |
|---|---|
| [architecture.md](architecture.md) | Barbican components (barbican-api, barbican-worker, barbican-keystone-listener), secret types, containers, orders, transport keys |
| [operations.md](operations.md) | CLI and API: secret store/list/get/delete, orders, containers, CAs, quotas |
| [internals.md](internals.md) | Crypto plugins (simple_crypto, PKCS#11, KMIP, Vault), secret store plugin interface, certificate plugins, transport key wrapping |
| [hardening.md](hardening.md) | TLS everywhere, oslo.policy, service accounts, Keystone federation, security groups, network segmentation, audit logging, DB and RabbitMQ security |

## Quick Reference

```bash
# Store a secret
openstack secret store \
  --name my-db-password \
  --payload 'sup3rs3cr3t' \
  --payload-content-type 'text/plain'

# Retrieve a secret payload
openstack secret get <SECRET_HREF> --payload

# Generate a 256-bit AES key via an order
openstack secret order create \
  --name my-aes-key \
  --algorithm aes \
  --bit-length 256 \
  key

# List secrets in the current project
openstack secret list

# Store a TLS certificate bundle as a container
openstack secret container create \
  --name my-tls \
  --type certificate \
  --secret "certificate=https://barbican.example.com/v1/secrets/abc123" \
  --secret "private_key=https://barbican.example.com/v1/secrets/def456"
```

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| Keystone | Authentication and RBAC enforcement | All Barbican API calls validated via Keystone tokens |
| MariaDB 10.6+ or PostgreSQL 14+ | Metadata and encrypted secret storage | Simple-crypto plugin stores encrypted blobs here |
| RabbitMQ 3.12+ | Async task queue for barbican-worker | oslo.messaging; used for order processing |
| Apache httpd 2.4+ or Nginx | WSGI server for barbican-api | Same pattern as other OpenStack services |
| `python-openstackclient` + `python-barbicanclient` | Unified CLI | `openstack secret` commands use barbicanclient |
| PKCS#11 HSM (optional) | Hardware key storage | Luna HSM, Thales, or SoftHSM for testing |
| HashiCorp Vault (optional) | External secret backend | Via `castellan` and Vault plugin |

## Service Registration

```bash
# Register Barbican in the service catalog
openstack service create --name barbican --description "Key Manager Service" key-manager

openstack endpoint create --region RegionOne \
  barbican public https://barbican.example.com:9311
openstack endpoint create --region RegionOne \
  barbican internal https://10.0.0.15:9311
openstack endpoint create --region RegionOne \
  barbican admin https://10.0.0.15:9311
```

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

Barbican release notes: https://docs.openstack.org/releasenotes/barbican/
