# Runbook: Ceph на cephadm-ansible

Пошаговая раскатка Ceph 20.2 (Tentacle) на трёх нодах лаба официальным инструментарием:
**cephadm** (оркестратор Ceph) + **cephadm-ansible** (официальные плейбуки и модули проекта Ceph).
Каждый шаг: что делаем → команда → что произошло → как проверить.

---

## 0. Роли и границы ответственности

```
 Windows (VMwareLab)                 ceph-1  = admin node             ceph-2, ceph-3
 ─────────────────────               ───────────────────────          ──────────────
 lab.cmd up ceph                     • ansible host (здесь запускаем   • storage nodes
   VM, сеть, диски,                    ansible-playbook)
   пользователь ceph-admin,          • bootstrap host (первый mon)
   SSH-доверие ceph-1 → все          • admin host (/etc/ceph/ключ admin)
```

| Слой | Чем делается | В проде это |
|---|---|---|
| Железо/VM, сеть, диск под OSD | `lab.cmd up ceph` | Terraform / MAAS / закупка серверов |
| ОС, пользователь `ceph-admin` с sudo, SSH-ключ admin-ноды | `lab.cmd` (provisioning) | kickstart / cloud-init / FreeIPA |
| Пакеты, время, кластер Ceph, сервисы, пулы | **этот репозиторий** (ты руками) | Ansible из git (CI, AWX/AAP или с admin-ноды) |

Термины cephadm-ansible:
- **admin host** — где лежат `/etc/ceph/ceph.conf` и `ceph.client.admin.keyring` (метка `_admin`), группа `[admin]` в инвентаре;
- **bootstrap host** — где выполняется `cephadm bootstrap` (обычно первый admin host, у нас `admin_node: ceph-1`);
- **ansible host** — откуда запускаются плейбуки (у нас тот же ceph-1).

Все Ansible-команды ниже выполняются **на ceph-1 под пользователем `ceph-admin` из каталога `~/ceph-cluster`**.

---

## 1. Инфраструктура и вход на admin node

На Windows, из `E:\VMwareLab`:

```powershell
.\lab.cmd up ceph        # 3 чистые ноды AlmaLinux 10
.\lab.cmd verify ceph    # должно быть: DATA_DISK ... BLANK (×3), ADMIN_ACCESS_OK (×3), PASS
.\lab.cmd ssh ceph-1 --admin
```

`--admin` входит как `ceph-admin` ключом кластера (`clusters\ceph\pki\admin_ed25519`). Без lab.cmd то же самое:
`ssh -i E:\VMwareLab\clusters\ceph\pki\admin_ed25519 ceph-admin@192.168.228.21`.

Проверь, что admin-нода видит все ноды по SSH и имеет sudo (это делает и `lab.cmd verify`):

```bash
for h in ceph-1 ceph-2 ceph-3; do ssh $h 'hostname; sudo -n true && echo sudo ok'; done
```

### Две сети: public и cluster

| Сеть | Подсеть | NIC | Кто ходит |
|---|---|---|---|
| public | 192.168.228.0/24 (VMnet1, сеть lab `lab`) | eth1 | клиенты, mon, mgr, дашборд, front-адрес OSD |
| cluster | 192.168.230.0/24 (VMnet2, сеть lab `cluster`) | eth2 | только OSD ↔ OSD: репликация, recovery, heartbeat (back-адрес OSD) |

Так делают в проде: трафик восстановления после отказа диска не забивает сеть, по которой ходят клиенты.
Сеть задаётся одной переменной `ceph_cluster_network` (inventory, group_vars): bootstrap передаёт её в
`cephadm bootstrap --cluster-network`, роль держит `ceph config set global cluster_network` (для кластера, развёрнутого
без неё, ставит и перезапускает OSD), фаза verify проверяет, что back-адрес каждого OSD лежит в этой сети.

Проверка после раскатки:

```bash
sudo ceph config get osd cluster_network
sudo ceph osd metadata 0 | grep -E '"(front|back)_addr"'   # front 192.168.228.x, back 192.168.230.x
```

---

## 2. Код на admin node

В проде: `git clone git@gitlab.example.com:infra/ceph-cluster.git`. В лабе git-сервера нет, копируем с Windows
(в PowerShell на Windows):

```powershell
scp -r -i E:\VMwareLab\clusters\ceph\pki\admin_ed25519 E:\ceph-cluster ceph-admin@192.168.228.21:~/
```

На ceph-1:

