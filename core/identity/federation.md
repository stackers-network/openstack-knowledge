# Keystone Federation

Federation lets Keystone accept identities asserted by an external **Identity Provider (IdP)** — such as a corporate SAML2 IdP (Shibboleth, ADFS), an OIDC provider (Keycloak, Okta, Google), or another OpenStack Keystone (K2K). Users authenticate to the IdP; Keystone trusts the assertion and issues its own token.

## Concepts

### Identity Provider (IdP)

An external system that authenticates users and issues signed assertions about their identity and attributes (e.g., email, group membership). Keystone trusts one or more IdPs.

### Service Provider (SP)

In SAML2 federation, the system receiving and consuming assertions. Keystone acts as the SP for incoming federated logins.

### Protocol

A protocol is a named link between an IdP and a mapping within Keystone. Each IdP can have multiple protocols (e.g., `saml2`, `oidc`).

### Mapping

A mapping rule set that translates IdP-supplied attributes (e.g., SAML attributes, OIDC claims) into Keystone local roles, groups, and projects. Mappings define **who gets what access** based on what the IdP says about them.

### Shadow Users

When a federated user logs in for the first time, Keystone **auto-provisions** a local user record in the `shadow_users` table. This is the shadow user. Shadow users:
- Are linked to the federated IdP and protocol
- Have no local password
- Are created with `domain_id` of the federated domain
- Can accumulate role assignments over time (e.g., via group mapping)
- Are updated on each login to reflect the latest IdP attributes

Shadow user creation is controlled by the mapping's `user` local rule. If `type` is `ephemeral`, no persistent shadow user is created (token-only access, no role persistence).

## SAML2 Federation Setup

This walkthrough configures Keystone as a SAML2 Service Provider, trusting an external IdP (e.g., Shibboleth, ADFS, or any SAML2 IdP).

### Install and Configure mod_shib

```bash
# RHEL/CentOS
dnf install shibboleth

# Ubuntu/Debian
apt install libapache2-mod-shib
```

Configure Shibboleth SP at `/etc/shibboleth/shibboleth2.xml`:

```xml
<SPConfig xmlns="urn:mace:shibboleth:3.0:native:sp:config" clockSkew="180">
  <ApplicationDefaults entityID="https://keystone.example.com/shibboleth"
                       REMOTE_USER="eppn persistent-id targeted-id">

    <Sessions lifetime="28800" timeout="3600" relayState="ss:mem"
              checkAddress="false" handlerSSL="false" cookieProps="http">
      <SSO entityID="https://idp.acme-corp.com/idp/shibboleth">
        SAML2
      </SSO>
      <Logout>SAML2 Local</Logout>
      <Handler type="MetadataGenerator" Location="/Metadata" signing="false"/>
      <Handler type="Status" Location="/Status"/>
      <Handler type="Session" Location="/Session" showAttributeValues="false"/>
      <Handler type="DiscoveryFeed" Location="/DiscoFeed"/>
    </Sessions>

    <MetadataProvider type="XML" validate="true"
        url="https://idp.acme-corp.com/idp/shibboleth"
        backingFilePath="idp-metadata.xml" maxRefreshDelay="7200">
    </MetadataProvider>

    <AttributeExtractor type="XML" validate="true" reloadChanges="false"
                        path="attribute-map.xml"/>
    <AttributeFilter type="XML" validate="true" path="attribute-policy.xml"/>
    <CredentialResolver type="File" use="signing"
                        key="sp-key.pem" certificate="sp-cert.pem"/>
  </ApplicationDefaults>
</SPConfig>
```

### Configure Apache for Federation

```apache
# /etc/httpd/conf.d/keystone-federation.conf
<VirtualHost *:5000>
  ServerName keystone.example.com
  WSGIScriptAlias / /usr/bin/keystone-wsgi-public
  WSGIProcessGroup keystone
  WSGIApplicationGroup %{GLOBAL}

  # SAML2 protected endpoint
  <Location /v3/OS-FEDERATION/identity_providers/acme-idp/protocols/saml2/auth>
    AuthType shibboleth
    ShibRequestSetting requireSession 1
    ShibRequireSession On
    ShibExportAssertion Off
    Require valid-user
  </Location>

  # WebSSO callback (for Horizon)
  <Location /v3/auth/OS-FEDERATION/websso/saml2>
    AuthType shibboleth
    ShibRequestSetting requireSession 1
    Require valid-user
  </Location>
</VirtualHost>
```

