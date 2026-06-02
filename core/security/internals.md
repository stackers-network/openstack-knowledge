# Barbican Internals

## Plugin Architecture

Barbican uses a two-layer plugin model:

```
barbican-api / barbican-worker
         │
         ▼
  ┌──────────────────┐
  │  Secret Store    │  High-level: where secrets are stored
  │  Plugin          │  (Castellan interface)
  └────────┬─────────┘
           │
  ┌────────▼─────────┐
  │  Crypto Plugin   │  Low-level: how encryption is performed
  │                  │  (used only by store_crypto plugin)
  └──────────────────┘
```

The **secret store plugin** controls where secrets live (database, HSM, Vault, KMIP appliance). The **crypto plugin** controls how encryption is performed when using the `store_crypto` secret store plugin (which stores encrypted blobs in the Barbican database).

## Secret Store Plugin Interface

All secret store plugins implement the `barbican.plugin.interface.secret_store.SecretStoreBase` abstract class. The key methods are:

### `store_secret(secret_dto)`

Called when a client posts a secret with a payload (e.g., `POST /v1/secrets` with payload data). Receives a `SecretDTO` containing:
- `secret_type`: enum value (`SYMMETRIC`, `PUBLIC`, `PRIVATE`, `CERTIFICATE`, `PASSPHRASE`, `OPAQUE`)
- `secret`: the raw secret bytes (already decrypted from transport key, if any)
- `content_type`: MIME type
- `content_encoding`: `base64` or `None`
- `key_spec`: `KeySpec(alg, bit_length, mode)`

Returns a `dict` of plugin-specific metadata to be persisted in `secret_store_metadata`. For `store_crypto`, this is the ciphertext and IV.

### `get_secret(secret_type, secret_metadata)`

Called when a client requests the payload (`GET /v1/secrets/<uuid>/payload`). Receives the metadata dict returned by `store_secret`. Returns the plaintext bytes.

### `delete_secret(secret_metadata)`

Called when a client deletes a secret. The plugin frees any external resources (e.g., deletes from HSM or KMIP store).

### `generate_symmetric_key(key_spec)`

Called when an order for a symmetric key is processed by the worker. Returns a `GeneratedSecret` containing the generated key bytes. For the `store_crypto` plugin, this delegates to the crypto plugin.

### `generate_asymmetric_key(key_spec)`

Similar to `generate_symmetric_key` but returns a `GeneratedAsymmetricSecret` with `public_key`, `private_key`, and optional `private_key_passphrase`.

### `supports(key_spec)`

Returns `True` if this plugin can handle the given `KeySpec`. Barbican iterates the list of enabled plugins and picks the first that returns `True`.

## Crypto Plugin Interface

The crypto plugin is a lower-level interface used by the `store_crypto` secret store plugin for the actual encryption/decryption work. All implementations inherit from `barbican.plugin.crypto.base.CryptoPluginBase`.

### `encrypt(encrypt_dto, kek_meta_dto, project_id)`

Encrypts a plaintext secret. Receives:
- `encrypt_dto.unencrypted`: plaintext bytes
- `kek_meta_dto`: Key Encryption Key metadata for this project

Returns an `EncryptDTO` with:
- `cypher_text`: the ciphertext bytes
- `kek_meta_extended`: any additional KÉK metadata to persist

### `decrypt(decrypt_dto, kek_meta_dto, project_id)`

Decrypts ciphertext. Returns the plaintext bytes.

### `bind_kek_metadata(kek_meta_dto)`

Called once per project to initialize or retrieve the Key Encryption Key for that project. For `simple_crypto`, this is a no-op (the master KEK wraps all project keys). For HSM plugins, this provisions a project-specific key on the HSM.

### `generate_symmetric(generate_dto, kek_meta_dto, project_id)`

Generates a new symmetric key inside the plugin. Returns `GeneratedSecret`.

### `generate_asymmetric(generate_dto, kek_meta_dto, project_id)`

Generates a new key pair inside the plugin. Returns `GeneratedAsymmetricSecret`.

### `supports(type_enum)`