```bash
chmod -R go-w ~/ceph-cluster   # scp с Windows делает каталог world-writable, и тогда Ansible
cd ~/ceph-cluster              # молча игнорирует ansible.cfg (ищи WARNING про world writable)
ansible --version | head -3    # config file = /home/ceph-admin/ceph-cluster/ansible.cfg
```

Структура:

| Путь | Что это |
|---|---|
| `ansible.cfg` | инвентарь по умолчанию, `collections_path=./collections`, SSH-настройки, лог `ansible.log` |
| `requirements.yml` | **запиненные** зависимости: cephadm-ansible v5.0.3, ansible.posix, community.general |
| `inventory/lab/hosts.yml` | ноды, их IP (`ceph_ip`), метки (`ceph_labels`), группы `ceph`, `admin`, `clients` |
| `inventory/lab/group_vars/all/main.yml` | только то, что зависит от окружения: пользователь, admin_node, версия Ceph, сети, дашборд |
| `inventory/lab/group_vars/all/vault.yml` | секреты, зашифрованы ansible-vault (создашь на шаге 4) |
| `roles/ceph_cluster/` | **роль** со всей логикой поверх cephadm-ansible: фазы time → bootstrap → ssh_key → hosts → services → pools → verify (`tasks/`), дефолты с комментариями (`defaults/main.yml`, там же **спецификации сервисов cephadm** `ceph_service_specs`, `ceph_config`, пулы), проверка входных переменных (`meta/argument_specs.yml`) |
| `playbooks/ceph.yml` | применяет роль к группе `ceph`; фазы запускаются тегами `ceph_*` |
| `playbooks/10-preflight.yml`, `99-purge.yml` | обёртки над плейбуками cephadm-ansible (пакеты; снос) |
| `site.yml` | preflight + роль целиком (когда уже понимаешь, что делает каждая фаза) |

---

## 3. Зависимости

На Windows Ansible как управляющий узел нативно не поддерживается. Используй WSL 2, например Ubuntu.
В PowerShell установи WSL (если он ещё не установлен):

```powershell
wsl --install -d Ubuntu
```

Открой Ubuntu из меню Пуск, затем выполни в терминале WSL:

```bash
sudo apt update
sudo apt install -y python3-venv python3-pip git
cd /mnt/e/ceph-cluster
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
```

Если проект находится на другом диске Windows, замени `/mnt/e/ceph-cluster` на соответствующий путь WSL
(например, `C:\ceph-cluster` доступен как `/mnt/c/ceph-cluster`). Все команды Ansible запускай из WSL,
не из PowerShell.

Для управляющей ноды нужен Python 3.10 или новее. `requirements.txt` устанавливает
`ansible-core==2.16.14` в локальное виртуальное окружение проекта, не затрагивая системный Python.
`requirements.yml` отдельно устанавливает Ansible-коллекции в `./collections`:
- `ceph.cephadm` — это и есть cephadm-ansible (на Galaxy не опубликован, поэтому ставится из git по тегу `v5.0.3`,
  релиз ветки `tentacle` под Ceph 20.2). Его модули дальше вызываются по полному имени `ceph.cephadm.*`;
- `ansible.posix`, `community.general` — нужны плейбукам cephadm-ansible. Версия community.general ограничена `<11`:
  более новые не поддерживают ansible-core 2.16, который идёт в EL9/EL10.

Проверка:

```bash
ansible-galaxy collection list -p collections
ansible-doc ceph.cephadm.cephadm_bootstrap | head -30     # документация модуля
ls collections/ansible_collections/ceph/cephadm/*.yml     # плейбуки upstream
```

Почему пиним: `ansible-galaxy install` без версий завтра поставит другое — так и случилось бы с community.general 12.

---

## 4. Секреты: ansible-vault

Пароль дашборда не хранится открытым текстом. Сначала пароль самого vault — в файл, доступный только тебе,
и переменная окружения, чтобы Ansible брал его сам:

```bash
read -rsp 'Vault password: ' p; echo; printf '%s\n' "$p" > ~/.vault-pass; unset p; chmod 600 ~/.vault-pass
echo 'export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault-pass' >> ~/.bashrc && source ~/.bashrc
```

Теперь зашифрованный файл секретов:

```bash
ansible-vault create inventory/lab/group_vars/all/vault.yml
```

Откроется редактор (vi), впиши и сохрани:

```yaml
vault_ceph_dashboard_password: "Пароль-Для-Дашборда-1"
```

