# PostgreSQL 3-Node High Availability Cluster Setup
## Autobase (postgresql_cluster) + HAProxy + Keepalived on Ubuntu 24.04

---

## Environment

| Node | IP Address | Role |
|------|------------|------|
| pg-master | 10.40.71.50 | Patroni Leader (Primary) |
| pg-worker1 | 10.40.71.51 | Patroni Replica |
| pg-worker2 | 10.40.71.52 | Patroni Replica |
| VIP | 10.40.70.200 | Virtual IP managed by Keepalived |

All nodes: Ubuntu 24.04 LTS, 2 CPU, 4 GB RAM, 25 GB Disk(Tested)

---

## Architecture Overview

```
Client
  |
10.40.70.200:5000  (VIP - managed by Keepalived, currently bound to the Leader node)
  |
HAProxy  (running on all 3 nodes)
  |
  +--- pg-master  (10.40.71.50)  Leader   - PostgreSQL read/write
  +--- pg-worker1 (10.40.71.51)  Replica  - PostgreSQL read-only / streaming
  +--- pg-worker2 (10.40.71.52) Replica  - PostgreSQL read-only / streaming
```

**Port map:**

| Port | Service | Description |
|------|---------|-------------|
| 5000 | HAProxy | Primary (write) |
| 5001 | HAProxy | Replica (read-only) |
| 5432 | PostgreSQL | Direct connection (bypass HAProxy) |
| 6432 | PgBouncer | Connection pooling |
| 8008 | Patroni API | Health check endpoint |
| 2379 | etcd | Client communication |
| 2380 | etcd | Peer communication |


---

## Step 1 - Set Hostnames

Run the following command on each node individually.

**On pg-master (10.40.71.50):**
```bash
hostnamectl set-hostname pg-master
exec bash
```

**On pg-worker1 (10.40.71.51):**
```bash
hostnamectl set-hostname pg-worker1
exec bash
```

**On pg-worker2 (10.40.71.52):**
```bash
hostnamectl set-hostname pg-worker2
exec bash
```

Verify:
```bash
hostname
```

---

## Step 2 - Update /etc/hosts on All Nodes

Run the following on **all three nodes**:

```bash
vim /etc/hosts
```

Append the following lines at the end of the file:

```
10.40.71.50  pg-master
10.40.71.51  pg-worker1
10.40.71.52 pg-worker2
```

Save and exit (`:wq`).

Verify name resolution from pg-master:
```bash
ping -c2 pg-worker1
ping -c2 pg-worker2
```

---

## Step 3 - Generate SSH Key on pg-master

Ansible will manage all nodes from pg-master over SSH. A passwordless key pair is required.

**On pg-master:**
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
```

Display the public key:
```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the output. You will need it in the next step.

---

## Step 4 - Distribute SSH Public Key to All Nodes

**On pg-master (self):**
```bash
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

**On pg-worker1:**
```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
vim ~/.ssh/authorized_keys
```

Paste the public key copied from pg-master. Save and exit.

```bash
chmod 600 ~/.ssh/authorized_keys
```

**On pg-worker2:**
```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
vim ~/.ssh/authorized_keys
```

Paste the public key. Save and exit.

```bash
chmod 600 ~/.ssh/authorized_keys
```

---

## Step 5 - Verify SSH Connectivity from pg-master

**On pg-master:**
```bash
ssh -o StrictHostKeyChecking=no root@pg-worker1 "hostname && echo OK"
ssh -o StrictHostKeyChecking=no root@pg-worker2 "hostname && echo OK"
ssh -o StrictHostKeyChecking=no root@10.40.71.50 "hostname && echo OK"
```

Expected output for each command:
```
pg-worker1
OK
```
```
pg-worker2
OK
```
```
pg-master
OK
```

---

## Step 6 - Install Ansible on pg-master

Ubuntu 24.04 ships with Ansible 2.16.x by default. Autobase requires a minimum of 2.17.0. Install from the official Ansible.

**On pg-master:**
```bash
apt update && apt install -y software-properties-common git python3-pip
add-apt-repository --yes --update ppa:ansible/ansible
apt install -y ansible
ansible --version
```

The output must show `ansible [core 2.17.x]` or higher.

**Packages installed:**
- `software-properties-common` - Provides `add-apt-repository` utility
- `git` - Required to clone the Autobase repository
- `python3-pip` - Python package manager, used by Ansible dependencies
- `ansible` - Automation engine that will orchestrate the cluster deployment

---

## Step 7 - Clone the Autobase Repository

**On pg-master:**
```bash
cd /opt
git clone --depth=1 https://github.com/vitabaks/postgresql_cluster.git
cd /opt/postgresql_cluster/automation
```

`--depth=1` fetches only the latest commit, significantly reducing download size and avoiding timeout errors on slow connections.

---

## Step 8 - Install Ansible Galaxy Collections

**On pg-master:**
```bash
cd /opt/postgresql_cluster/automation
ansible-galaxy collection build --output-path /tmp/autobase-collection .
ansible-galaxy collection install /tmp/autobase-collection/vitabaks-autobase-*.tar.gz --force
ansible-galaxy install -r requirements.yml
```

**What this does:**
- Builds the local Autobase collection into a distributable archive
- Installs the collection and all its dependencies (community.postgresql, ansible.posix, etc.)
- `requirements.yml` installs any additional roles declared by the project

---

## Step 9 - Create the Inventory File

**On pg-master:**
```bash
mkdir -p /opt/postgresql_cluster/automation/inventory
vim /opt/postgresql_cluster/automation/inventory/hosts.ini
```

Enter the following content:

```ini
[master]
10.40.71.50 ansible_user=root