Returns `True` if the plugin supports the requested operation type.

## Available Plugins

### simple_crypto (default)

The default plugin for new deployments. It stores encrypted secret blobs directly in the Barbican MariaDB database.

**Key hierarchy:**

```
Master KEK (barbican.conf)
    │  wraps (AES-GCM)
    ▼
Project KEK (one per project, stored in kek_data table)
    │  encrypts (AES-GCM)
    ▼
Secret ciphertext (stored in encrypted_data table)
```

- The master KEK is a 32-byte Fernet-compatible key stored in `/etc/barbican/barbican.conf` under `[simple_crypto_plugin] kek`
- Each project gets a unique Project KEK encrypted by the master KEK
- Secrets are encrypted with the Project KEK using AES-256-GCM
- The IV and ciphertext are stored in `encrypted_data`

Configuration:

```ini
[secretstore]
enabled_secretstore_plugins = store_crypto

[crypto]
enabled_crypto_plugins = simple_crypto

[simple_crypto_plugin]
kek = dGhpcyBpcyBhIHRlc3Qga2V5Zm9yYmFyYmljYW4hCg==
```

Weaknesses: the master KEK is a secret in a config file. In production, protect this file and consider rotating it periodically.

### PKCS#11 (HSM)

Routes crypto operations to a Hardware Security Module via the PKCS#11 standard interface. The HSM never exposes private key material; all cryptographic operations happen inside the HSM boundary.

