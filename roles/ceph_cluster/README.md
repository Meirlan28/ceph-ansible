# Role: ceph_cluster

Day-1 of a cephadm Ceph cluster on top of **cephadm-ansible** (`ceph.cephadm` modules): time sync, bootstrap,
orchestrator SSH key, hosts with labels, `ceph config`, service specs, pools, verification.

Package installation is not part of the role: run the upstream `cephadm-preflight.yml` first
(`playbooks/10-preflight.yml`). Purge stays upstream too (`playbooks/99-purge.yml`).

## Usage

```yaml
- hosts: ceph            # every storage node
  become: true
  roles:
    - ceph_cluster
```

Required inventory: group `admin` (first host bootstraps; or set `admin_node`), per host `ceph_ip` and `ceph_labels`,
`ceph_dashboard_password` (vault). Requires collections `ceph.cephadm` (cephadm-ansible) and `ansible.posix`.

## Phases (tags)

| Tag | Runs on | What |
|---|---|---|
| `ceph_time` | all | chrony: `ntp_server` follows `ntp_upstream`, others follow it (skip with `ceph_ntp_mode: system`) |
| `ceph_bootstrap` | admin_node | `ceph.cephadm.cephadm_bootstrap` unless `/etc/ceph/ceph.conf` exists |
| `ceph_ssh_key` | all | cephadm public key → `authorized_keys` of `cephadm_ssh_user`; `ceph cephadm set-user` |
| `ceph_hosts` | all (delegated to admin_node) | `ceph.cephadm.ceph_orch_host` with `ceph_labels`, `_admin` for group `admin` |
| `ceph_services` | admin_node | `ceph.cephadm.ceph_config` for `config.yml`, `ceph.cephadm.ceph_orch_apply` for `services/*.yml`, wait for mons/OSDs |
| `ceph_pools` | admin_node | create pools of `pools.yml`, enable application, `rbd pool init`, wait for HEALTH_OK |
| `ceph_verify` | admin_node | HEALTH_OK, quorum, OSDs up/in, PGs active+clean, RBD bench |
| (`tasks_from: drift.yml`) | admin_node | read-only: hosts/labels, service specs, declared config, pools vs the cluster; lists every difference |

## Variables

All with defaults and comments in [defaults/main.yml](defaults/main.yml); validated by
[meta/argument_specs.yml](meta/argument_specs.yml). Most often overridden:

| Variable | Default |
|---|---|
| `admin_node` | first host of group `admin` |
| `cephadm_ssh_user` | `ansible_user` |
| `ceph_release` / `ceph_image` | `20.2.4` / `quay.io/ceph/ceph:v<release>` |
| `ceph_ip`, `ceph_labels` (per host) | `ansible_host`, `[]` |
| `ceph_cluster_network` | `""` |
| `ceph_monitoring_stack` | `false` |
| `ceph_ntp_mode`, `ntp_upstream`, `ntp_server` | `cluster`, `[pool.ntp.org]`, `admin_node` |
| `ntp_allow` | `ceph_ip` of every host in group `ceph_inventory_group` (`ceph`), independent of `--limit`; or CIDRs |
| `ntp_wait_retries` / `ntp_max_offset` | 12 (× 5 s = 1 min) / 0.05 s — time check fails fast with sources and a hint |
| `ceph_state_dir` | `<inventory dir>/ceph`: desired state files `services/*.yml` (native cephadm specs), `config.yml`, `pools.yml` |
| `ceph_service_specs` / `ceph_config` / `ceph_pools` | read from those files; set one in group_vars only to bypass its file |
| `ceph_drift_fail` | `true` — `tasks/drift.yml` (playbooks/drift.yml) fails when the cluster differs |
| `ceph_wait_retries` / `ceph_wait_delay` / `ceph_verify_retries` | 12 / 5 s / 18 — deployment waits up to 1 min, verification up to 90 s |

## cephadm-ansible v5.0.3 workarounds inside the role

- bootstrap re-run check done by the role (`/etc/ceph/ceph.conf`), the module looks in `/var/lib/ceph`;
- `initial_dashboard_user/password`, `skip_monitoring_stack` instead of the silently ignored deprecated parameters;
- OSD spec carries cephadm defaults (`filter_logic`, `objectstore`) so `ceph_orch_apply` stays unchanged on re-runs.
