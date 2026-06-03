# Compatibility — Version Matrix

This skill covers OpenStack release compatibility: which Python versions, database versions, message broker versions, and host operating systems are supported for each release. Check this before deploying or upgrading.

**Always verify against the official release notes** for the exact versions you are deploying — minor updates within a release cycle sometimes adjust minimum versions. Official reference: https://releases.openstack.org

---

## Release Overview

OpenStack follows a 6-month cadence: April (`.1`) and October (`.2`) releases. SLURP releases (Skip-Level Upgrade Release Process) are designated approximately yearly and allow direct upgrade skipping one intermediate release. Non-SLURP releases require sequential upgrades.

| Release | Code Name | Release Date | SLURP | Support Status |
|---|---|---|---|---|
| 2024.1 | Caracal | Apr 2024 | Yes | Maintained |
| 2024.2 | Dalmatian | Oct 2024 | No | Maintained |
| 2025.1 | Epoxy | Apr 2025 | Yes | Maintained |
| 2025.2 | Flamingo | Oct 2025 | No | Maintained |
| 2026.1 | Gazpacho | Apr 2026 | Yes | Maintained (current stable) |
| 2026.2 | Hibiscus | Oct 2026 (est.) | No | Development |

SLURP releases receive extended support (approximately 18 months after release). Non-SLURP releases receive standard support (approximately 12 months).

---

## Python Compatibility

| Release | Minimum Python | Maximum Python | Notes |
|---|---|---|---|
| 2024.1 (Caracal) | 3.10 | 3.11 | Python 3.9 support removed |
| 2024.2 (Dalmatian) | 3.10 | 3.12 | Python 3.12 added |
| 2025.1 (Epoxy) | 3.10 | 3.12 | All three versions tested in CI |
| 2025.2 (Flamingo) | 3.10 | 3.12 | Consistent with 2025.1 |
| 2026.1 (Gazpacho) | 3.10 | 3.13 | Official tested runtimes: Python 3.10–3.13 (3.10 remains the minimum; 3.13 added) |

Check the Python version on your target nodes:

```bash
python3 --version
# Ubuntu 22.04: 3.10 (compatible with 2024.x and 2025.x)
# Ubuntu 24.04: 3.12 (compatible with all current releases)
# Rocky Linux 9: 3.9 default (use python3.11 from AppStream for 2025.x+)
# Rocky Linux 10: 3.12 (compatible with all current releases)
```

---

## Database Compatibility

OpenStack services use MySQL/MariaDB as the primary and best-tested database. PostgreSQL is still accepted by most services but receives significantly less CI coverage and community testing — MariaDB is strongly recommended for production. Galera Cluster (based on MariaDB) is the standard HA database configuration.

| Release | MariaDB Minimum | MariaDB Recommended | MySQL Minimum | Notes |
|---|---|---|---|---|
| 2024.1 (Caracal) | 10.6 | 10.11 | 8.0 | MariaDB 10.4 support removed |
| 2024.2 (Dalmatian) | 10.6 | 10.11 | 8.0 | No change |
| 2025.1 (Epoxy) | 10.6 | 10.11 | 8.0 | MariaDB 10.11 LTS recommended |
| 2025.2 (Flamingo) | 10.6 | 10.11 | 8.0 | Verify official release notes for exact minimums |
| 2026.1 | 10.11 | 11.x | 8.0+ | MariaDB 10.6 support expected to be removed; 10.11+ required — verify release notes |

Check your MariaDB version:

```bash
mysql --version
mysqladmin -u root -p version | grep "Server version"
```

### MariaDB Galera Notes

```bash
# Check Galera cluster size and sync status
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"
# wsrep_local_state_comment should be 'Synced' on all nodes

# Check Galera version
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_provider_version';"
```

Galera 4 (shipped with MariaDB 10.4+) is required for all current releases. Galera 3 reached end-of-life.

---

## RabbitMQ Compatibility

| Release | RabbitMQ Minimum | RabbitMQ Recommended | Erlang Minimum | Notes |
|---|---|---|---|---|
| 2024.1 (Caracal) | 3.10 | 3.12 | 25.0 | oslo.messaging supports AMQP 0-9-1 |
| 2024.2 (Dalmatian) | 3.11 | 3.12 | 25.0 | No change |
| 2025.1 (Epoxy) | 3.12 | 3.13 | 26.0 | RabbitMQ 3.12+ required |
| 2025.2 (Flamingo) | 3.12 | 3.13 | 26.0 | Verify release notes |
| 2026.1 | 3.13 | 3.13+ | 26.0+ | RabbitMQ 3.12 support expected to be removed — verify release notes |

Check RabbitMQ version:

```bash
rabbitmqctl status | grep "RabbitMQ\|Erlang"
rabbitmq-diagnostics server_version
```

---

## Host OS Compatibility

OpenStack is packaged and tested against specific host operating systems. Using an untested OS may work but is not supported by the OpenStack project or distributions.

