# ceph-cluster

Ceph cluster as code on top of the upstream tooling:

- **cephadm** — Ceph's own orchestrator (bootstrap, `ceph orch`, service specs);
- **cephadm-ansible** — the official Ceph project playbooks and modules, pinned to **v5.0.3** (`tentacle`
  branch) and installed as the `ceph.cephadm` collection from git (`requirements.yml`);
- Ceph **20.2.4 Tentacle** from download.ceph.com, image `quay.io/ceph/ceph:v20.2.4`, EL9/EL10.

This repo only adds what upstream leaves to you: the inventory, cluster settings, service specs and a few
playbooks that call the upstream modules (`ceph.cephadm.cephadm_bootstrap`, `ceph_orch_host`,
`ceph_orch_apply`, `ceph_config`).

**How to use it step by step, with explanations: [docs/RUNBOOK.md](docs/RUNBOOK.md)** (in Russian).

## Layout

```
requirements.yml            pinned collections: ceph.cephadm (git v5.0.3), ansible.posix, community.general
ansible.cfg                 inventory, ./collections, SSH options, ansible.log
inventory/lab/
  hosts.yml                 nodes, ceph_ip, ceph_labels; groups ceph / admin / clients
  group_vars/all/main.yml   release, image, user, NTP, ceph config, pools
  group_vars/all/vault.yml  secrets (ansible-vault, you create it)
services/                   cephadm service specs: mon, mgr, crash, osd (drive group)
playbooks/
  10-preflight.yml          upstream cephadm-preflight.yml (repo, packages, chrony, firewalld)
  20-time.yml               one time source for all nodes (mons allow 50 ms drift)
  30-bootstrap.yml          ceph.cephadm.cephadm_bootstrap on admin_node
  40-ssh-key.yml            upstream cephadm-distribute-ssh-key.yml
  50-hosts.yml              ceph.cephadm.ceph_orch_host (+ labels)
  60-services.yml           ceph.cephadm.ceph_config + ceph_orch_apply services/*.yml, wait for mons/OSDs
  70-pools.yml              pools, wait for HEALTH_OK
  90-verify.yml             health assertions + RBD bench
  99-purge.yml              upstream cephadm-purge-cluster.yml (-e fsid=...)
site.yml                    10 → 70 in order
```

## Short version

On the admin node, as the deployment user, from the repo directory:

```bash
ansible-galaxy collection install -r requirements.yml
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault-pass                # file with the vault password, mode 600
ansible-vault create inventory/lab/group_vars/all/vault.yml   # vault_ceph_dashboard_password: ...
ansible all -m ping
ansible-playbook site.yml
ansible-playbook playbooks/90-verify.yml
```

Every playbook is idempotent; a second run reports `changed=0` (verified).

Upstream v5.0.3 quirks this repo works around (module_utils path, bootstrap re-run check, deprecated dashboard
parameters, spec comparison, purge `client_group`) are listed in the runbook.

## Using it for another environment

Copy `inventory/lab` to `inventory/<env>`, set hosts, `ceph_ip`, `ceph_labels`, `admin_node`, networks and NTP
servers, adjust `services/osd.yml` device filters, and run with `-i inventory/<env>/hosts.yml`.
The hosts need SSH + passwordless sudo for `ansible_user`/`cephadm_ssh_user` from the Ansible host.
