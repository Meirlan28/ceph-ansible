# ceph-ansible

Full Ceph deployment with **cephadm**, driven by Ansible. Declarative and idempotent: re-running
`site.yml` converges an existing cluster, nothing is redone.

- Ceph **20.2.4 Tentacle** (cephadm RPM from download.ceph.com + `quay.io/ceph/ceph:v20.2.4`)
- EL9 / EL10 (AlmaLinux, Rocky, RHEL), podman
- mon on every `mons` host, mgr (2 by default), one OSD service per host pinned to explicit
  `/dev/disk/by-id/...` devices, dashboard with a password passed only through a file, RBD pool
- tuned for small VMs (OSD memory target 1 GiB, monitoring stack off)

Only `ansible-core` is required (no collections).

## With VMwareLab (two commands from zero)

```powershell
.\lab.cmd up ceph        # 3 empty AlmaLinux 10 nodes with a blank 10 GB disk each
.\lab.cmd deploy ceph    # this repo: full Ceph, HEALTH_OK, RBD pool, dashboard
```

`lab.cmd deploy` runs this playbook on the first node (it acts as the Ansible controller), generates
the inventory with each node's disk IDs and creates/stores the dashboard password (DPAPI):
`lab.cmd secret ceph ceph_dashboard_password`. Other playbooks:

```powershell
.\lab.cmd verify ceph                                  # infra checks + verify.yml
.\lab.cmd deploy ceph purge.yml -e confirm_purge=yes   # remove Ceph, wipe OSD disks
.\lab.cmd deploy ceph site.yml --tags ...              # any ansible-playbook arguments
```

## Standalone

```bash
cp inventory/example.yml inventory/mylab.local.yml    # fill hosts and ceph_osd_devices
ansible-playbook -i inventory/mylab.local.yml site.yml -e ceph_dashboard_password='...'
ansible-playbook -i inventory/mylab.local.yml verify.yml
```

Hosts must allow SSH + sudo for the Ansible user and resolve each other's hostnames (inventory
names must equal the machines' hostnames, cephadm checks this).

## Inventory contract

| Group / variable | Meaning |
|---|---|
| `mons` | monitor hosts; the first one bootstraps the cluster |
| `mgrs` | manager candidates (`ceph_mgr_count` of them get a mgr) |
| `osds` | OSD hosts |
| `ceph_osd_devices` (host) | list of stable device paths; empty + `ceph_osd_auto: true` = all blank disks |
| `ceph_public_ip` (host) | address daemons bind to; defaults to `ansible_host` |
| `ceph_dashboard_password` | required when the dashboard is enabled |

Everything else (release, pools, `ceph config` options, timeouts) is in `group_vars/all.yml`.

## What the playbooks do

| Play | Hosts | Steps |
|---|---|---|
| prereqs | all | Ceph repo, `cephadm` RPM, podman, lvm2, chrony (bootstrap host serves NTP to the others; waits for < 10 ms), firewalld services, image pull |
| bootstrap | first mon | `cephadm bootstrap --mon-ip ...` (once), `ceph config set` only for values that differ |
| hosts | all | cephadm root SSH key, `cluster-spec.yml` (hosts + labels, mon, mgr, OSD per host) → `ceph orch apply -i` |
| services | first mon | wait for mon quorum and all OSDs up/in, dashboard module, self-signed cert, admin user |
| pools | first mon | pools + applications (`rbd pool init`), wait for `HEALTH_OK` (fails with `ceph health detail` otherwise) |

`verify.yml`: HEALTH_OK, quorum = `mons`, OSDs up/in, all PGs `active+clean`, RBD write/read bench on a
temporary image. `purge.yml`: `cephadm rm-cluster --zap-osds` on every host (needs `-e confirm_purge=yes`).

Ceph CLI on any node: `sudo cephadm shell -- ceph -s` (all hosts carry the `_admin` label).

## Notes from testing

- **Clock skew.** Mons allow 50 ms of drift. VMs syncing to internet NTP independently drifted 50-120 ms apart
  (MON_CLOCK_SKEW). With `ceph_ntp_mode: cluster` (default) the bootstrap host is the time source for the
  others (LAN, polled every 4-16 s, step when > 100 ms); offsets stay at a few ms. `system` keeps chrony as is.
- **Host order.** Ansible does not keep host order inside groups, so `mons[0]` is not stable between runs; the
  bootstrap host is `ceph_bootstrap_host` (default: alphabetically first mon), exposed as group `ceph_bootstrap`.
- **Restarts.** cephadm re-checks hosts every 60 s here (default 600 s), so a host that was still booting does
  not keep CEPHADM_HOST_CHECK_FAILED; `verify.yml` waits up to 3 minutes for HEALTH_OK.

## Changing the cluster

- Add a node: add it to the inventory groups, re-run `site.yml` (spec changes → `orch apply`).
- Removing a node is not automated: drain first (`ceph orch host drain HOST`), then remove it.
- Upgrade: bump `ceph_release`, run `prereqs` for the new cephadm, then `ceph orch upgrade start --image ...`.