### Register the IdP in Keystone

```bash
# Create the Identity Provider record
openstack identity provider create acme-idp \
  --remote-id https://idp.acme-corp.com/idp/shibboleth \
  --description "ACME Corp SAML2 IdP"
```

### Create a Mapping

Mappings translate SAML2 attributes into Keystone roles and groups.

```bash
# Write mapping rules to a JSON file
cat > /tmp/acme-saml-mapping.json << 'EOF'
[
  {
    "local": [
      {
        "user": {
          "name": "{0}",
          "email": "{1}"
        }
      },
      {
        "group": {
          "id": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4"
        }
      }
    ],
    "remote": [
      {
        "type": "MELLON_NAME_ID"
      },
      {
        "type": "MELLON_mail"
      },
      {
        "type": "MELLON_isMemberOf",
        "any_one_of": [
          "cn=openstack-users,ou=Groups,dc=acme-corp,dc=com"
        ]
      }
    ]
  },
  {
    "local": [
      {
        "user": {
          "name": "{0}"
        }
      },
      {
        "group": {
          "id": "b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5"
        }
      }
    ],
    "remote": [
      {
        "type": "MELLON_NAME_ID"
      },
      {
        "type": "MELLON_isMemberOf",
        "any_one_of": [
          "cn=openstack-admins,ou=Groups,dc=acme-corp,dc=com"
        ]
      }
    ]
  }
]
EOF

openstack mapping create acme-saml-mapping \
  --rules /tmp/acme-saml-mapping.json
```

(The group IDs `a1b2c3d4...` are pre-existing Keystone groups that have role assignments on projects.)

### Create the Protocol

```bash
openstack federation protocol create saml2 \
  --identity-provider acme-idp \
  --mapping acme-saml-mapping
```

### Test the Federation Flow

```bash
# Authenticate via ECP (non-browser SAML2 profile)
openstack token issue \
  --os-auth-type v3samlpassword \
  --os-auth-url http://10.0.0.10:5000/v3 \
  --os-identity-provider acme-idp \
  --os-protocol saml2 \
  --os-identity-provider-url https://idp.acme-corp.com/idp/profile/SAML2/SOAP/ECP \
  --os-username alice@acme-corp.com \
  --os-password s3cur3P@ss \
  --os-project-name engineering \
  --os-project-domain-name Default
```

## OIDC Federation Setup

This walkthrough configures Keystone as an OIDC Relying Party, trusting an OIDC IdP (e.g., Keycloak at `https://keycloak.example.com`).

### Install mod_auth_openidc

```bash
# RHEL/CentOS
dnf install mod_auth_openidc

# Ubuntu/Debian
apt install libapache2-mod-auth-openidc
```

### Configure mod_auth_openidc

```apache
# /etc/httpd/conf.d/keystone-oidc.conf
OIDCProviderMetadataURL https://keycloak.example.com/realms/openstack/.well-known/openid-configuration
OIDCClientID keystone-sp
OIDCClientSecret oidc_client_secret_here
OIDCRedirectURI http://keystone.example.com:5000/v3/OS-FEDERATION/identity_providers/keycloak-idp/protocols/oidc/auth
OIDCCryptoPassphrase random_passphrase_here
OIDCResponseType code
OIDCScope "openid email profile groups"
OIDCPassClaimsAs environment
OIDCClaimPrefix OIDC_CLAIM_

<Location /v3/OS-FEDERATION/identity_providers/keycloak-idp/protocols/oidc/auth>
  AuthType openid-connect
  Require valid-user
</Location>

<Location /v3/auth/OS-FEDERATION/websso/oidc>
  AuthType openid-connect
  Require valid-user
</Location>
```

### Register the IdP in Keystone

