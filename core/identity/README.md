# Identity — Keystone

Keystone is the OpenStack Identity service. It is the **first service deployed** in any OpenStack cloud and the **central dependency** for every other service. It provides authentication, authorization, service discovery (via the catalog), and federation.

## When to Read This Skill

- Deploying Keystone for the first time
- Managing users, projects, groups, roles, or domains
- Configuring the service catalog and endpoints
- Setting up federated identity (SAML2, OIDC, K2K)
- Diagnosing authentication or authorization failures
- Rotating Fernet keys or credential encryption keys
- Understanding token lifecycle, middleware, or RBAC enforcement
- Extending Keystone with custom auth plugins or identity backends

## Sub-Files

| File | What It Covers |
|---|---|
| [architecture.md](architecture.md) | Keystone components, identity model (domains/projects/users/groups/roles), token providers (Fernet, JWS), application credentials, service catalog, trusts, RBAC scopes |
| [operations.md](operations.md) | CLI and API: create/manage users, projects, roles, endpoints, catalog, tokens, application credentials, bootstrap, DB sync, key management, config reference |
| [internals.md](internals.md) | Token lifecycle, Fernet key rotation, auth middleware pipeline, identity/assignment/resource/catalog backends, credential encryption, token validation caching |
| [federation.md](federation.md) | Identity providers, SAML2 walkthrough, OIDC walkthrough, Keystone-to-Keystone federation, mapping rules, shadow users |

## Quick Reference

```bash
# Issue a token (verify auth works)
openstack token issue

# Create a project
openstack project create engineering --domain Default --description "Engineering team"

# Create a user and assign a role
openstack user create alice --domain Default --password s3cur3P@ss --email alice@example.com
openstack role add --project engineering --user alice member

# List the service catalog endpoints
openstack endpoint list

# Rotate Fernet keys (run on all Keystone nodes)
keystone-manage fernet_rotate --keystone-user keystone --keystone-group keystone
```

## Dependencies

| Dependency | Purpose | Notes |
|---|---|---|
| MariaDB 10.6+ or PostgreSQL 14+ | Keystone database | MariaDB is most common in production |
| Memcached 1.5+ | Token validation cache, dogpile.cache backend | Reduces DB load significantly |
| Apache httpd 2.4+ or Nginx | WSGI server for Keystone API | mod_wsgi or uwsgi; Apache most common |
| Python 3.10+ | Runtime | 3.11 recommended for 2025.x releases |
| `python-openstackclient` | Unified CLI | Wraps the REST API |
| `python-keystoneclient` | Lower-level Python bindings | Used by other services internally |
| `keystonemiddleware` | Auth token middleware | Deployed on every other OpenStack service |

## Releases Covered

This skill covers **OpenStack 2025.1 (Epoxy)**, **2025.2**, and **2026.1**.

Keystone release notes: https://docs.openstack.org/releasenotes/keystone/