| Release | Ubuntu 22.04 (Jammy) | Ubuntu 24.04 (Noble) | Rocky Linux 9 | Rocky Linux 10 | Debian 12 (Bookworm) | CentOS Stream 9 |
|---|---|---|---|---|---|---|
| 2024.1 (Caracal) | Yes | Partial (backports) | Yes | No | Yes | Yes |
| 2024.2 (Dalmatian) | Yes | Yes | Yes | No | Yes | Yes |
| 2025.1 (Epoxy) | Yes | Yes | Yes | Yes (early) | Yes | Limited |
| 2025.2 (Flamingo) | Yes | Yes | Yes | Yes | Yes | Verify |
| 2026.1 | Verify | Yes | Verify (RDO) | Verify (RDO) | — | — |

> **2026.1 (Gazpacho) official tested runtimes:** Ubuntu 24.04 and Debian 13 (Debian 12 also tested to validate SLURP upgrades from 2025.1). Distros outside that set — Rocky/RHEL via RDO, Ubuntu 22.04 — are community-packaged rather than upstream-tested; verify availability before relying on them. Source: https://governance.openstack.org/tc/reference/runtimes/2026.1.html
>
> *(This table predates the release and lists a Debian 12 column; Gazpacho's tested Debian is 13.)*

**Ubuntu** packages are maintained by the Ubuntu Cloud Archive (UCA) for LTS releases. UCA provides the current OpenStack release on Ubuntu LTS.

```bash
# Enable Ubuntu Cloud Archive for 2025.1 (Epoxy) on 22.04
add-apt-repository cloud-archive:epoxy
apt update

# Check which UCA pocket is active
apt-cache policy nova-api | grep -A3 "Candidate"
```

**Rocky Linux / RHEL** uses RDO or CentOS SIG packages:

```bash
# Enable OpenStack repo on Rocky Linux 9
dnf install centos-release-openstack-epoxy
dnf update
```

**Debian** uses packages from the Debian Cloud Team maintained in the Debian archive or backports.

---

## OpenStack Client Compatibility

The `python-openstackclient` (OSC) follows OpenStack releases but is generally backward-compatible across 2-3 releases. The micro-version negotiation in Nova, Cinder, and Neutron APIs handles server-side compatibility.

| Client Version | Compatible With | Notes |
|---|---|---|
| 6.x | 2024.x | Maintained |
| 7.x | 2025.1, 2025.2 | Current stable |
| 8.x (upcoming) | 2026.1 | Check release notes |

```bash
# Check client version
openstack --version

# Check negotiated API micro-version
openstack versions show

# Force a specific micro-version (Nova example)
openstack --os-compute-api-version 2.87 server list
```

---

## Service API Micro-versions

Major services support micro-versioned APIs. Clients and services negotiate the highest mutually supported version.

| Service | API Version | Min Micro-version | Max Micro-version (2025.1) |
|---|---|---|---|
| Nova | v2.1 | 2.1 | 2.96 |
| Cinder | v3 | 3.0 | 3.70 |
| Neutron | v2.0 | N/A (no micro-versions) | N/A |
| Glance | v2 | N/A | N/A |
| Keystone | v3 | N/A | N/A |
| Ironic | v1 | 1.1 | 1.86 |
| Octavia | v2 | N/A | N/A |

```bash
# Check Nova API max micro-version on the server
openstack versions show | grep compute

# Check Cinder max micro-version
openstack versions show | grep volume
```

---

## SLURP Upgrade Paths

SLURP allows skipping one intermediate non-SLURP release during upgrades.

Valid SLURP upgrade paths:

```
2024.1 (Caracal) → 2025.1 (Epoxy)     # Skip 2024.2 Dalmatian
2025.1 (Epoxy)   → 2026.1             # Skip 2025.2 Flamingo
```

Non-SLURP sequential paths (required for non-SLURP releases):

```
2024.1 (Caracal) → 2024.2 (Dalmatian) → 2025.1 (Epoxy)
2025.1 (Epoxy) → 2025.2 (Flamingo) → 2026.1
```

Before any upgrade:
1. Check `reference/known-issues/` for the target release.
2. Run `openstack-ansible prechecks` or `kolla-ansible prechecks` to validate the current state.
3. Backup all databases.
4. Review deprecated configuration options removed in the target release.

---

## Ceph Compatibility

Ceph integration (for Nova, Cinder, Glance, Manila) follows Ceph's own release cadence:

| OpenStack Release | Ceph Reef (18.x) | Ceph Squid (19.x) | Ceph Tentacle (20.x) |
|---|---|---|---|
| 2024.1 (Caracal) | Yes | No | No |
| 2024.2 (Dalmatian) | Yes | Yes | No |
| 2025.1 (Epoxy) | Yes | Yes | No |
| 2025.2 (Flamingo) | Yes | Yes | Verify |
| 2026.1 | Verify | Yes | Yes (expected) |

```bash
# Check Ceph version
ceph version
ceph --version

# Check cluster health
ceph health detail
ceph status
```

---

## Keystone Token Provider Compatibility

Fernet tokens are the only supported token provider since Queens. UUID tokens were removed in Stein. JWS (JSON Web Signature) token provider is available as an alternative since Stein.

| Release | Fernet | JWS | UUID |
|---|---|---|---|
| 2024.x+ | Yes (default) | Yes | Removed |

```bash
# Check current token provider
grep "provider" /etc/keystone/keystone.conf | grep -v "^#"

# Rotate Fernet keys (run periodically)
keystone-manage fernet_rotate --keystone-user keystone --keystone-group keystone
```
