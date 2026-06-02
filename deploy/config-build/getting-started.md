# Getting Started — First Steps

Before building any configuration, do these steps first. Always.

## 1. Read the Release Notes

Before anything else, read the release notes for the target release:
https://docs.openstack.org/releasenotes/kolla-ansible/

This catches breaking changes, deprecated options, and known issues before you hit them.

## 2. Get the STOCK globals.yml and Inventory

**Never write globals.yml from scratch.** Copy the full template from kolla-ansible. It has every option with documentation comments, and hidden defaults you need to know about (like `kolla_base_distro_version`).

```bash
source /opt/kolla-venv/bin/activate

# Stock files are under share/, NOT the Python package path
KOLLA_SHARE="/opt/kolla-venv/share/kolla-ansible"

# Copy the full globals.yml template
cp "$KOLLA_SHARE/etc_examples/kolla/globals.yml" globals.yml

# Copy the stock inventory (all-in-one or multinode)
mkdir -p inventory
cp "$KOLLA_SHARE/ansible/inventory/all-in-one" inventory/hosts
# or: cp "$KOLLA_SHARE/ansible/inventory/multinode" inventory/hosts
```

**Important:** The stock files live at `/opt/kolla-venv/share/kolla-ansible/`, NOT at the path returned by `kolla_ansible.__file__`. The Python path points to the package code, not the etc_examples.

**Why use the stock files?** `git diff` will show exactly what was customized. A from-scratch file hides which defaults are relied on and makes it easy to miss important settings. The stock file also has inline documentation for every option.

**Commit the originals first:**
```bash
git add globals.yml inventory/
git commit -m "stock kolla-ansible globals.yml and inventory"
```

Then edit and commit your changes separately — the diff shows exactly what you customized.

If kolla-ansible isn't installed yet, download directly:

```bash
curl -sL https://opendev.org/openstack/kolla-ansible/raw/branch/stable/2025.1/etc/kolla/globals.yml > globals.yml
curl -sL https://opendev.org/openstack/kolla-ansible/raw/branch/stable/2025.1/ansible/inventory/all-in-one > inventory/hosts
```

## 3. Verify Host Interfaces

**Always verify actual interface names** on the target hosts before editing globals.yml. Operators often get interface names wrong.

```bash
ssh root@<host> ip -br addr
```

Look for:
- Management interface: the one with an IP on the management network
- External interface: the one for provider/external networks (should be UP but no IP)

## 4. Generate passwords.yml

`kolla-genpwd` requires the passwords template file to exist first:

```bash
source /opt/kolla-venv/bin/activate
KOLLA_SHARE="/opt/kolla-venv/share/kolla-ansible"

# kolla-genpwd REQUIRES the template file to exist first — it will fail without it
cp "$KOLLA_SHARE/etc_examples/kolla/passwords.yml" passwords.yml

# Generate random passwords (fills in the template)
kolla-genpwd -p passwords.yml
```

**`kolla-genpwd` will fail if the file doesn't exist.** You must seed it from the template first.

**Copy back to local repo immediately.** If passwords are generated on the host, `scp` them back:
```bash
scp root@<host>:/etc/kolla/passwords.yml ./passwords.yml
```

Without this, losing the host means losing all credentials.

**Encrypt at rest** — passwords.yml contains plaintext credentials for every service:
```bash
# Option 1: SOPS
sops --encrypt --in-place passwords.yml

# Option 2: Ansible Vault
ansible-vault encrypt passwords.yml

# Option 3: .gitignore (minimum)
echo "passwords.yml" >> .gitignore
```

## 5. Apply Known Issue Workarounds

Before editing globals.yml, check `deploy/deploy/known-issues-2025.1.md` for required workarounds. For 2025.1, you MUST set:

```yaml
enable_proxysql: "no"
keystone_wsgi_provider: "apache"
kolla_install_type: "binary"
```

## 6. AIO Self-SSH Setup

For all-in-one deployments, the host must be able to SSH to itself (kolla-ansible uses SSH even for AIO):

```bash
ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
ssh -o StrictHostKeyChecking=no root@<host-ip> 'echo ok'
```

## 7. Copy Config to /etc/kolla

Before any kolla-ansible command, config must be in `/etc/kolla/`:

```bash
sudo mkdir -p /etc/kolla
sudo cp globals.yml /etc/kolla/globals.yml
sudo cp passwords.yml /etc/kolla/passwords.yml
# If you have service overrides:
sudo cp -r config/ /etc/kolla/config/
```

## Order of Operations

1. Read release notes
2. Install kolla-ansible in a venv
3. Copy stock `globals.yml` and `inventory/hosts` from kolla-ansible
4. Commit stock files
5. Verify host interface names
6. Edit `globals.yml` — uncomment and modify only what you need
7. Edit `inventory/hosts` — set host assignments at the top
8. Apply known issue workarounds to `globals.yml`
9. Generate `passwords.yml`
10. Copy everything to `/etc/kolla/`
11. Run `kolla-ansible bootstrap-servers`
12. Run `kolla-ansible prechecks`
13. Run `kolla-ansible pull`
14. Run `kolla-ansible deploy`
15. Run `kolla-ansible post-deploy`
16. Copy `passwords.yml` back to local repo + encrypt