В `main.yml` стоит `ceph_dashboard_password: "{{ vault_ceph_dashboard_password }}"` — соглашение "vault_*-переменные
в vault.yml, ссылки на них в main.yml": по main.yml видно, какие секреты есть, но не их значения. Файл можно
коммитить, он зашифрован (`head -1 inventory/lab/group_vars/all/vault.yml` → `$ANSIBLE_VAULT;1.1;AES256`).
Посмотреть/изменить: `ansible-vault view ...`, `ansible-vault edit ...`.

Важно: раз в `group_vars` есть зашифрованный файл, пароль vault нужен **любой** команде ansible (даже `ping`).
С `ANSIBLE_VAULT_PASSWORD_FILE` это происходит само; без него добавляй `--ask-vault-pass`.
`~/.vault-pass` в репозиторий не попадает (`.gitignore`); в CI его роль играет секрет пайплайна.

---

## 5. Инвентарь и доступ

```bash
ansible-inventory --graph
ansible all -m ping
ansible all -m command -a 'id'          # become: true → uid=0(root)
```

`hosts.yml`: имена нод должны совпадать с их `hostname` (cephadm это проверяет). `ceph_labels` — метки, по которым
`ceph_service_specs` (defaults роли) размещают демоны: mon на всех трёх, mgr на ceph-1/2, osd на всех.

---

## 6. Preflight (cephadm-ansible)

```bash
ansible-playbook playbooks/10-preflight.yml
```

Upstream-плейбук `cephadm-preflight.yml` на каждой ноде: репозиторий `download.ceph.com/rpm-20.2.4/el10`
(из `ceph_origin: community` + `ceph_release`), EPEL (на не-RHEL), пакеты `cephadm ceph-common podman lvm2 chrony
firewalld`, включает chronyd и firewalld.

Проверка (на любой ноде):

```bash
rpm -q cephadm ceph-common podman
dnf repolist | grep -iE 'ceph|epel'
ceph --version            # 20.2.4 tentacle (stable)
```

---

## 7. Время

```bash
ansible-playbook playbooks/ceph.yml --tags ceph_time
```

Мониторы допускают расхождение часов 50 мс. В проде все ноды смотрят в одни корпоративные NTP. В лабе ВМ,
синхронизирующиеся с интернетом по отдельности, расходились на 50–120 мс (HEALTH_WARN MON_CLOCK_SKEW), поэтому
ceph-1 раздаёт время остальным (`ntp_server`), а сам ходит в `pool.ntp.org`.

Проверка: `chronyc tracking` (на ceph-2/3 `Reference ID ... (ceph-1)`, `System time` — единицы мс), `chronyc sources`.

---

## 8. Bootstrap

```bash
ansible-playbook playbooks/ceph.yml --tags ceph_bootstrap
```

Модуль `ceph.cephadm.cephadm_bootstrap` = `cephadm bootstrap --mon-ip 192.168.228.21 --ssh-user ceph-admin
--image quay.io/ceph/ceph:v20.2.4 --skip-monitoring-stack ...` на ceph-1:
- генерирует fsid, поднимает **первый mon и mgr** в podman-контейнерах (systemd-юниты `ceph-<fsid>@mon.ceph-1` …);
- кладёт в `/etc/ceph`: `ceph.conf`, `ceph.client.admin.keyring`, `ceph.pub` (SSH-ключ оркестратора);
- `--ssh-user ceph-admin`: оркестратор будет ходить на ноды не root'ом, а ceph-admin + sudo;
- если `/etc/ceph/ceph.conf` уже есть, шаг пропускается (модуль идемпотентен).

Проверка на ceph-1:

```bash
sudo ceph -s                    # 1 mon, 1 mgr, 0 osd, HEALTH_WARN (OSD ещё нет) — это нормально
sudo ceph orch host ls          # пока только ceph-1
sudo ls /etc/ceph
sudo cephadm ls | grep '"name"' # демоны на этой ноде
sudo ceph config get mgr mgr/cephadm/ssh_user
```

`ceph` работает на хосте без `cephadm shell`, потому что preflight поставил `ceph-common`, а ключ admin лежит в /etc/ceph.

---

## 9. SSH-ключ оркестратора

```bash
ansible-playbook playbooks/ceph.yml --tags ceph_ssh_key
```

Фаза роли `tasks/ssh_key.yml` делает то же, что upstream `cephadm-distribute-ssh-key.yml`: берёт публичный ключ
(`ceph cephadm get-pub-key` на `admin_node`), добавляет его в `authorized_keys` пользователя `cephadm_ssh_user`
(ceph-admin) на всех нодах и делает `ceph cephadm set-user`. В роли, а не отдельным плейбуком — чтобы роль шла
целиком от bootstrap до пулов одним прогоном. Без этого mgr не сможет управлять ceph-2/3.