```bash
openstack identity provider create keycloak-idp \
  --remote-id https://keycloak.example.com/realms/openstack \
  --description "Keycloak OIDC IdP"
```

### Create an OIDC Mapping

```bash
cat > /tmp/keycloak-oidc-mapping.json << 'EOF'
[
  {
    "local": [
      {
        "user": {
          "name": "{0}",
          "email": "{1}",
          "domain": {
            "name": "federated"
          }
        }
      },
      {
        "group": {
          "id": "c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6"
        }
      }
    ],
    "remote": [
      {
        "type": "OIDC_CLAIM_preferred_username"
      },
      {
        "type": "OIDC_CLAIM_email"
      },
      {
        "type": "OIDC_CLAIM_groups",
        "any_one_of": ["openstack-users"]
      }
    ]
  },
  {
    "local": [
      {
        "user": {
          "name": "{0}"
        }
      },
      {
        "group": {
          "id": "d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1"
        }
      }
    ],
    "remote": [
      {
        "type": "OIDC_CLAIM_preferred_username"
      },
      {
        "type": "OIDC_CLAIM_groups",
        "any_one_of": ["openstack-admins"]
      }
    ]
  }
]
EOF

openstack mapping create keycloak-oidc-mapping \
  --rules /tmp/keycloak-oidc-mapping.json
```

### Create the OIDC Protocol

```bash
openstack federation protocol create oidc \
  --identity-provider keycloak-idp \
  --mapping keycloak-oidc-mapping
```

### Test the OIDC Flow

```bash
# Get an unscoped federated token
openstack token issue \
  --os-auth-type v3oidcpassword \
  --os-auth-url http://10.0.0.10:5000/v3 \
  --os-identity-provider keycloak-idp \
  --os-protocol oidc \
  --os-discovery-endpoint https://keycloak.example.com/realms/openstack/.well-known/openid-configuration \
  --os-client-id keystone-sp \
  --os-client-secret oidc_client_secret_here \
  --os-username alice \
  --os-password s3cur3P@ss

# Exchange unscoped token for a scoped token
openstack token issue \
  --os-auth-type v3token \
  --os-token <unscoped-token-from-above> \
  --os-project-name engineering \
  --os-project-domain-name Default
```

## Keystone-to-Keystone (K2K) Federation

K2K allows one OpenStack Keystone (the IdP Keystone, `cloud-a`) to issue tokens that a second Keystone (the SP Keystone, `cloud-b`) accepts. Users authenticate to cloud-a and receive tokens that grant access to resources in cloud-b.

### Architecture

```
  Cloud A (IdP Keystone)                  Cloud B (SP Keystone)
  ─────────────────────                   ──────────────────────
  User authenticates                      Receives SAML2 assertion
  Keystone issues token          ────→    Validates assertion
  Keystone generates SAML2               Issues scoped token
  assertion for cloud-b                   User accesses resources
```

### Configure Cloud A (IdP Keystone)

```ini
# /etc/keystone/keystone.conf on cloud-a
[saml]
# SP entity ID (our Keystone's identity)
idp_entity_id = https://keystone-a.example.com/v3/OS-FEDERATION/saml2/idp
idp_sso_endpoint = https://keystone-a.example.com/v3/OS-FEDERATION/saml2/auth
# Signing certificate for SAML2 assertions
certfile = /etc/keystone/ssl/certs/signing_cert.pem
keyfile = /etc/keystone/ssl/private/signing_key.pem
# Contact info
idp_contact_company = Example Corp
idp_contact_name = OpenStack Admin
idp_contact_email = admin@example.com
idp_metadata_path = /etc/keystone/saml2_idp_metadata.xml
```

Generate IdP metadata:

```bash
keystone-manage saml_idp_metadata > /etc/keystone/saml2_idp_metadata.xml
```

### Register Cloud B as a Service Provider in Cloud A

```bash
# On cloud-a
openstack service provider create cloud-b \
  --auth-url https://keystone-b.example.com/v3/OS-FEDERATION/identity_providers/cloud-a/protocols/saml2/auth \
  --service-provider-url https://keystone-b.example.com/Shibboleth.sso/SAML2/ECP \
  --description "Cloud B Service Provider"
```

