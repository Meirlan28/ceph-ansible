# ceph-cluster

Ceph cluster as code on top of the upstream tooling:

- **cephadm** — Ceph's own orchestrator (bootstrap, `ceph orch`, service specs);
- **cephadm-ansible** — the official Ceph project playbooks and modules, pinned to **v5.0.3** (`tentacle`
  branch) and installed as the `ceph.cephadm` collection from git (`requirements.yml`);
- Ceph **20.2.4 Tentacle** from download.ceph.com, image `quay.io/ceph/ceph:v20.2.4`, EL9/EL10.

This repo only adds what upstream leaves to you: the inventory and the role `ceph_cluster`, which wraps the
upstream modules (`ceph.cephadm.cephadm_bootstrap`, `ceph_orch_host`,
`ceph_orch_apply`, `ceph_config`).

**How to use it step by step, with explanations: [docs/RUNBOOK.md](docs/RUNBOOK.md)** (in Russian).

## Layout

```
requirements.yml            pinned collections: ceph.cephadm (git v5.0.3), ansible.posix, community.general
requirements.txt            ansible-core 2.16.14 for the controller's Python virtual environment
ansible.cfg                 inventory, ./collections, SSH options, ansible.log
inventory/lab/
  hosts.yml                 nodes, ceph_ip, ceph_labels; groups ceph / admin / clients
  group_vars/all/main.yml   environment only: user, admin_node, release, networks, dashboard
  group_vars/all/vault.yml  secrets (ansible-vault, you create it)
  ceph/                     desired cluster state: services/*.yml (native cephadm specs), config.yml, pools.yml
roles/ceph_cluster/         the role: time, bootstrap, ssh_key, hosts, services, pools, verify (tags ceph_*), drift
                            defaults/main.yml: knobs with comments; state is read from inventory/<env>/ceph
playbooks/
  10-preflight.yml          upstream cephadm-preflight.yml (repo, packages, chrony, firewalld)
  ceph.yml                  role ceph_cluster on group ceph
  drift.yml                 read-only: cluster vs inventory + inventory/<env>/ceph, lists differences
  99-purge.yml              upstream cephadm-purge-cluster.yml (-e fsid=...)
site.yml                    preflight + role
```

## Short version

### Windows

Ansible нельзя штатно запустить как управляющий узел в Windows. Используйте WSL 2 (например, Ubuntu):

```powershell
wsl --install -d Ubuntu
```

После установки откройте Ubuntu (WSL), перейдите в проект и установите зависимости в Linux-среде:

```bash
sudo apt update
sudo apt install -y python3-venv python3-pip git
cd /mnt/e/ceph-cluster
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
```

Дальше запускайте `ansible-playbook` из этого же WSL-терминала с активированным `.venv`.

### Linux

On the admin node, as the deployment user, from the repo directory:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault-pass                # file with the vault password, mode 600
ansible-vault create inventory/lab/group_vars/all/vault.yml   # vault_ceph_dashboard_password: ...
ansible all -m ping
ansible-playbook site.yml                              # verification runs as the role's last phase
```

Every playbook is idempotent; a second run reports `changed=0` (verified).

The cluster state lives in the repo: hosts and labels in `hosts.yml`, cephadm service specs (native format, as
`ceph orch ls --export`), `ceph config` and pools in `inventory/lab/ceph/`. Change a file, run
`ansible-playbook playbooks/drift.yml` to see the difference, apply with `playbooks/ceph.yml --tags ...`, commit.

Upstream v5.0.3 quirks this repo works around (module_utils path, bootstrap re-run check, deprecated dashboard
parameters, spec comparison, purge `client_group`) are listed in the runbook.

## Using it for another environment

Copy `inventory/lab` to `inventory/<env>`, set hosts, `ceph_ip`, `ceph_labels`, `admin_node`, networks and NTP
servers, adjust `inventory/<env>/ceph/` (OSD device filters in `services/osd.yml`) if needed, and run with `-i inventory/<env>/hosts.yml`.
The hosts need SSH + passwordless sudo for `ansible_user`/`cephadm_ssh_user` from the Ansible host.