Проверка: `ssh ceph-2 'grep -c ceph ~/.ssh/authorized_keys'` и после следующего шага
`sudo ceph cephadm check-host ceph-2`.

---

## 10. Хосты

```bash
ansible-playbook playbooks/ceph.yml --tags ceph_hosts
```

`ceph.cephadm.ceph_orch_host` = `ceph orch host add <имя> <ip> --labels ...` для каждой ноды из группы `ceph`
(+ метка `_admin` для группы `admin`). Как только хост добавлен, cephadm начинает размещать на нём демоны,
подходящие по уже применённым спецификациям (например, crash).

Проверка: `sudo ceph orch host ls` — 3 хоста, метки mon/mgr/osd, у ceph-1 `_admin`.

---

## 11. Конфигурация и сервисы

```bash
ansible-playbook playbooks/ceph.yml --tags ceph_services
```

1. `ceph.cephadm.ceph_config` — `ceph config set` для списка `ceph_config` (память OSD 1 GiB и т.п.). Ставится
   **до** создания OSD, чтобы они сразу стартовали с нужным лимитом.
2. `ceph.cephadm.ceph_orch_apply` — `ceph orch apply` для каждой спецификации из `ceph_service_specs`. Это декларативно: ты описываешь
   "mon на всех хостах с меткой mon", оркестратор сам добивается этого состояния и держит его.
3. Ожидание: кворум из всех mon, OSD по числу osd-хостов в состоянии up/in.

OSD-спека (последний элемент `ceph_service_specs` в `roles/ceph_cluster/defaults/main.yml`) — drive group: на хостах с меткой osd взять все *доступные* диски. Доступный = без
разделов, LVM и ФС (в лабе это пустой диск 10 GB; системный диск недоступен). Перед применением в проде смотрят:

```bash
sudo ceph orch device ls                                    # какие диски cephadm видит и годны ли они
sudo ceph orch ls osd --export > /tmp/osd.yml && sudo ceph orch apply -i /tmp/osd.yml --dry-run   # что он сделает
```

Проверка:

```bash
sudo ceph orch ls               # сервисы: mon 3/3, mgr 2/2, crash 3/3, osd 3
sudo ceph orch ps               # все демоны и где они работают
sudo ceph osd tree              # 3 OSD, по одному на хост, up
```

---

## 12. Пулы

```bash
ansible-playbook playbooks/ceph.yml --tags ceph_pools
```

Создаёт недостающие пулы из `ceph_pools` (`rbd`, 32 PG), включает приложение и делает `rbd pool init`, затем ждёт
HEALTH_OK. Проверка: `sudo ceph osd pool ls detail`, `sudo ceph -s` → `HEALTH_OK`, `pgs: 33 active+clean`.

---

## 13. Проверка и дашборд

```bash
ansible-playbook playbooks/ceph.yml --tags ceph_verify
```

HEALTH_OK, кворум, OSD up/in, все PG active+clean, RBD bench на временном образе.

Дашборд: https://192.168.228.21:8443 (сертификат самоподписанный), логин `admin`, пароль из vault.
При первом входе дашборд потребует сменить пароль — так ведёт себя `cephadm bootstrap` по умолчанию.

Повторный запуск любого шага (или всей роли `ansible-playbook playbooks/ceph.yml`, или `site.yml`) ничего не меняет: `changed=0`.

Время на 3 ВМ (проверено): preflight ~2 мин, time 15 с, bootstrap ~1.5 мин, ssh-key 5 с, hosts 20 с,
services ~3 мин, pools 30 с — около 7 минут на всё; `site.yml` на уже развёрнутом кластере — около минуты.

---

## Типовые операции

| Задача | Как |
|---|---|
| Поменять настройку Ceph | переопредели `ceph_config` в group_vars (целиком список, дефолт — в defaults роли) → `playbooks/ceph.yml --tags ceph_services` |
| Поменять размещение демонов | переопредели `ceph_service_specs` в group_vars или поменяй `ceph_labels` → `--tags ceph_hosts,ceph_services` |
| Добавить ноду | VMwareLab: нода в `clusters\ceph\cluster.json` → `lab.cmd up ceph`; здесь: нода в `hosts.yml` → `10`, `20`, `40`, `50`, `60` с `--limit` новой ноды где возможно |
| Вывести ноду | `sudo ceph orch host drain ceph-3` → дождаться `ceph orch ps ceph-3` пусто → `sudo ceph orch host rm ceph-3` → убрать из инвентаря; потом из lab (`up --prune`) |
| Обновить Ceph | `sudo ceph orch upgrade start --image quay.io/ceph/ceph:vX.Y.Z`, следить `ceph orch upgrade status`; потом поднять `ceph_release` и прогнать preflight (пакеты cephadm/ceph-common) |
| Снести и начать заново | `ansible-playbook playbooks/99-purge.yml -e fsid=$(sudo ceph fsid)` (спросит подтверждение; без вопроса `-e ireallymeanit=yes`). Удаляет кластер, **обнуляет OSD-диски и удаляет пакеты** cephadm/ceph-common → начинать снова с шага 6. Или в лабе `lab.cmd destroy ceph` + `up` (тогда с шага 1) |

