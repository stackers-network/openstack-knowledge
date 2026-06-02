# Compatibility Matrix

Version compatibility for supported OpenStack releases deployed with kolla-ansible.

## OpenStack Release × Host OS

| OpenStack Release | Rocky Linux 10 | Ubuntu 24.04 (Noble) |
|---|---|---|
| 2025.1 (Epoxy) | Supported | Supported |
| 2025.2 | Supported | Supported |

Rocky Linux 9 and Ubuntu 22.04 are not supported by this stack. Upstream kolla-ansible may still support them — check the upstream release notes if you need older OS support.

## OpenStack Release × kolla-ansible Branch

| OpenStack Release | kolla-ansible Branch | PyPI Package |
|---|---|---|
| 2025.1 (Epoxy) | `stable/2025.1` | `kolla-ansible==2025.1.x` |
| 2025.2 | `stable/2025.2` | `kolla-ansible==2025.2.x` |

Always use the kolla-ansible branch that matches your target OpenStack release. Mixing versions is not supported.

## OpenStack Release × Networking Plugin

| OpenStack Release | OVN | Open vSwitch | LinuxBridge |
|---|---|---|---|
| 2025.1 (Epoxy) | Supported (recommended) | Supported | Supported |
| 2025.2 | Supported (recommended) | Supported | Supported |

OVN is the recommended default for new deployments. LinuxBridge and OVS remain supported but receive less upstream development focus.

## OpenStack Release × Storage Backend

| OpenStack Release | Ceph (external) | LVM | NFS |
|---|---|---|---|
| 2025.1 (Epoxy) | Supported (recommended for prod) | Supported | Supported |
| 2025.2 | Supported (recommended for prod) | Supported | Supported |

### Ceph Version Compatibility

| OpenStack Release | Ceph Reef (18.x) | Ceph Squid (19.x) |
|---|---|---|
| 2025.1 (Epoxy) | Supported | Supported |
| 2025.2 | Supported | Supported |

## OpenStack Release × Python Version

| OpenStack Release | Python 3.11 | Python 3.12 |
|---|---|---|
| 2025.1 (Epoxy) | Supported | Supported |
| 2025.2 | Supported | Supported |

Rocky Linux 10 ships Python 3.12. Ubuntu 24.04 ships Python 3.12. Both are supported.

## Caveats

- **kolla-ansible must be installed in a virtualenv** — do not install it system-wide. Use `python3 -m venv` and install with pip.
- **Ansible version** — kolla-ansible pins its Ansible dependency. Do not override the Ansible version manually.
- **Docker vs Podman** — kolla-ansible 2025.1+ supports both Docker and Podman as container runtimes. Docker remains the default. Podman support should be considered experimental for production.