Supported HSMs:
- Thales Luna Network HSM
- Entrust nShield
- SoftHSM (testing only — software implementation of PKCS#11)

Configuration:

```ini
[secretstore]
enabled_secretstore_plugins = store_crypto

[crypto]
enabled_crypto_plugins = p11_crypto

[p11_crypto_plugin]
# Path to the PKCS#11 shared library
library_path = /usr/lib/libCryptoki2_64.so
# HSM login password (partition PIN)
login = partition_pin_here
# Label of the key wrapping key stored in the HSM
mkek_label = barbican_mkek
# Length of the MKEK in bytes (32 = 256-bit AES)
mkek_length = 32
# Label for the HMAC key used for integrity checks
hmac_label = barbican_hmac
# PKCS#11 slot number (use pkcs11-tool --list-slots to find)
slot_id = 1
# Number of PKCS#11 sessions to pool
rw_session_pool_size = 10
```

Install the `barbican-plugin-crypto-p11` package which includes the `oslo_utils` C extensions for PKCS#11 session management.

### KMIP (Key Management Interoperability Protocol)

Stores secrets on an external KMIP-compliant key manager (e.g., Thales CipherTrust, IBM SKLM, or PyKMIP for testing). The secret material lives on the KMIP server; Barbican stores only the KMIP object ID.

Configuration:

```ini
[secretstore]
enabled_secretstore_plugins = kmip_plugin

[kmip_plugin]
# KMIP server address and port
host = kmip.example.com
port = 5696
# Mutual TLS for KMIP connection
keyfile = /etc/barbican/kmip-client.key
certfile = /etc/barbican/kmip-client.crt
ca_certs = /etc/barbican/kmip-ca.crt
# Username/password authentication (if not using cert auth)
username = barbican
password = kmip_pass
```

### HashiCorp Vault

Stores secrets in a HashiCorp Vault KV secrets engine. Barbican acts as a Vault client via `castellan-vault`.

Configuration:

```ini
[secretstore]
enabled_secretstore_plugins = vault_plugin

[vault_plugin]
vault_url = https://vault.example.com:8200
# Vault token with access to the secrets engine
root_token_id = s.VaultTokenHere
# Or AppRole authentication
use_ssl = true
ssl_ca_crt_file = /etc/barbican/vault-ca.crt
kv_mountpoint = secret
```

Vault plugin stores each Barbican secret as a Vault KV entry at `secret/barbican/<project_id>/<secret_id>`.

## Certificate Plugin Interface

Certificate plugins implement `barbican.plugin.interface.certificate_manager.CertificatePluginBase`. Key methods:

### `issue_certificate_request(order_id, order_meta, plugin_meta, user_meta)`

Receives the CSR (in `order_meta['request_data']`) and submits it to the configured CA. May return `WAITING` (async) or `ISSUED` (synchronous CA). On `ISSUED`, populates `plugin_meta` with the signed certificate.

### `check_certificate_status(order_id, order_meta, plugin_meta, user_meta)`

Called periodically by the worker when the CA is async (e.g., an enterprise CA with an approval workflow). Returns the current status: `WAITING`, `ISSUED`, `CLIENT_AUTH_ISSUE_ERROR`, or `CA_UNAVAILABLE_FOR_REQUEST`.

### `modify_certificate_request(order_id, order_meta, plugin_meta, user_meta)`

Called to update an existing certificate request (e.g., to cancel or revoke).

Built-in certificate plugins:
- **Symantec**: Posts to the Symantec managed PKI API
- **Dogtag** (Red Hat Certificate System): Full CA integration via the Dogtag REST API
- **CFSSL**: Integration with Cloudflare's CFSSL CA
- **Simple certificate**: Local CA for testing (wraps OpenSSL)

## Transport Key Wrapping

Transport keys allow clients to encrypt secret payloads before sending them to the Barbican API, so the payload is never in plaintext in the API process.

### Server Side

1. barbican-api generates an RSA key pair at startup (or on demand)
2. The public key is exposed via `GET /v1/transport_keys`
3. The private key is held in memory by the API worker processes

### Client Side

```python
from cryptography.hazmat.primitives.asymmetric import padding
from cryptography.hazmat.primitives import hashes, serialization
import base64, requests

# 1. Retrieve transport public key
r = requests.get("https://barbican.example.com:9311/v1/transport_keys",
                 headers={"X-Auth-Token": token})
transport_key_href = r.json()["transport_keys"][0]["transport_key_ref"]
tk_r = requests.get(transport_key_href, headers={"X-Auth-Token": token})
pem = tk_r.json()["transport_key"].encode()

# 2. Load the public key
pub_key = serialization.load_pem_public_key(pem)

# 3. Encrypt the secret payload
plaintext = b"sup3rs3cr3t"
ciphertext = pub_key.encrypt(
    plaintext,
    padding.OAEP(
        mgf=padding.MGF1(algorithm=hashes.SHA256()),
        algorithm=hashes.SHA256(),
        label=None
    )
)
wrapped = base64.b64encode(ciphertext).decode()

# 4. Submit the wrapped secret
requests.post("https://barbican.example.com:9311/v1/secrets",
    headers={"X-Auth-Token": token, "Content-Type": "application/json"},
    json={
        "name": "my-wrapped-secret",
        "trans_wrapped_key": wrapped,
        "transport_key_ref": transport_key_href,
        "payload_content_type": "text/plain",
        "secret_type": "passphrase"
    }
)
```

## Castellan: The Unified Key Manager Interface

Other OpenStack services (Nova, Cinder, Octavia, Swift) access Barbican through the **castellan** library, which provides a service-agnostic `KeyManager` interface. This means the same Nova code works whether secrets are in Barbican, Vault, or another backend.

Configuration in Nova (`/etc/nova/nova.conf`):

```ini
[key_manager]
backend = barbican
```

Configuration in Cinder (`/etc/cinder/cinder.conf`):

```ini
[key_manager]
backend = barbican
```

Castellan uses the service user's Keystone token to authenticate against Barbican. No additional credential configuration is needed if `[keystone_authtoken]` is already configured.

## Key Rotation

Barbican does not have a built-in key rotation command. To rotate the simple_crypto master KEK:

1. Generate a new master KEK value
2. Update `[simple_crypto_plugin] kek` in `barbican.conf`
3. Restart `barbican-api` and `barbican-worker`
4. All new secrets are encrypted under the new master KEK
5. Existing secrets still decrypt because their Project KEKs were encrypted under the old master — you must re-encrypt them manually or accept the mixed state until secrets are rotated naturally

For PKCS#11 or KMIP backends, key rotation is handled by the HSM or KMIP server's own policies.