## Особенности cephadm-ansible v5.0.3 (на что наткнулись при проверке)

Апстрим — это не чёрный ящик: когда что-то ведёт себя странно, читай код модуля
(`collections/ansible_collections/ceph/cephadm/plugins/modules/*.py`). Всё ниже уже учтено в репозитории.

| Что | Симптом | Как учтено |
|---|---|---|
| Коллекция не опубликована на Galaxy | `ansible-galaxy collection install ceph.cephadm` → not found | ставится из git по тегу (`requirements.yml`) |
| community.general ≥ 11 не поддерживает ansible-core 2.16 | `Collection ... does not support Ansible version` | `version: ">=10.0,<11.0"` |
| Модули импортируют `ansible.module_utils.ceph_common` по-старому | `No module named 'module_utils'` | `module_utils = ...` в `ansible.cfg` |
| Проверка "уже bootstrapped" смотрит `/var/lib/ceph/ceph.conf`, а cephadm пишет `/etc/ceph` | повторный bootstrap падает | свой `stat /etc/ceph/ceph.conf` перед модулем |
| `dashboard_user/password` — устаревшие алиасы, модуль их молча игнорирует | дашборд со случайным паролем | `initial_dashboard_user/password`, `skip_monitoring_stack` |
| `ceph_orch_apply` сравнивает спеку по полям с `ceph orch ls --export` | OSD-спека "changed" на каждом запуске | в OSD-спеке (`ceph_service_specs`) явно `filter_logic`, `objectstore` |
| Purge читает `client_group`, который задаёт только preflight | `'client_group' is undefined` | `client_group: clients` в group_vars |

## Работа с кодом дальше

Рабочей копией считай `~/ceph-cluster` на ceph-1: там уже есть `.git` (скопирован с Windows), `vault.yml` и
`collections/`. Правишь там, коммитишь там (`git add -A && git commit`). Если удобнее править на Windows — копируй
обратно только изменённые файлы (`scp file ceph-admin@192.168.228.21:ceph-cluster/...`), не затирая `vault.yml`.
В проде вместо этого: git-сервер, ветка, merge request, пайплайн.

## Отладка

```bash
ansible-playbook playbooks/ceph.yml --tags ceph_services -v   # -vvv: SSH и аргументы модулей
ansible-playbook playbooks/ceph.yml --check --diff
ansible-playbook playbooks/ceph.yml --start-at-task "ceph orch apply osd"
ansible-playbook playbooks/ceph.yml --list-tasks --list-tags    # что и в каком порядке сделает роль
less ansible.log                                        # лог всех запусков (ansible.cfg)
sudo ceph health detail
sudo ceph log last cephadm                              # что делал оркестратор
sudo cephadm logs --name mon.ceph-1                     # journald конкретного демона
sudo podman ps                                          # контейнеры на ноде
```

Где что лежит на нодах: `/etc/ceph` (конфиг и ключи на admin-хостах), `/var/lib/ceph/<fsid>/` (данные демонов),
юниты `ceph-<fsid>@<daemon>.service`, контейнеры podman с образом `quay.io/ceph/ceph:v20.2.4`.

## Лаб vs прод

| | Лаб | Обычно в проде |
|---|---|---|
| Код | scp с Windows | git, merge request, CI (ansible-lint, `--check`) |
| Ansible host | ceph-1 | отдельная deploy-нода / AWX / AAP |
| Время | ceph-1 раздаёт остальным | корпоративные NTP на всех нодах |
| Сети | одна public | public + отдельная cluster network (`ceph_cluster_network`) |
| OSD | `all: true`, 1 диск 10 GB | фильтры по rotational/size/model, DB/WAL на NVMe |
| Мониторинг | выключен (RAM) | `monitoring: true` или внешний Prometheus |
| Секреты | vault-пароль вводится руками | vault в CI/AAP, HashiCorp Vault |
