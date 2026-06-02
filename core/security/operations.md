# Barbican Operations

## Install and Bootstrap Barbican

```bash
# Install packages (Ubuntu/Debian)
apt install barbican-api barbican-worker python3-barbicanclient

# Sync the database
barbican-db-manage upgrade

# Register service and endpoints
openstack service create --name barbican --description "Key Manager Service" key-manager

openstack endpoint create --region RegionOne \
  barbican public https://barbican.example.com:9311
openstack endpoint create --region RegionOne \
  barbican internal https://10.0.0.15:9311
openstack endpoint create --region RegionOne \
  barbican admin https://10.0.0.15:9311

# Create service user
openstack user create barbican \
  --domain Default \
  --password barbican_service_pass
openstack role add --project service --user barbican admin
```

## Manage Secrets

### Store a Secret

Store a plaintext password:

```bash
openstack secret store \
  --name my-db-password \
  --payload 'sup3rs3cr3t' \
  --payload-content-type 'text/plain'
```

Store a binary secret (base64-encoded):

```bash
openstack secret store \
  --name my-aes-key \
  --payload "$(openssl rand -base64 32)" \
  --payload-content-type 'application/octet-stream' \
  --payload-content-encoding base64 \
  --secret-type symmetric \
  --algorithm aes \
  --bit-length 256
```

Store a PEM private key:

```bash
openstack secret store \
  --name my-tls-private-key \
  --payload "$(cat /etc/ssl/private/server.key)" \
  --payload-content-type 'application/pkcs8' \
  --secret-type private
```

Store a PEM certificate:

```bash
openstack secret store \
  --name my-tls-cert \
  --payload "$(cat /etc/ssl/certs/server.crt)" \
  --payload-content-type 'application/pkix-cert' \
  --secret-type certificate
```

Set an expiration date on a secret:

```bash
openstack secret store \
  --name temp-api-key \
  --payload 'xK9mV2pQ7rL3nZ8wA1sD4tF6gH0jU5eI' \
  --payload-content-type 'text/plain' \
  --expiration '2026-01-01T00:00:00'
```

### List Secrets

```bash
openstack secret list
```

Filter by name:

```bash
openstack secret list --name my-db-password
```

Filter by algorithm and secret type:

```bash
openstack secret list --algorithm aes --secret-type symmetric
```

Show only secrets expiring before a date:

```bash
openstack secret list --expiring-in-days 30
```

### Get a Secret

Show metadata (does not reveal the payload):

```bash
openstack secret get https://barbican.example.com:9311/v1/secrets/abc12345-1234-1234-1234-abcdef012345
```

Retrieve the payload:

```bash
openstack secret get \
  https://barbican.example.com:9311/v1/secrets/abc12345-1234-1234-1234-abcdef012345 \
  --payload
```

Retrieve a binary payload (base64-encoded output):

```bash
openstack secret get \
  https://barbican.example.com:9311/v1/secrets/abc12345-1234-1234-1234-abcdef012345 \
  --payload \
  --payload-content-type 'application/octet-stream'
```

### Delete a Secret

```bash
openstack secret delete \
  https://barbican.example.com:9311/v1/secrets/abc12345-1234-1234-1234-abcdef012345
```

Deleting a secret that is referenced by a container will fail with a 409 Conflict. Delete the container first.

## Manage Orders

Orders tell Barbican to generate key material on your behalf. Use orders instead of `secret store` when you want Barbican (and its configured HSM or crypto plugin) to generate the secret — not the client.

### Create a Symmetric Key Order

```bash
openstack secret order create \
  --name my-volume-key \
  --algorithm aes \
  --bit-length 256 \
  --mode cbc \
  key
```

Parameters:
- `--algorithm`: `aes`, `hmacsha256`, `hmacsha384`, `hmacsha512`
- `--bit-length`: 128, 192, 256 (for AES)
- `--mode`: `cbc`, `ctr`, `gcm` (AES block cipher mode)

### Create an Asymmetric Key Pair Order

```bash
openstack secret order create \
  --name my-keypair \
  --algorithm rsa \
  --bit-length 4096 \
  asymmetric
```

Parameters:
- `--algorithm`: `rsa`, `ec`, `dsa`
- `--bit-length`: 2048, 4096 (RSA); 256, 384, 521 (EC)
- `--curve` (EC only): `prime256v1`, `secp384r1`, `secp521r1`

### Create a Certificate Order

```bash
# Full certificate (Barbican generates key + CSR, submits to CA)
openstack secret order create \
  --name my-server-cert \
  --algorithm rsa \
  --bit-length 2048 \
  --subject-dn "CN=api.example.com,O=Example Corp,C=US" \
  certificate
```

### List Orders

```bash
openstack secret order list
```

### Get an Order

```bash
openstack secret order get \
  https://barbican.example.com:9311/v1/orders/order-uuid-here
```

The `status` field shows the current state:

| Status | Meaning |
|---|---|
| `PENDING` | Queued, not yet picked up by worker |
| `PROCESSING` | Worker is actively processing |
| `ACTIVE` | Completed successfully; `secret_href` contains the result |
| `ERROR` | Failed; check `error_status_code` and `error_reason` |

Poll until ACTIVE:

```bash
# Loop until status is ACTIVE
ORDER_HREF="https://barbican.example.com:9311/v1/orders/order-uuid-here"
while true; do
  STATUS=$(openstack secret order get "$ORDER_HREF" -f value -c Status)
  echo "Status: $STATUS"
  [ "$STATUS" = "ACTIVE" ] && break
  [ "$STATUS" = "ERROR" ] && echo "Order failed!" && break
  sleep 2
done
```

## Manage Containers

Containers group related secrets (e.g., a TLS certificate + private key).