### Configure Cloud B (SP Keystone)

Cloud B uses the same Shibboleth/mod_shib setup as a regular SAML2 SP, but the IdP is cloud-a's Keystone:

```bash
# On cloud-b
openstack identity provider create cloud-a-idp \
  --remote-id https://keystone-a.example.com/v3/OS-FEDERATION/saml2/idp \
  --description "Cloud A Keystone IdP"
```

Create mapping for cloud-a users on cloud-b:

```bash
cat > /tmp/k2k-mapping.json << 'EOF'
[
  {
    "local": [
      {
        "user": {
          "name": "{0}",
          "domain": {
            "name": "cloud-a-federated"
          }
        }
      },
      {
        "group": {
          "id": "e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2"
        }
      }
    ],
    "remote": [
      {
        "type": "openstack_user"
      },
      {
        "type": "openstack_roles",
        "any_one_of": ["member", "admin"]
      }
    ]
  }
]
EOF

openstack mapping create k2k-mapping --rules /tmp/k2k-mapping.json

openstack federation protocol create saml2 \
  --identity-provider cloud-a-idp \
  --mapping k2k-mapping
```

### Authenticate K2K (Python keystoneauth1 example)

```python
from keystoneauth1 import session
from keystoneauth1.identity.v3 import Password
from keystoneauth1.identity.v3 import Keystone2Keystone

# Step 1: Authenticate to cloud-a
auth_a = Password(
    auth_url="https://keystone-a.example.com/v3",
    username="alice",
    password="s3cur3P@ss",
    user_domain_name="Default",
    project_name="engineering",
    project_domain_name="Default",
)
session_a = session.Session(auth=auth_a)

# Step 2: Use cloud-a token to get cloud-b token
auth_b = Keystone2Keystone(
    base_plugin=auth_a,
    service_provider="cloud-b",
    project_name="cross-cloud-project",
    project_domain_name="Default",
)
session_b = session.Session(auth=auth_b)
token_b = session_b.get_token()
```

## Mapping Rules

Mapping rules are JSON arrays. Each rule has a `local` and `remote` section.

### Rule Structure

```json
[
  {
    "local": [
      { "user": { "name": "{0}" } },
      { "group": { "id": "..." } },
      { "projects": [ { "name": "{1}", "roles": [ { "name": "member" } ] } ] }
    ],
    "remote": [
      { "type": "ATTRIBUTE_NAME" },
      { "type": "ATTRIBUTE_NAME", "any_one_of": ["value1", "value2"] },
      { "type": "ATTRIBUTE_NAME", "not_any_of": ["denied_value"] },
      { "type": "ATTRIBUTE_NAME", "regex": true, "any_one_of": ["pattern.*"] }
    ]
  }
]
```

### `remote` Conditions

| Condition | Meaning |
|---|---|
| `any_one_of` | Attribute must contain at least one listed value |
| `not_any_of` | Attribute must NOT contain any listed value |
| `blacklist` | Exclude these values (deprecated — use `not_any_of`) |
| `whitelist` | Include only these values (deprecated — use `any_one_of`) |
| `regex: true` | Treat `any_one_of` values as Python regex patterns |

### `local` Directives

| Directive | Effect |
|---|---|
| `user.name` | Set the federated user's local name |
| `user.email` | Set the user's email |
| `user.domain.name` | Assign the user to a specific domain |
| `user.type` | `local` (persistent shadow user) or `ephemeral` (no DB record) |
| `group.id` | Add user to an existing Keystone group (by UUID) |
| `group.name` + `group.domain` | Add user to a group by name+domain |
| `projects` | Directly assign the user to projects with specific roles |

### Auto-Provision Projects (Direct Project Mapping)

```json
{
  "local": [
    {
      "user": { "name": "{0}" },
      "projects": [
        {
          "name": "federated-sandbox",
          "roles": [ { "name": "member" } ]
        }
      ]
    }
  ],
  "remote": [
    { "type": "OIDC_CLAIM_preferred_username" }
  ]
}
```

