# Decision Guides

Use these guides when choosing between equivalent OpenStack components. Each guide presents a feature comparison table, recommendation by use case, and migration notes where applicable.

## Available Guides

| Guide | Decision |
|---|---|
| [networking.md](networking.md) | ML2/OVS vs ML2/OVN vs ML2/Linux Bridge — choose a Neutron ML2 mechanism driver |
| [storage-block.md](storage-block.md) | Cinder backend selection — LVM/iSCSI vs Ceph RBD vs NFS vs vendor drivers |
| [storage-object.md](storage-object.md) | Swift vs Ceph RadosGW — choose an object storage backend |
| [dashboard.md](dashboard.md) | Horizon vs Skyline — choose a web dashboard |
| [deployment-tools.md](deployment-tools.md) | Kolla-Ansible vs OpenStack-Ansible vs Charmed vs Kayobe vs manual |

## How to Use These Guides

1. Read the feature comparison table to identify hard requirements (HA, DPDK, existing infrastructure).
2. Review the "Recommendation by Use Case" section for your scenario.
3. Check the compatibility matrix at `reference/compatibility/` to confirm the chosen option is supported in your target release.
4. Check known issues at `reference/known-issues/` for the target release before committing to a choice.