[replica]
10.40.71.51 ansible_user=root
10.40.71.52 ansible_user=root

[postgres_cluster:children]
master
replica

[etcd_cluster:children]
postgres_cluster

[balancers:children]
postgres_cluster
```

Save and exit.

**Important:** The `[balancers]` group is required for HAProxy and Keepalived to be deployed. If this group is missing, the load balancing components will be silently skipped during deployment.

---

## Step 10 - Create the Variables File

**On pg-master:**
```bash
mkdir -p /opt/postgresql_cluster/automation/vars
vim /opt/postgresql_cluster/automation/vars/main.yml
```

Enter the following content:

```yaml
cluster_name: "postgres-cluster"
postgresql_version: 17
dcs_type: "etcd"

patroni_superuser_username: "postgres"
patroni_superuser_password: "Kx9mR2vNpQ7wLsY4"
patroni_replication_username: "replicator"
patroni_replication_password: "Tz5jH8cBnW3qXuA6"

with_haproxy_load_balancing: true
cluster_vip: "10.40.70.200"
```

Save and exit.


**Note on passwords:** Passwords are stored in plain text here for simplicity in a test environment. In production, use `ansible-vault encrypt vars/main.yml` to encrypt the file at rest and pass `--ask-vault-pass` at deploy time.

---

## Step 11 - Verify Ansible Connectivity

**On pg-master:**
```bash
ansible all -i /opt/postgresql_cluster/automation/inventory/hosts.ini -m ping
```

All three nodes must return `pong`:

```
10.40.71.50  | SUCCESS => { "ping": "pong" }
10.40.71.51  | SUCCESS => { "ping": "pong" }
10.40.71.52 | SUCCESS => { "ping": "pong" }
```

Do not proceed if any node fails.

---

## Step 12 - Deploy the Cluster

**On pg-master:**
```bash
cd /opt/postgresql_cluster/automation
ansible-playbook playbooks/deploy_pgcluster.yml \
  -i inventory/hosts.ini \
  --extra-vars "@vars/main.yml"
```

This playbook installs and configures the following on all nodes:
- PostgreSQL 17
- Patroni
- etcd (with TLS)
- PgBouncer
- HAProxy
- Keepalived
- confd
- Netdata (monitoring agent)
- chrony

**Expected final output:**

```
PLAY RECAP
10.40.71.50  : ok=177  changed=84  unreachable=0  failed=0
10.40.71.51  : ok=126  changed=67  unreachable=0  failed=0
10.40.71.52 : ok=126  changed=68  unreachable=0  failed=0
```

`failed=0` on all nodes confirms a successful deployment.

---

## Step 13 - Verify the Cluster

### 13.1 - Service Status

**On pg-master:**
```bash
systemctl status patroni etcd pgbouncer haproxy keepalived --no-pager
```

All five services must show `active (running)`.

### 13.2 - Virtual IP Assignment

VIP is managed by Keepalived and floats between nodes based on HAProxy health scores. It is not guaranteed to be on pg-master — it may be assigned to any of the three nodes. Check all nodes to find where the VIP is currently assigned.

**On pg-master:**
```bash
ip a | grep 10.40.70.200
```

**On pg-worker1:**
```bash
ssh root@pg-worker1 "ip a | grep 10.40.70.200"
```

**On pg-worker2:**
```bash
ssh root@pg-worker2 "ip a | grep 10.40.70.200"
```

Expected output on whichever node holds the VIP:
```
inet 10.40.70.200/32 scope global enp0s3
```

other two nodes will return no output for this command. This is expected behavior — only one node holds the VIP at any given time.

### 13.3 - Patroni Cluster State

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Expected output:
```
+ Cluster: postgres-cluster ---+----+-------------+-----+
| Member     | Host           | Role    | State     | Lag |
+------------+----------------+---------+-----------+-----+
| pg-master  | 10.40.71.50  | Leader  | running   |     |
| pg-worker1 | 10.40.71.51  | Replica | streaming |   0 |
| pg-worker2 | 10.40.71.52 | Replica | streaming |   0 |
+------------+----------------+---------+-----------+-----+
```

`streaming` state and `Lag: 0` confirm that replication is healthy and real-time.

### 13.4 - Primary Connection Test (Write)

```bash
psql -h 10.40.70.200 -p 5000 -U postgres -W -c "SELECT pg_is_in_recovery();"
```

Expected output: `f` (false) - this node is the primary and accepts writes.

### 13.5 - Replica Connection Test (Read)

```bash
psql -h 10.40.70.200 -p 5001 -U postgres -W -c "SELECT pg_is_in_recovery();"
```

Expected output: `t` (true) - this node is a replica in recovery mode.

### 13.6 - Replication Status

```bash
psql -h 10.40.70.200 -p 5000 -U postgres -W -c "SELECT client_addr, state, sent_lsn, write_lsn, replay_lsn FROM pg_stat_replication;"
```

Two rows must be returned, one per replica.


---