## Manage Identity Providers

### Create an Identity Provider

```bash
openstack identity provider create acme-idp \
  --remote-id https://idp.acme-corp.com/idp/shibboleth \
  --description "ACME Corp corporate IdP"
```

Multiple `--remote-id` values can be specified (useful for IdPs with multiple entity IDs).

### List Identity Providers

```bash
openstack identity provider list
```

### Show an Identity Provider

```bash
openstack identity provider show acme-idp
```

### Update an Identity Provider

```bash
openstack identity provider set acme-idp \
  --remote-id https://idp.acme-corp.com/idp/shibboleth \
  --remote-id https://idp2.acme-corp.com/idp/shibboleth
```

### Delete an Identity Provider

```bash
openstack identity provider delete acme-idp
```

## Manage Federation Protocols

### Create a Protocol

```bash
openstack federation protocol create saml2 \
  --identity-provider acme-idp \
  --mapping acme-saml-mapping
```

### List Protocols for an IdP

```bash
openstack federation protocol list --identity-provider acme-idp
```

### Show a Protocol

```bash
openstack federation protocol show saml2 --identity-provider acme-idp
```

### Update a Protocol's Mapping

```bash
openstack federation protocol set saml2 \
  --identity-provider acme-idp \
  --mapping acme-saml-mapping-v2
```

### Delete a Protocol

```bash
openstack federation protocol delete saml2 --identity-provider acme-idp
```

## Manage Mappings

### Create a Mapping

```bash
openstack mapping create acme-saml-mapping \
  --rules /tmp/acme-saml-mapping.json
```

### List Mappings

```bash
openstack mapping list
```

### Show a Mapping

```bash
openstack mapping show acme-saml-mapping
```

### Update a Mapping

```bash
openstack mapping set acme-saml-mapping \
  --rules /tmp/acme-saml-mapping-v2.json
```

### Delete a Mapping

```bash
openstack mapping delete acme-saml-mapping
```

## WebSSO (Horizon Integration)

To allow Horizon users to log in via federated IdPs, configure WebSSO.

### Keystone Config

```ini
[federation]
trusted_dashboard = http://203.0.113.10/dashboard/auth/websso/
sso_callback_template = /etc/keystone/sso_callback_template.html
```

### Horizon Config

```python
# /etc/openstack-dashboard/local_settings.py
WEBSSO_ENABLED = True
WEBSSO_INITIAL_CHOICE = "credentials"
WEBSSO_CHOICES = (
    ("credentials", _("Keystone Credentials")),
    ("acme-idp_saml2", _("ACME Corp SSO (SAML2)")),
    ("keycloak-idp_oidc", _("Keycloak OIDC")),
)
# Format: "<identity_provider_name>_<protocol>"
```

The Horizon login page will show a dropdown with the configured IdP choices.

## Troubleshoot Federation

### SAML2 Attribute Mapping Issues

```bash
# Check what attributes Shibboleth is passing
# Add to Apache config temporarily:
# ShibUseHeaders On  (exposes SHIB_ headers for debugging)

# View active Shibboleth session (user must be logged in)
curl http://keystone.example.com/Shibboleth.sso/Session

# Check Shibboleth log
tail -f /var/log/shibboleth/shibd.log
```

### OIDC Claims Debug

```bash
# Add to Apache (development only):
# OIDCHTMLErrorTemplate /tmp/oidc_error.html
# LogLevel debug

# Verify OIDC_CLAIM_ environment variables are passed:
# Add a debug handler that prints env vars to a test endpoint
```

### Keystone Mapping Evaluation

```bash
# Test mapping evaluation against a set of remote attributes
openstack mapping purge --all
# Check keystone.log for mapping evaluation details
# Enable debug logging temporarily:
tail -f /var/log/keystone/keystone.log | grep -i mapping
```

### Shadow User Issues

```bash
# List shadow users for a specific IdP
openstack user list --domain federated

# Show a federated user (shadow user)
openstack user show alice --domain federated

# Force re-evaluation: delete the shadow user (they will be re-created on next login)
openstack user delete alice --domain federated
```