### Create a Generic Container

```bash
SECRET_A=$(openstack secret store --name key-a --payload 'value-a' \
  --payload-content-type 'text/plain' -f value -c "Secret href")
SECRET_B=$(openstack secret store --name key-b --payload 'value-b' \
  --payload-content-type 'text/plain' -f value -c "Secret href")

openstack secret container create \
  --name my-config-bundle \
  --type generic \
  --secret "api_key=$SECRET_A" \
  --secret "db_password=$SECRET_B"
```

### Create a Certificate Container

Requires secrets named exactly `certificate`, `private_key`, and optionally `private_key_passphrase` and `intermediates`:

```bash
CERT_REF="https://barbican.example.com:9311/v1/secrets/cert-uuid"
KEY_REF="https://barbican.example.com:9311/v1/secrets/key-uuid"
INT_REF="https://barbican.example.com:9311/v1/secrets/intermediates-uuid"

openstack secret container create \
  --name my-tls \
  --type certificate \
  --secret "certificate=$CERT_REF" \
  --secret "private_key=$KEY_REF" \
  --secret "intermediates=$INT_REF"
```

This is the container type consumed by Octavia when you configure a TLS-terminated load balancer listener.

### Create an RSA Key Pair Container

Requires `public_key` and `private_key`; optionally `private_key_passphrase`:

```bash
openstack secret container create \
  --name my-rsa-pair \
  --type rsa \
  --secret "public_key=$PUB_REF" \
  --secret "private_key=$PRIV_REF"
```

### List Containers

```bash
openstack secret container list
```

### Get a Container

```bash
openstack secret container get \
  https://barbican.example.com:9311/v1/containers/container-uuid
```

### Delete a Container

```bash
openstack secret container delete \
  https://barbican.example.com:9311/v1/containers/container-uuid
```

Deleting a container does **not** delete the underlying secrets. Delete them separately if they are no longer needed.

## Manage Certificate Authorities

### List Available CAs

```bash
openstack ca list
```

### Show CA Details

```bash
openstack ca show \
  https://barbican.example.com:9311/v1/cas/ca-uuid
```

Show the CA's certificate chain:

```bash
openstack ca get \
  https://barbican.example.com:9311/v1/cas/ca-uuid \
  --cacert
```

### Set a Preferred CA (per project)

```bash
openstack ca set \
  https://barbican.example.com:9311/v1/cas/ca-uuid \
  --preferred
```

### Add a Sub-CA

If the configured plugin supports it (e.g., Dogtag), create a subordinate CA:

```bash
openstack ca create \
  --name "Project Dev CA" \
  --subject-dn "CN=Dev CA,O=Example Corp,C=US" \
  --parent-ca-ref https://barbican.example.com:9311/v1/cas/root-ca-uuid
```

## Manage Per-Project Quotas

Quotas limit the number of secrets, containers, and orders per project. The default is unlimited (-1).

### Show Current Quotas

```bash
# Show effective quotas for the current project (includes defaults and overrides)
openstack secret quota show

# Show quota overrides for a specific project (admin only)
openstack secret quota show --project <PROJECT_ID>
```

### Set Project Quota Overrides (admin)

```bash
openstack secret quota set \
  --project <PROJECT_ID> \
  --secrets 500 \
  --containers 200 \
  --orders 100 \
  --consumers 1000 \
  --cas 10
```

### Delete Quota Overrides (revert to defaults)

```bash
openstack secret quota delete --project <PROJECT_ID>
```

### Configure System-Wide Default Quotas

In `/etc/barbican/barbican.conf`:

```ini
[quota]
quota_secrets = 1000
quota_orders = 100
quota_containers = 500
quota_consumers = 1000
quota_cas = 10
```

## Secret ACLs

By default, secrets are accessible only to the project that created them. ACLs extend access to specific users.

### Add Read Access for a User

```bash
openstack acl set \
  --user <USER_ID1> \
  --user <USER_ID2> \
  --operation-type read \
  https://barbican.example.com:9311/v1/secrets/secret-uuid
```

### View ACL

```bash
openstack acl get \
  https://barbican.example.com:9311/v1/secrets/secret-uuid
```

### Remove ACL

```bash
openstack acl delete \
  https://barbican.example.com:9311/v1/secrets/secret-uuid
```

## Direct REST API Usage

For automation or when `python-barbicanclient` is not available:

```bash
TOKEN=$(openstack token issue -f value -c id)
BARBICAN_URL="https://barbican.example.com:9311"

# Store a secret via REST
curl -s -X POST "$BARBICAN_URL/v1/secrets" \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-db-password",
    "payload": "sup3rs3cr3t",
    "payload_content_type": "text/plain",
    "secret_type": "passphrase"
  }' | python3 -m json.tool

# List secrets
curl -s "$BARBICAN_URL/v1/secrets" \
  -H "X-Auth-Token: $TOKEN" | python3 -m json.tool

# Get payload
curl -s "$BARBICAN_URL/v1/secrets/abc12345/payload" \
  -H "X-Auth-Token: $TOKEN" \
  -H "Accept: text/plain"
```

## Verify Barbican Health

```bash
# Check API is reachable
curl -s https://barbican.example.com:9311/ | python3 -m json.tool

# Check worker is running
systemctl status barbican-worker

# Check keystone-listener is running
systemctl status barbican-keystone-listener

# Check the barbican-api logs
journalctl -u apache2 -f   # or httpd

# Run a round-trip test
SECRET=$(openstack secret store \
  --name health-check \
  --payload 'test-payload' \
  --payload-content-type 'text/plain' \
  -f value -c "Secret href")
openstack secret get "$SECRET" --payload
openstack secret delete "$SECRET"
echo "Barbican round-trip OK"
```
