# Zabbix Monitoring Platform -- End-to-End Deployment & Design Guide

## HA Deployment Across Two On-Premises Sites

**Target Environments:** Windows, Linux, Kubernetes, Microsoft 365
**Infrastructure Platform:** VMware vSphere (separate clusters per site)
**Load Balancing:** KEMP LoadMaster VLM (HA pair per site)

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Sizing & Capacity Planning](#2-sizing--capacity-planning)
3. [DNS & Naming Strategy](#3-dns--naming-strategy)
4. [Installation -- OS & Packages](#4-installation----os--packages)
5. [HA Infrastructure -- etcd Cluster](#5-ha-infrastructure----etcd-cluster)
6. [HA Infrastructure -- Database Layer](#6-ha-infrastructure----database-layer)
7. [HA Infrastructure -- Zabbix Server Cluster](#7-ha-infrastructure----zabbix-server-cluster)
8. [HA Infrastructure -- Frontend & Load Balancers (KEMP LoadMaster)](#8-ha-infrastructure----frontend--load-balancers-kemp-loadmaster)
9. [Proxy Architecture -- Distributed Monitoring](#9-proxy-architecture----distributed-monitoring)
10. [Windows Monitoring](#10-windows-monitoring)
11. [Linux Monitoring](#11-linux-monitoring)
12. [Kubernetes Monitoring](#12-kubernetes-monitoring)
13. [Microsoft 365 Monitoring](#13-microsoft-365-monitoring)
14. [Agent Deployment & Automation](#14-agent-deployment--automation)
15. [Alerting & Notification](#15-alerting--notification)
16. [Encryption & Security](#16-encryption--security)
17. [Network & Firewall Requirements](#17-network--firewall-requirements)
18. [Backup & Disaster Recovery](#18-backup--disaster-recovery)
19. [Operational Runbook](#19-operational-runbook)
20. [Known Limitations & Gotchas](#20-known-limitations--gotchas)

---

## 1. Architecture Overview

### Reference Architecture (Two-Site HA)

```
═══════════════════════════════════════════════════════════════════════════
                   SITE A (Primary) -- VMware Cluster A
═══════════════════════════════════════════════════════════════════════════

  [KEMP VLM-A1 (Active)]  [KEMP VLM-A2 (Passive)]   ← HA pair (CARP)
       │ Shared VIP-A (Frontend + DB)                             │
       ├──────────────────────────────────────┐                   │
  [Frontend-A]   [Zabbix Server Node-01 (Active)]                 │
                      │                                           │
               [PostgreSQL Primary + TimescaleDB]                 │
               [Patroni + etcd Node-1]                            │
                                                                  │
  [Proxy Group: SiteA-Proxies]                                    │
  [Proxy-A1]  [Proxy-A2]                                          │
       │                                                          │
  [Windows Agents]  [Linux Agents]  [K8s Cluster(s)]              │
                                                                  │
─ ─ ─ ─ ─ ─ ─ ─ ─ ─ WAN / Cross-Site Link ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
                                                                  │
═══════════════════════════════════════════════════════════════════════════
                   SITE B (Secondary) -- VMware Cluster B
═══════════════════════════════════════════════════════════════════════════

  [KEMP VLM-B1 (Active)]  [KEMP VLM-B2 (Passive)]   ← HA pair (CARP)
       │ Shared VIP-B (Frontend + DB)                             │
       ├──────────────────────────────────────┐                   │
  [Frontend-B]   [Zabbix Server Node-02 (Standby)]                │
                      │                                           │
               [PostgreSQL Standby (sync replica)]                │
               [Patroni + etcd Node-2]                            │
                                                                  │
  [Proxy Group: SiteB-Proxies]                                    │
  [Proxy-B1]  [Proxy-B2]                                          │
       │                                                          │
  [Windows Agents]  [Linux Agents]  [K8s Cluster(s)]              │
                                                                  │
═══════════════════════════════════════════════════════════════════════════

  [etcd Node-3 (Arbitrator)] ── at either site or a third location
  [M365 Monitoring Host]     ── monitored via proxy group (survives site failure)
```

### Component Summary

| Component | Count | Deployment |
|-----------|-------|------------|
| VMware Clusters | 2 | 1 per site (separate vSphere clusters) |
| KEMP LoadMaster VLM | 4 | HA pair per site (active/passive via CARP) |
| Zabbix Server (HA cluster) | 2 | 1 per site (active/standby) |
| PostgreSQL + TimescaleDB | 2 | 1 per site (primary/standby via Patroni) |
| etcd (consensus) | 3 | 2 at primary + 1 at secondary (or distributed) |
| Zabbix Frontend | 2 | 1 per site behind KEMP VIP |
| Zabbix Proxies | 4+ | 2+ per site in proxy groups |
| K8s In-Cluster Proxy | 1 per cluster | Helm chart deployment |
| Zabbix Agents | All hosts | DaemonSet in K8s, package on VMs |

### Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Infrastructure | **VMware vSphere** (separate cluster per site) | Existing platform, proven enterprise hypervisor |
| Load Balancer | **KEMP LoadMaster VLM** (HA pair per site) | Existing investment, L4/L7 support, Patroni health check integration |
| Zabbix Version | **7.0 LTS** or **7.2** | Native HA cluster, proxy groups, async pollers |
| Database | **PostgreSQL 16 + TimescaleDB 2.x** | Best performance for time-series data, 20-25% improvement, up to 90% compression |
| Database HA | **Patroni + etcd** | Battle-tested automatic failover, strong consistency |
| Agent Version | **Zabbix Agent 2** | Plugin architecture, native K8s/Docker/DB monitoring |
| Agent Mode | **Active** (agents push to server/proxy) | Scales better, works through NAT/firewalls, enables auto-registration |
| Encryption | **PSK** (default), upgrade path to certificates | Simple deployment, unique per host |

---

## 2. Sizing & Capacity Planning

### Understanding NVPS (New Values Per Second)

NVPS is the key capacity metric. Calculate it as:

```
NVPS = Total_Items / Average_Polling_Interval_Seconds

Example: 5,000 hosts x 200 items/host x (1/60s avg) = ~16,667 NVPS
```

### VM Sizing by Tier

#### Zabbix Server VMs

| Scale | Hosts | Est. NVPS | vCPU | RAM | Notes |
|-------|-------|-----------|------|-----|-------|
| Small | < 500 | < 5K | 4 | 16 GB | DB can be co-located |
| Medium | 500-2,000 | 5K-20K | 8 | 32 GB | Separate DB recommended |
| Large | 2,000-10,000 | 20K-100K | 16 | 64 GB | Separate DB required |
| Very Large | 10,000+ | 100K+ | 32 | 96 GB | Dedicated DB cluster |

#### Database Server VMs

| Scale | vCPU | RAM | Storage | Type |
|-------|------|-----|---------|------|
| Small | 4 | 16 GB | 200 GB SSD | Co-located OK |
| Medium | 8 | 32 GB | 500 GB SSD | Separate server |
| Large | 16 | 64-128 GB | 1-2 TB NVMe | Separate server |
| Very Large | 32+ | 128-256 GB | Multi-TB NVMe | Dedicated cluster |

> **Storage I/O is the primary bottleneck.** Use NVMe SSDs for any deployment exceeding 10K NVPS. Spinning disks are not viable for production.

#### Proxy VMs

| Proxy Scale | vCPU | RAM | Local DB |
|-------------|------|-----|----------|
| Light (< 1K NVPS) | 2 | 4 GB | SQLite |
| Medium (1K-5K NVPS) | 4 | 8 GB | PostgreSQL |
| Heavy (5K+ NVPS) | 8 | 16 GB | PostgreSQL |

#### Frontend / KEMP / etcd VMs

| Component | vCPU | RAM | Storage | Notes |
|-----------|------|-----|---------|-------|
| Frontend (Nginx + PHP-FPM) | 2 | 4 GB | 20 GB | |
| KEMP LoadMaster VLM | 2 | 4 GB | 32 GB (thick) | OVF deploy, VMXNET3 NICs, static MACs |
| etcd Node | 2 | 4 GB | 20 GB SSD | |

### Database Sizing Formulas

**History storage:**
```
Size = NVPS * 86,400 * Retention_Days * Bytes_Per_Value
  Numeric value  ~ 50 bytes
  Text/log value ~ 500 bytes

Example: 16K NVPS * 86,400 * 14 days * 50 bytes = ~967 GB
```

**Trend storage:**
```
Size = Items * 24 * 365 * Years * 90 bytes

Example: 1,000,000 items * 24 * 365 * 5 years * 90 bytes = ~3.9 TB
With TimescaleDB compression (~90% savings): ~400 GB
```

**Event storage:** ~250 bytes per event

### Recommended Retention Policy

| Data Type | History | Trends |
|-----------|---------|--------|
| Numeric metrics | 14 days | 365 days |
| Character/text | 7 days | N/A |
| Log data | 7 days | N/A |
| Events | 365 days | N/A |

### Total VM Inventory Per Site

This is the concrete bill of materials. Each site deploys the following VMs (assumes a **Medium** tier deployment of 500-2,000 hosts, 5K-20K NVPS):

| VM | vCPU | RAM | Storage | Qty/Site | Purpose |
|----|------|-----|---------|----------|---------|
| Zabbix Server (HA node) | 8 | 32 GB | 100 GB | 1 | Active or standby server |
| PostgreSQL + TimescaleDB | 8 | 32 GB | 500 GB SSD | 1 | Primary or replica database |
| etcd | 2 | 4 GB | 20 GB SSD | 1-2 | Consensus cluster member |
| Zabbix Frontend (Nginx + PHP-FPM) | 2 | 4 GB | 20 GB | 1 | Web UI |
| KEMP LoadMaster VLM | 2 | 4 GB | 32 GB | 2 | HA pair (active/passive) |
| Zabbix Proxy | 4 | 8 GB | 50 GB | 2 | Proxy group members |
| **Site Total** | **32-34** | **96-100 GB** | **~804 GB** | **8-9** | |

**Combined totals (both sites + arbitrator):**

| Resource | Total |
|----------|-------|
| VMs | 17-19 (8-9 per site + 1 etcd arbitrator) |
| vCPU | 66-70 |
| RAM | 196-204 GB |
| Storage | ~1.6 TB (SSD/NVMe for database VMs) |

> **Note:** Scale these values using the sizing tiers in the tables above. For Large (2K-10K hosts), double the server and database VM specs. The KEMP, etcd, and frontend VMs remain the same across tiers.

---

## 3. DNS & Naming Strategy

### DNS Records

| Record | Type | Value | Purpose |
|--------|------|-------|---------|
| `zabbix.example.com` | A | 10.1.1.100 (Site A KEMP Frontend VIP) | Primary user-facing URL |
| `zabbix-b.example.com` | A | 10.2.1.100 (Site B KEMP Frontend VIP) | Secondary / DR access URL |
| `db-vip-a.example.com` | A | 10.1.1.200 (Site A KEMP DB VIP) | Database VIP used by Site A components |
| `db-vip-b.example.com` | A | 10.2.1.200 (Site B KEMP DB VIP) | Database VIP used by Site B components |
| `zabbix-node-siteA` | A | 10.1.1.10 | Zabbix Server HA Node 1 |
| `zabbix-node-siteB` | A | 10.2.1.10 | Zabbix Server HA Node 2 |
| `proxy-siteA-01` | A | 10.1.1.40 | Zabbix Proxy at Site A |
| `proxy-siteA-02` | A | 10.1.1.41 | Zabbix Proxy at Site A |
| `proxy-siteB-01` | A | 10.2.1.40 | Zabbix Proxy at Site B |
| `proxy-siteB-02` | A | 10.2.1.41 | Zabbix Proxy at Site B |

### Cross-Site Frontend Failover Strategy

The Zabbix frontend is stateless -- both sites can serve the UI simultaneously because they both read from the same database (via the active PostgreSQL primary). Two approaches for cross-site access:

**Option A: Manual DNS swing (simplest)**
- Normal: `zabbix.example.com` -> Site A KEMP VIP
- Site A failure: Update DNS to point `zabbix.example.com` -> Site B KEMP VIP
- Pros: Simple, no additional infrastructure
- Cons: Manual intervention, DNS TTL propagation delay (set TTL to 60-300s)

**Option B: Dedicated URL per site (recommended)**
- Users bookmark `zabbix.example.com` (Site A) and know `zabbix-b.example.com` (Site B) exists
- If Site A is down, users access Site B URL directly
- Pros: No DNS changes needed, instant failover
- Cons: Users must know both URLs

**Option C: KEMP Global Server Load Balancing (GSLB)**
- If your KEMP license includes GEO functionality, configure GSLB to automatically route `zabbix.example.com` to the healthy site
- Pros: Fully automatic failover
- Cons: Requires KEMP GEO license, additional configuration

### Agent Config Name Resolution

Agents use `ServerActive` to reach proxies. Use DNS names (not IPs) for easier maintenance:

```ini
# Agents resolve proxy names via DNS -- updating IPs requires only DNS change
ServerActive=proxy-siteA-01;proxy-siteA-02
Server=proxy-siteA-01,proxy-siteA-02
```

> **Important:** Set DNS TTL for proxy and server records to **60 seconds** or lower to enable fast failover.

---

## 4. Installation -- OS & Packages

### Base OS

**Rocky Linux 9** (or RHEL 9) for all Zabbix VMs. Rocky Linux is binary-compatible with RHEL, free, and well-supported by all components.

### Package Installation -- Step by Step

#### 4a. PostgreSQL 16 + TimescaleDB (Database VMs)

```bash
# PostgreSQL 16 from PGDG repo
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf -qy module disable postgresql
dnf install -y postgresql16-server postgresql16

# TimescaleDB from official repo
tee /etc/yum.repos.d/timescale_timescaledb.repo <<'EOL'
[timescale_timescaledb]
name=timescale_timescaledb
baseurl=https://packagecloud.io/timescale/timescaledb/el/$(rpm -E %{rhel})/$basearch
repo_gpgcheck=1
gpgcheck=0
enabled=1
gpgkey=https://packagecloud.io/timescale/timescaledb/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
EOL

dnf install -y timescaledb-2-postgresql-16 timescaledb-2-loader-postgresql-16
```

#### 4b. Zabbix Repository (All Zabbix VMs)

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rocky/9/x86_64/zabbix-release-latest-7.0.el9.noarch.rpm
dnf clean all
```

> If EPEL is enabled, add `excludepkgs=zabbix*` to `/etc/yum.repos.d/epel.repo` to prevent conflicts.

#### 4c. Zabbix Server (Server VMs)

```bash
dnf install -y zabbix-server-pgsql zabbix-sql-scripts zabbix-selinux-policy zabbix-agent2
```

#### 4d. Zabbix Frontend (Frontend VMs)

```bash
dnf install -y zabbix-web-pgsql zabbix-nginx-conf zabbix-selinux-policy
```

#### 4e. Zabbix Proxy (Proxy VMs)

```bash
# SQLite backend (< 1K NVPS per proxy)
dnf install -y zabbix-proxy-sqlite3 zabbix-selinux-policy zabbix-agent2

# OR PostgreSQL backend (> 1K NVPS per proxy)
dnf install -y zabbix-proxy-pgsql zabbix-sql-scripts zabbix-selinux-policy zabbix-agent2
```

#### 4f. Zabbix Agent 2 (All Monitored Linux Hosts)

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rocky/9/x86_64/zabbix-release-latest-7.0.el9.noarch.rpm
dnf clean all
dnf install -y zabbix-agent2
```

Optional Agent 2 plugins: `zabbix-agent2-plugin-postgresql`, `zabbix-agent2-plugin-mongodb`, `zabbix-agent2-plugin-mssql`.

### Database Creation & Schema Import

This sequence must run on the **initial primary** PostgreSQL node, **before** Patroni takes over.

```bash
# 1. Initialize PostgreSQL (standalone, before Patroni)
/usr/pgsql-16/bin/postgresql-16-setup initdb
systemctl start postgresql-16

# 2. Create Zabbix user and database
sudo -u postgres createuser --pwprompt zabbix
sudo -u postgres createdb -O zabbix -E Unicode -T template0 zabbix

# 3. Import Zabbix schema
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix

# 4. Configure TimescaleDB
timescaledb-tune --pg-config=/usr/pgsql-16/bin/pg_config --quiet --yes
systemctl restart postgresql-16

# 5. Enable TimescaleDB extension
echo "CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;" | sudo -u postgres psql zabbix

# 6. Import Zabbix TimescaleDB migration
cat /usr/share/zabbix-sql-scripts/postgresql/timescaledb/schema.sql | sudo -u zabbix psql zabbix

# 7. Stop PostgreSQL -- Patroni will manage it from here
systemctl stop postgresql-16
systemctl disable postgresql-16
```

> **Order of operations:** PostgreSQL schema import must happen **before** Patroni bootstrap. Patroni will take over managing the PostgreSQL instance. After Patroni is configured, you never start PostgreSQL directly again -- Patroni controls it.

### SELinux Configuration

```bash
# The zabbix-selinux-policy package handles most cases. If needed:
setsebool -P httpd_can_connect_zabbix on
setsebool -P httpd_can_network_connect_db on
setsebool -P zabbix_can_network on
```

### Frontend PHP-FPM Configuration

```bash
# Set timezone in /etc/php-fpm.d/zabbix.conf
sed -i 's/; php_value\[date.timezone\].*/php_value[date.timezone] = UTC/' /etc/php-fpm.d/zabbix.conf

# Change user/group from apache to nginx
sed -i 's/^user = apache/user = nginx/' /etc/php-fpm.d/zabbix.conf
sed -i 's/^group = apache/group = nginx/' /etc/php-fpm.d/zabbix.conf

# Configure Nginx listener in /etc/nginx/conf.d/zabbix.conf
# Uncomment and set: listen 8080; server_name zabbix.example.com;
```

---

## 5. HA Infrastructure -- etcd Cluster

### Cluster Topology

| Node | IP Address | Site | Purpose |
|------|------------|------|---------|
| etcd-1 | 10.1.1.30 | Site A | Full cluster member |
| etcd-2 | 10.2.1.30 | Site B | Full cluster member |
| etcd-3 | 10.1.1.31 | Site A (or Site B / third location) | Quorum tiebreaker |

> **Critical for DR:** If etcd-3 is at Site A and Site A is lost, Site B cannot form quorum (1 of 3) and Patroni will **not** auto-promote. Place etcd-3 at Site B or a third location for automatic failover when Site A fails.

### Installation

```bash
# Option A: PGDG Extras (if you already have the PGDG repo)
dnf install -y 'dnf-command(config-manager)'
dnf config-manager --enable pgdg-rhel9-extras
dnf install -y etcd

# Option B: Binary install (recommended for latest version)
ETCD_VER=v3.5.21
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
  -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/
sudo install -m 0755 /tmp/etcd-${ETCD_VER}-linux-amd64/etcd /usr/local/bin/etcd
sudo install -m 0755 /tmp/etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/etcdctl

useradd -r -s /sbin/nologin etcd
mkdir -p /var/lib/etcd /etc/etcd
chown etcd:etcd /var/lib/etcd
chmod 700 /var/lib/etcd
```

### Configuration -- etcd-1 (`/etc/etcd/etcd.conf.yml`)

```yaml
name: 'etcd-1'
data-dir: /var/lib/etcd

listen-peer-urls: 'https://10.1.1.30:2380'
initial-advertise-peer-urls: 'https://10.1.1.30:2380'
listen-client-urls: 'https://10.1.1.30:2379,https://127.0.0.1:2379'
advertise-client-urls: 'https://10.1.1.30:2379'

initial-cluster-token: 'patroni-etcd-cluster'
initial-cluster-state: 'new'
initial-cluster: 'etcd-1=https://10.1.1.30:2380,etcd-2=https://10.2.1.30:2380,etcd-3=https://10.1.1.31:2380'

# Tuning -- adjust heartbeat to ~1.5x inter-site RTT
heartbeat-interval: 250
election-timeout: 2500
snapshot-count: 10000
quota-backend-bytes: 2147483648

auto-compaction-mode: periodic
auto-compaction-retention: '1h'

# TLS: Client
client-transport-security:
  cert-file: /etc/etcd/pki/server.crt
  key-file: /etc/etcd/pki/server.key
  trusted-ca-file: /etc/etcd/pki/ca.crt
  client-cert-auth: false

# TLS: Peer
peer-transport-security:
  cert-file: /etc/etcd/pki/peer.crt
  key-file: /etc/etcd/pki/peer.key
  trusted-ca-file: /etc/etcd/pki/ca.crt
  client-cert-auth: true
```

etcd-2 and etcd-3 use identical config except `name`, `listen-*-urls`, and `advertise-*-urls` with their respective IPs.

### TLS Certificate Generation

```bash
mkdir -p ~/etcd-pki && cd ~/etcd-pki

# Generate CA
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -subj "/CN=etcd-ca/O=MyOrg" -out ca.crt

# Generate server + peer certs per node (example for etcd-1)
for ROLE in server peer; do
  openssl genrsa -out etcd-1-${ROLE}.key 2048
  openssl req -new -key etcd-1-${ROLE}.key -subj "/CN=etcd-1" \
    -addext "subjectAltName=IP:10.1.1.30,IP:127.0.0.1,DNS:etcd-1" \
    -out etcd-1-${ROLE}.csr
  openssl x509 -req -in etcd-1-${ROLE}.csr -CA ca.crt -CAkey ca.key \
    -CAcreateserial -out etcd-1-${ROLE}.crt -days 1825 -sha256 \
    -extfile <(printf "subjectAltName=IP:10.1.1.30,IP:127.0.0.1,DNS:etcd-1")
done
# Repeat for etcd-2 (10.2.1.30) and etcd-3 (10.1.1.31)
```

Deploy to each node at `/etc/etcd/pki/` with `ca.crt`, `server.crt`, `server.key`, `peer.crt`, `peer.key`.

### Systemd Unit (`/etc/systemd/system/etcd.service`)

```ini
[Unit]
Description=etcd key-value store
After=network-online.target
Wants=network-online.target

[Service]
User=etcd
Group=etcd
Type=notify
Environment="ETCD_CONFIG_FILE=/etc/etcd/etcd.conf.yml"
ExecStart=/usr/local/bin/etcd
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```

### Bootstrap Procedure

Start all 3 nodes within 30 seconds of each other:

```bash
# On all 3 nodes
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
```

### Verification

```bash
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS="https://10.1.1.30:2379,https://10.2.1.30:2379,https://10.1.1.31:2379"
export ETCDCTL_CACERT="/etc/etcd/pki/ca.crt"

etcdctl member list -w table          # Show all members
etcdctl endpoint health -w table      # Health check
etcdctl endpoint status -w table      # Show leader, DB size
```

### etcd Authentication for Patroni

```bash
etcdctl user add root              # Set password
etcdctl user add patroni           # Set password
etcdctl role add patroni-role
etcdctl role grant-permission patroni-role readwrite --prefix=true /service/
etcdctl user grant-role patroni patroni-role
etcdctl user grant-role root root
etcdctl auth enable
```

### Patroni etcd3 Connection Config

Add this to each `patroni.yml`:

```yaml
etcd3:
  hosts: 10.1.1.30:2379,10.2.1.30:2379,10.1.1.31:2379
  protocol: https
  cacert: /etc/patroni/pki/ca.crt
  username: patroni
  password: '<secret>'
```

---

## 6. HA Infrastructure -- Database Layer

The Zabbix server HA cluster coordinates through the database. The database is the single point of failure unless independently made highly available.

### Recommended: PostgreSQL + Patroni + etcd

**Architecture:**

| Site | Components |
|------|------------|
| Site A | PostgreSQL Primary + Patroni + etcd-1 + KEMP VLM HA pair (DB VIP) |
| Site B | PostgreSQL Standby + Patroni + etcd-2 + KEMP VLM HA pair (DB VIP) |
| Arbitrator | etcd-3 (lightweight VM at either site or third location) |

**How it works:**
- Patroni manages PostgreSQL nodes and orchestrates failover
- etcd provides distributed consensus (requires 3 nodes for quorum)
- KEMP LoadMaster queries Patroni's REST API (port 8008) to route writes to the current primary
- KEMP HA pair provides a shared VIP via CARP protocol with automatic failover

**Patroni configuration example (`patroni.yml`):**
```yaml
scope: zabbix-cluster
name: pg-siteA

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.1.1.20:8008

etcd3:
  hosts: 10.1.1.30:2379,10.2.1.30:2379,10.1.1.31:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    synchronous_mode: true
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 500
        shared_buffers: 8GB
        work_mem: 64MB
        effective_cache_size: 24GB
        wal_level: replica
        max_wal_senders: 5
        max_replication_slots: 5
        shared_preload_libraries: timescaledb

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.1.1.20:5432
  data_dir: /var/lib/postgresql/16/main
  authentication:
    replication:
      username: replicator
      password: <secret>
    superuser:
      username: postgres
      password: <secret>
```

### Synchronous vs. Asynchronous Replication

The Patroni config uses `synchronous_mode: true`, meaning every write waits for cross-site replica confirmation. This guarantees zero data loss on failover but caps write throughput based on inter-site latency.

**NVPS ceiling by inter-site round-trip time:**

| Inter-site RTT | Sync Mode NVPS Ceiling | Recommendation |
|----------------|------------------------|----------------|
| < 2ms (same campus) | 50K+ | **Synchronous** -- negligible penalty |
| 2-5ms (metro area) | 15-25K | **Synchronous** -- acceptable for most deployments |
| 5-10ms | 8-15K | **Evaluate** -- sync if NVPS target is below ceiling |
| 10-20ms | 4-8K | **Asynchronous recommended** -- sync caps small deployments |
| 20ms+ | < 4K | **Asynchronous required** -- sync is not viable |

**Measure your actual RTT before choosing:**
```bash
ping -c 50 <site-b-pg-ip>
# Use the average RTT value against the table above
```

**To switch to asynchronous replication:**
```bash
patronictl -c /etc/patroni/patroni.yml edit-config -s synchronous_mode=false
```

**Tradeoff:** Async replication risks losing the most recent transactions (typically < 1 second of data) if the primary fails before the replica catches up. For Zabbix monitoring data, this usually means losing a few seconds of metric samples -- generally acceptable. For environments requiring zero data loss, keep synchronous mode and ensure inter-site latency supports your NVPS target.

> **Recommendation:** Measure RTT, calculate your expected NVPS (Section 2), and compare against the ceiling table. If your NVPS target is within 50% of the ceiling, use **asynchronous** mode to leave headroom.

### TimescaleDB Configuration

Install TimescaleDB on top of PostgreSQL for history table partitioning and compression:

```sql
-- After Zabbix DB schema creation
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;

-- Run Zabbix TimescaleDB migration
-- (via zabbix_server --with-timescaledb)
```

**Key benefits:**
- Automatic time-based partitioning of history tables
- Compression saves up to **90% disk space**
- Partition-drop housekeeping is orders of magnitude faster than row DELETE
- 20-25% query performance improvement

**Tuning:**
```sql
-- For better trend update performance on high-item-count deployments
SELECT set_chunk_time_interval('trends', INTERVAL '7 days');
SELECT set_chunk_time_interval('trends_uint', INTERVAL '7 days');
```

### Connection Pooling with PgBouncer (Recommended for Medium+)

For deployments exceeding 1,000 hosts, PgBouncer between Zabbix and PostgreSQL reduces connection overhead and improves failover behavior. Deploy PgBouncer **on each KEMP-fronted site**, co-located with the frontend or as a separate lightweight VM.

**Why PgBouncer matters:**
- Zabbix server opens many persistent connections (`max_connections: 500` fills fast with server + 2 frontends + proxies)
- During Patroni failover, PgBouncer transparently reconnects to the new primary -- Zabbix sees a brief pause, not connection errors
- Connection multiplexing reduces PostgreSQL memory usage per connection

**PgBouncer configuration (`/etc/pgbouncer/pgbouncer.ini`):**
```ini
[databases]
zabbix = host=127.0.0.1 port=5432 dbname=zabbix

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction               # Required for Zabbix (uses prepared statements in session mode)
max_client_conn = 500
default_pool_size = 50
reserve_pool_size = 10
reserve_pool_timeout = 3
server_reset_query = DISCARD ALL
log_connections = 0
log_disconnections = 0
```

> **Important:** Zabbix requires `pool_mode = transaction` (not `session`). Session mode would defeat the purpose of pooling. Zabbix 7.0+ is compatible with transaction-mode PgBouncer.

**If using PgBouncer**, update the KEMP DB Virtual Service to route to PgBouncer port (6432) instead of PostgreSQL directly (5432), or run PgBouncer locally on each Zabbix server VM and point `DBHost=127.0.0.1` / `DBPort=6432`.

### Alternative: MariaDB + Galera Cluster

For environments preferring MySQL/MariaDB:

| Site | Components |
|------|------------|
| Site A | MariaDB Node 1 (active/active) |
| Site B | MariaDB Node 2 (active/active) |
| Arbitrator | Galera garbd at a third location |

**Considerations:**
- Galera provides synchronous multi-primary replication
- Cross-site latency directly impacts write throughput
- Two-node Galera without an arbitrator is prone to split-brain
- No TimescaleDB equivalent (use MySQL native partitioning for history tables)

---

## 7. HA Infrastructure -- Zabbix Server Cluster

### How Native HA Works

Zabbix 7.x includes a built-in active/standby HA mechanism coordinated through the shared database. No external clustering software (Pacemaker, Corosync) is needed for the server layer.

- All nodes update a heartbeat every **5 seconds** in the `ha_node` table
- Only the **active node** performs data collection, trigger evaluation, and alerting
- Standby nodes run only the HA manager process (near-zero resource usage)

### Configuration

**Node 1 (Site A) -- `/etc/zabbix/zabbix_server.conf`:**
```ini
HANodeName=zabbix-node-siteA
NodeAddress=10.1.1.10:10051
DBHost=10.1.1.200                        # LOCAL Site A KEMP DB VIP
DBPort=5432
DBName=zabbix
DBUser=zabbix
DBPassword=<secret>
```

**Node 2 (Site B) -- `/etc/zabbix/zabbix_server.conf`:**
```ini
HANodeName=zabbix-node-siteB
NodeAddress=10.2.1.10:10051
DBHost=10.2.1.200                        # LOCAL Site B KEMP DB VIP
DBPort=5432
DBName=zabbix
DBUser=zabbix
DBPassword=<secret>
```

> **Critical:** Each server uses its **local** KEMP DB VIP, not a shared DNS name. Both KEMP instances have both PG nodes as Real Servers with Patroni `/primary` health checks. If PG fails over to the other site, the local KEMP routes cross-site to the new primary automatically. If the local site goes down entirely, the other site's server uses its own KEMP and is unaffected.

### Failover Behavior

| Scenario | Failover Time |
|----------|---------------|
| Graceful shutdown (active reports "stopped") | ~5 seconds |
| Ungraceful failure (crash/network loss) | failover_delay + 5 seconds |
| Default failover delay | 60 seconds (configurable: 10s - 15m) |
| **Recommended production setting** | **30 seconds** |

### Runtime Commands

```bash
zabbix_server -R ha_status                        # Check cluster status
zabbix_server -R ha_set_failover_delay=30s        # Set failover delay
zabbix_server -R ha_remove_node=<node_name>       # Remove dead node
```

### Performance Tuning (`zabbix_server.conf`)

These parameters must be tuned to match your NVPS target. Defaults are too low for anything beyond a small deployment.

| Parameter | Default | Small (<5K) | Medium (5-20K) | Large (20-100K) |
|-----------|---------|-------------|----------------|-----------------|
| `StartPollers` | 5 | 10 | 50 | 100 |
| `StartPollersUnreachable` | 1 | 3 | 10 | 25 |
| `StartPreprocessors` | 3 | 10 | 25 | 50 |
| `StartDBSyncers` | 4 | 4 | 8 | 16 |
| `StartHTTPPollers` | 1 | 5 | 10 | 20 |
| `CacheSize` | 32M | 64M | 256M | 512M |
| `ValueCacheSize` | 8M | 64M | 256M | 1G |
| `HistoryCacheSize` | 16M | 64M | 128M | 256M |
| `HistoryIndexCacheSize` | 4M | 16M | 32M | 64M |
| `TrendCacheSize` | 4M | 16M | 32M | 64M |
| `TrendFunctionCacheSize` | 4M | 16M | 32M | 64M |

**Example for a Medium deployment (add to `zabbix_server.conf`):**
```ini
# Process tuning
StartPollers=50
StartPollersUnreachable=10
StartPreprocessors=25
StartDBSyncers=8
StartHTTPPollers=10

# Cache tuning
CacheSize=256M
ValueCacheSize=256M
HistoryCacheSize=128M
HistoryIndexCacheSize=32M
TrendCacheSize=32M
TrendFunctionCacheSize=32M
```

> **Monitor these at runtime:** Zabbix has built-in internal items (`zabbix[wcache,values,*]`, `zabbix[rcache,buffer,pfree]`, `zabbix[process,poller,avg,busy]`) that show whether caches are full or processes are saturated. Link the **"Zabbix server health"** template to the server itself and alert when `pfree` drops below 20% or process busy exceeds 75%.

### Agent/Proxy Configuration for HA

All Zabbix server HA node addresses must be listed:

```ini
# Passive checks (comma-separated)
Server=zabbix-node-siteA,zabbix-node-siteB

# Active checks (semicolon-separated -- CRITICAL: commas cause data duplication)
ServerActive=zabbix-node-siteA;zabbix-node-siteB
```

---

## 8. HA Infrastructure -- Frontend & Load Balancers (KEMP LoadMaster)

### Frontend Configuration

The Zabbix frontend is stateless and reads the active HA node from the `ha_node` database table. **Do not hardcode** `ZBX_SERVER` or `ZBX_SERVER_PORT` in `zabbix.conf.php` when HA is enabled.

Deploy Nginx + PHP-FPM on each site with the frontend connected to the database VIP:

**Frontend at Site A:**
```php
// /etc/zabbix/web/zabbix.conf.php
$DB['TYPE']     = 'POSTGRESQL';
$DB['SERVER']   = '10.1.1.200';  // LOCAL Site A KEMP DB VIP
$DB['PORT']     = '5432';
$DB['DATABASE'] = 'zabbix';
$DB['USER']     = 'zabbix';
$DB['PASSWORD'] = '<secret>';
// Do NOT set $ZBX_SERVER -- frontend auto-detects active node via ha_node table
```

**Frontend at Site B:**
```php
// /etc/zabbix/web/zabbix.conf.php
$DB['TYPE']     = 'POSTGRESQL';
$DB['SERVER']   = '10.2.1.200';  // LOCAL Site B KEMP DB VIP
$DB['PORT']     = '5432';
$DB['DATABASE'] = 'zabbix';
$DB['USER']     = 'zabbix';
$DB['PASSWORD'] = '<secret>';
// Do NOT set $ZBX_SERVER -- frontend auto-detects active node via ha_node table
```

> **Why local VIPs?** If Site A goes down, Site B's frontend and server still reach the database through their local KEMP, which routes to the new PG primary at Site B. A shared `db-vip.example.com` pointing to Site A's KEMP would be a single point of failure.

### KEMP LoadMaster Deployment (Per Site)

Each site has a **KEMP VLM HA pair** (active/passive) deployed as OVF appliances on the local VMware cluster. The HA pair provides shared VIPs for both frontend and database traffic.

**KEMP HA pair per site:**

| Unit | Individual IP | Role |
|------|---------------|------|
| KEMP VLM-A1 | 10.1.1.50 | Active |
| KEMP VLM-A2 | 10.1.1.51 | Passive (standby) |
| Shared VIP (Frontend) | 10.1.1.100 | CARP failover between A1/A2 |
| Shared VIP (Database) | 10.1.1.200 | CARP failover between A1/A2 |

> **KEMP HA uses the CARP protocol** (not VRRP). Config syncs between units over TCP port 6973. Both units must be on the same broadcast domain with < 100ms latency.

### VMware Requirements for KEMP HA

The KEMP VLM uses MAC-based failover. The vSphere port group hosting the KEMP HA pair **must** have these security policies set:

| vSwitch Policy | Setting | Reason |
|----------------|---------|--------|
| **MAC Address Changes** | Accept | Allows CARP shared MAC activation on failover |
| **Forged Transmits** | Accept | Allows standby unit to send traffic with shared MAC after promotion |

Also required:
- **NIC type**: VMXNET3 (set during OVF deployment)
- **Static MAC addresses** on all KEMP VM NICs (license is MAC-bound)
- **Thick-provisioned** disk (recommended by KEMP)

### Virtual Service 1: Zabbix Frontend (HTTPS)

Create a Layer 7 Virtual Service for HTTPS access to the Zabbix web UI.

```
Virtual Service: Zabbix-Frontend-HTTPS
  Virtual Address:     10.1.1.100       (KEMP shared VIP)
  Port:                443
  Protocol:            tcp
  SSL Acceleration:    Enabled
  Scheduling Method:   Round Robin
  Persistence:         Active Cookie or Source IP
  Persistence Timeout: 30 minutes

Real Servers:
  frontend-siteA:  10.1.1.15:80   Weight: 1000
  frontend-siteB:  10.2.1.15:80   Weight: 1000   (or Weight: 0 for local-only)

Health Check:
  Method:              HTTP
  Check URL:           /
  Check Port:          80
  Expected Response:   200
```

**SSL certificate**: Import the certificate at **Certificates & Security > SSL Certificates**, then assign it to the Virtual Service.

> **Note:** If you want each site's KEMP to prefer its local frontend, set the remote-site Real Server weight to 0 (backup only) or use **Fixed Weighted (Chained Failover)** scheduling.

### Virtual Service 2: PostgreSQL Database (Layer 4 + Patroni Health Check)

Create a Layer 4 TCP Virtual Service that uses the Patroni REST API to find the current primary.

```
Virtual Service: PostgreSQL-Primary
  Virtual Address:     10.1.1.200       (Site A local KEMP DB VIP)
  Port:                5432
  Protocol:            tcp
  Force L7:            Disabled (pure Layer 4)
  Scheduling Method:   Fixed Weighted (Chained Failover)
  Persistence:         Source IP

Real Servers:
  pg-siteA:  10.1.1.20:5432   Weight: 1000
  pg-siteB:  10.2.1.20:5432   Weight: 1000

Health Check:
  Method:              HTTP
  Check Port:          8008           ← Patroni REST API port (different from service port)
  Check URL:           /primary       ← Returns HTTP 200 only on the current primary
  Expected Response:   200
```

**How it works:** Patroni's `/primary` endpoint returns HTTP 200 on the leader node and HTTP 503 on replicas. The KEMP health check polls port 8008 on each Real Server. When a Patroni failover occurs, the new primary starts returning 200, and KEMP automatically routes all database traffic to it -- no manual intervention needed.

### KEMP HA Configuration Steps

1. Deploy two KEMP VLM OVF appliances per site (same VLAN, same port group)
2. Assign individual management IPs to each unit
3. Access WUI at `https://<individual-ip>:8443` (default credentials: `bal` / `1fourall` -- change immediately)
4. On Unit 1: **System Configuration > High Availability > HA Parameters**
5. Set **Shared IP Address** for each interface
6. Enter partner unit's individual IP to pair
7. Configuration auto-syncs from active to passive via TCP 6973
8. Enable the **REST API** at **System Configuration > Miscellaneous Options > Remote Access** (for automation)

### KEMP REST API (Automation)

KEMP provides a REST API on the same port as the WUI (default 8443) for infrastructure-as-code workflows:

```bash
# List all Virtual Services
curl -k "https://bal:<password>@10.1.1.50:8443/access/listvs"

# Create a Virtual Service
curl -k "https://bal:<password>@10.1.1.50:8443/access/addvs?vs=10.1.1.100&port=443&prot=tcp&nickname=Zabbix-Frontend"

# Add a Real Server
curl -k "https://bal:<password>@10.1.1.50:8443/access/addrs?vs=10.1.1.100&port=443&prot=tcp&rs=10.1.1.15&rsport=80"
```

A **PowerShell module** (`Kemp.LoadBalancer.Powershell`) is also available on the PowerShell Gallery for Windows-based automation.

---

## 9. Proxy Architecture -- Distributed Monitoring

### Proxy Groups (Zabbix 7.0+)

Instead of assigning hosts to individual proxies, assign them to a **proxy group**. The Zabbix server dynamically distributes and rebalances hosts across proxies in the group.

**Configuration (via Zabbix Frontend):**

1. `Administration > Proxy groups > Create proxy group`
2. Set **Failover period**: 60s (time before failed proxy's hosts redistribute)
3. Set **Minimum proxies**: 2 (group enters "Degrading" state if only 1 remains)
4. Assign individual proxies to the group
5. Assign monitored hosts to the proxy group

### Recommended Layout

| Site | Proxy Group | Proxies | Purpose |
|------|-------------|---------|---------|
| Site A | `SiteA-Proxies` | Proxy-A1, Proxy-A2 | Monitor all Site A hosts |
| Site B | `SiteB-Proxies` | Proxy-B1, Proxy-B2 | Monitor all Site B hosts |
| K8s Cluster | In-cluster proxy (Helm) | 1 per cluster | K8s-specific monitoring |

### Proxy Configuration

**Active proxy (recommended) -- `/etc/zabbix/zabbix_proxy.conf`:**
```ini
ProxyMode=0                              # Active mode
Server=zabbix-node-siteA;zabbix-node-siteB
Hostname=proxy-siteA-01
ProxyLocalBuffer=1                       # Hours to keep data after sending
ProxyOfflineBuffer=24                    # Hours to buffer during WAN outage
ConfigFrequency=10                       # Seconds between config syncs
DataSenderFrequency=1                    # Seconds between data sends
CacheSize=128M
DBName=/var/lib/zabbix/proxy.db          # SQLite for < 1K NVPS
TLSConnect=psk
TLSPSKIdentity=PSK_PROXY_A1
TLSPSKFile=/etc/zabbix/proxy.psk
```

### Proxy vs. Direct Monitoring

| Factor | Direct (Server-to-Agent) | Via Proxy |
|--------|--------------------------|-----------|
| Server load | Higher | Lower (proxy offloads collection & preprocessing) |
| Firewall | Agent needs rule to/from server | Only proxy needs server connectivity |
| WAN dependency | Every poll traverses WAN | Only proxy sync traverses WAN (batched) |
| Data loss on WAN outage | Lost polls | Proxy buffers locally (no data loss) |
| Scalability | Limited by single server | Near-linear by adding proxies |

> **Recommendation:** Always use proxies at each site. Even for same-LAN hosts, proxies provide data buffering during maintenance and offload preprocessing from the server.

### Proxy Group Limitations

- SNMP traps are **NOT supported** for proxies in proxy groups (use standalone proxies for SNMP trap receivers)
- External scripts must be **identical** on all proxies in a group
- Only Zabbix 7.0+ agents support proxy group redirection
- Active agent redirection can loop if a proxy is unreachable (network partition)

### SNMP & Network Device Monitoring

Switches, firewalls, routers, storage arrays, and WAN links between sites should be monitored via SNMP. Zabbix includes hundreds of vendor templates.

**Key templates:**

| Vendor | Template | Monitors |
|--------|----------|----------|
| Cisco IOS | Cisco IOS by SNMP | Interfaces, CPU, memory, BGP/OSPF, environmental sensors |
| Cisco Catalyst/Nexus | Cisco [model] by SNMP | Interfaces, VLANs, stacking, PoE |
| Palo Alto | Palo Alto by SNMP | Sessions, throughput, threat events, HA state |
| HP/Aruba | HP Enterprise Switch by SNMP | Interfaces, CPU, memory, fan/PSU |
| Dell EMC | Dell [model] by SNMP | Interfaces, storage, environmental |
| Generic | Interfaces by SNMP | Interface discovery, traffic, errors, CRC |

**SNMP v3 (recommended):**
```
{$SNMPV3_SECURITYNAME}  = zabbix-monitor
{$SNMPV3_AUTHPROTOCOL}  = SHA256
{$SNMPV3_AUTHPASSPHRASE} = <secret>
{$SNMPV3_PRIVPROTOCOL}  = AES256
{$SNMPV3_PRIVPASSPHRASE} = <secret>
{$SNMPV3_CONTEXTNAME}   =
```

**SNMP trap receiver (standalone proxy):**

SNMP traps cannot use proxy groups. Deploy a **standalone proxy** (not in a group) at each site as the trap receiver:

| Site | Proxy | Purpose |
|------|-------|---------|
| Site A | `proxy-siteA-snmptrap` | SNMP trap receiver for Site A network devices |
| Site B | `proxy-siteB-snmptrap` | SNMP trap receiver for Site B network devices |

Configure `snmptrapd` on the standalone proxy and set `SNMPTrapperFile` in the proxy config. Network devices send traps to their local proxy IP.

**Firewall ports for SNMP:**

| Port | Protocol | Direction | Purpose |
|------|----------|-----------|---------|
| 161/UDP | SNMP | Proxy -> Device | SNMP polling |
| 162/UDP | SNMP | Device -> Proxy | SNMP traps (inbound to standalone proxy) |

---

## 10. Windows Monitoring

### Agent Deployment

Use **Zabbix Agent 2** for all Windows hosts. It provides native plugins for IIS, MSSQL, Active Directory, and Windows services.

**Deployment methods:**
- **MSI installer** via GPO for domain-joined machines
- **SCCM/Intune** for managed deployments
- **Ansible** (`win_package` module) for cross-platform consistency

**Agent configuration (`zabbix_agent2.conf`):**
```ini
ServerActive=proxy-siteA-01;proxy-siteA-02
Server=proxy-siteA-01,proxy-siteA-02
Hostname=WIN-SERVER-01
HostMetadata=Windows IIS SQLServer Production
TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=PSK_WIN_SERVER_01
TLSPSKFile=C:\Program Files\Zabbix Agent 2\zabbix_agent2.psk
Plugins.SystemRun.LogRemoteCommands=0
```

### Key Templates

| Template | Monitors |
|----------|----------|
| Windows by Zabbix agent active | CPU, memory, disk, network, services, uptime |
| IIS by Zabbix agent | Application pools, sites, requests, connections |
| MSSQL by ODBC | Connections, buffer cache, page life expectancy, locks, transaction log |
| Active Directory DS by Zabbix agent | LDAP binds, Kerberos auth, replication, DRA counters |
| Windows services by Zabbix agent active | Auto-discovered service state monitoring |

### Key Windows Metrics

**Core OS:**
```
perf_counter_en["\Processor(_Total)\% Processor Time"]
perf_counter_en["\Memory\Available Bytes"]
perf_counter_en["\Memory\Pages/sec"]
perf_counter_en["\PhysicalDisk(_Total)\% Disk Time"]
perf_counter_en["\PhysicalDisk(_Total)\Current Disk Queue Length"]
```

**Active Directory:**
```
perf_counter_en["\NTDS\DRA Pending Replication Syncs"]
perf_counter_en["\NTDS\LDAP Successful Binds/sec"]
perf_counter_en["\NTDS\Kerberos Authentications"]
perf_counter_en["\DNS\Total Query Received/sec"]
```

**SQL Server:**
```
perf_counter_en["\SQLServer:Buffer Manager\Buffer cache hit ratio"]
perf_counter_en["\SQLServer:Buffer Manager\Page life expectancy"]
perf_counter_en["\SQLServer:General Statistics\User Connections"]
perf_counter_en["\SQLServer:Locks(_Total)\Lock Waits/sec"]
```

### Windows Event Log Monitoring

```ini
# Agent configuration for event log collection
# Collect critical/error events from System log
eventlog[System,,"Warning|Error|Critical",,,,skip]

# Security audit failures
eventlog[Security,,"Audit Failure",,,,skip]

# Application errors
eventlog[Application,,"Error|Critical",,,skip]
```

### Auto-Registration for Windows

**Agent HostMetadata strategy:**
```ini
# Encode OS + roles + environment in metadata
HostMetadata=Windows IIS SQLServer Production DMZ
HostMetadata=Windows DC DNS DHCP Production Internal
HostMetadata=Windows FileServer Production Internal
```

**Zabbix auto-registration action:**

| Condition | Operations |
|-----------|------------|
| Metadata contains "Windows" | Add to group: Windows Servers, Link: Windows by Zabbix agent active |
| Metadata contains "IIS" | Add to group: Web Servers, Link: IIS by Zabbix agent |
| Metadata contains "SQLServer" | Add to group: Database Servers, Link: MSSQL by ODBC |
| Metadata contains "DC" | Add to group: Domain Controllers, Link: Active Directory DS |

---

## 11. Linux Monitoring

### Agent Deployment at Scale

Use **Ansible** for cross-platform (Windows + Linux) consistency:

```yaml
# roles/zabbix_agent2/tasks/main.yml (simplified)
- name: Install Zabbix Agent 2
  package:
    name: zabbix-agent2
    state: present

- name: Install Agent 2 plugins
  package:
    name: "{{ item }}"
    state: present
  loop: "{{ zabbix_agent2_plugins | default([]) }}"

- name: Deploy agent configuration
  template:
    src: zabbix_agent2.conf.j2
    dest: /etc/zabbix/zabbix_agent2.conf
    owner: root
    group: zabbix
    mode: '0640'
  notify: Restart zabbix-agent2

- name: Deploy PSK file
  copy:
    content: "{{ zabbix_psk_value }}"
    dest: /etc/zabbix/zabbix_agent2.psk
    owner: root
    group: zabbix
    mode: '0640'
  when: zabbix_psk_value is defined
```

### Key Templates

| Template | Targets |
|----------|---------|
| Linux by Zabbix agent active | All Linux: CPU, memory, disk, network, processes |
| Systemd by Zabbix agent 2 | systemd service unit monitoring |
| Nginx by Zabbix agent 2 | Nginx connections, requests, status |
| Apache by Zabbix agent 2 | Apache workers, requests, mod_status |
| PostgreSQL by Zabbix agent 2 | Connections, replication lag, locks, WAL stats |
| MySQL by Zabbix agent 2 | Threads, InnoDB, replication, slow queries |
| Docker by Zabbix agent 2 | Container discovery, CPU/memory/network per container |

### Key Linux Metrics

| Category | Items | Interval |
|----------|-------|----------|
| CPU utilization | `system.cpu.util[,user/system/iowait/steal]`, `system.cpu.load[all,avg1/5/15]` | 60s |
| Memory | `vm.memory.size[available/total/used]`, swap usage | 60s |
| Disk space | `vfs.fs.size[{#FSNAME},pfree]`, `vfs.fs.inode[{#FSNAME},pfree]` | 300s |
| Disk I/O | `vfs.dev.read/write[{#DEVNAME}]`, `vfs.dev.read/write.await` | 60s |
| Network | `net.if.in/out[{#IFNAME}]`, errors, drops | 60s |
| Processes | `proc.num[,,run/zomb]`, named process CPU/mem | 60s |
| Log files | `log[/var/log/syslog,"error\|critical"]` | 60s |

### Log File Monitoring Best Practices

```ini
# Always use "skip" mode to avoid processing old entries on agent restart
log[/var/log/syslog,"error|critical|alert",,100,,skip]
log[/var/log/auth.log,"Failed password",,50,,skip]

# Use logrt for rotated files
logrt[/var/log/nginx/error.log.*,"",,200,,skip]
```

### Auto-Registration for Linux

**HostMetadata encoding:**
```ini
# Ansible template
HostMetadata=Linux {{ role }} {{ environment }}
# Examples:
# HostMetadata=Linux Nginx Production
# HostMetadata=Linux PostgreSQL Production
# HostMetadata=Linux Docker Staging
```

---

## 12. Kubernetes Monitoring

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                       │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ kube-state-  │  │ Zabbix Proxy │  │ Zabbix Agent  │  │
│  │ metrics      │──│ (active,     │──│ DaemonSet     │  │
│  │ (Deployment) │  │  sqlite3)    │  │ (1 per node)  │  │
│  └──────────────┘  └──────┬───────┘  └───────────────┘  │
│                           │                              │
│  K8s API Server ──────────┘                              │
└───────────────────────────┼──────────────────────────────┘
                            │ Port 10051 (outbound)
                            ▼
                   [External Zabbix Server]
```

### Helm Chart Deployment

```bash
# Add official Zabbix Helm repo
helm repo add zabbix-chart-7.2 \
  https://cdn.zabbix.com/zabbix/integrations/kubernetes-helm/7.2

# Export and customize values
helm show values zabbix-chart-7.2/zabbix-helm-chrt > k8s-zabbix-values.yaml

# Install in dedicated namespace
kubectl create namespace monitoring
helm install zabbix zabbix-chart-7.2/zabbix-helm-chrt \
  --dependency-update \
  -f k8s-zabbix-values.yaml \
  -n monitoring
```

**Key values to customize:**
```yaml
zabbixProxy:
  enabled: true
  env:
    ZBX_SERVER_HOST: "proxy-siteA-01"    # External Zabbix server/proxy address
    ZBX_PROXYMODE: 0                      # Active mode
    ZBX_CACHESIZE: "256M"                 # Increase for large clusters

zabbixAgent:
  enabled: true
  tolerations:
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"               # Monitor control plane nodes

kube-state-metrics:
  enabled: true                           # Set false if KSM already deployed
```

### Template Setup

After Helm deployment, configure these on the Zabbix server:

| Host to Create | Template to Link | Key Macros |
|----------------|-----------------|------------|
| "K8s Cluster State" | Kubernetes cluster state by HTTP | `{$KUBE.API.URL}`, `{$KUBE.API.TOKEN}` |
| "K8s Nodes" | Kubernetes nodes by HTTP | `{$KUBE.API.URL}`, `{$KUBE.API.TOKEN}` |
| Auto-discovered | Kubernetes kubelet by HTTP | Auto-assigned |
| Auto-discovered | Kubernetes API server by HTTP | Auto-assigned |
| Auto-discovered | Kubernetes Controller manager by HTTP | Auto-assigned |
| Auto-discovered | Kubernetes Scheduler by HTTP | Auto-assigned |

**Retrieve the service account token:**
```bash
kubectl get secret zabbix-service-account -n monitoring \
  -o jsonpath='{.data.token}' | base64 -d
```

### What Gets Monitored

**Cluster State (auto-discovered):**
- Deployments, StatefulSets, DaemonSets, ReplicaSets (replica counts, availability)
- Pods (phase, restart count, conditions, container states)
- PVs/PVCs (capacity, phase)
- Jobs, CronJobs (completion, scheduling)
- Namespaces, Services, Endpoints

**Per Node (34 items):**
- CPU/memory capacity and allocatable
- Pod capacity and current count
- Node conditions (Ready, DiskPressure, MemoryPressure, PIDPressure)
- Resource requests vs. limits vs. allocatable

**Per Pod (8 items):**
- Phase (Running/Pending/Failed)
- Container ready/initialized/scheduled conditions
- Restart count and uptime

**Control Plane:**
- API server request rates, latencies, error rates, certificate expiry
- Controller manager and scheduler metrics
- etcd (via separate etcd by HTTP template)

### Built-In Triggers

| Trigger | Severity |
|---------|----------|
| Pod crash looping (restarts > 1 in 15min) | Warning |
| Pod not ready for 10+ minutes | High |
| Node Not Ready | Warning |
| Disk/Memory/PID pressure on node | Warning |
| CPU/Memory over-commitment (> 90/100% of allocatable) | Warning/Average |
| Deployment replica mismatch | Warning |
| PVC stuck in Pending | Warning |
| API server 5xx error rate exceeded | Average |
| Certificate expiration (7d / 24h) | Warning/High |

### Filtering for Large Clusters

Set these macros to limit discovery scope:

```
{$KUBE.LLD.FILTER.POD.NAMESPACE.MATCHES}       = "production|staging"
{$KUBE.LLD.FILTER.POD.NAMESPACE.NOT_MATCHES}    = "kube-system|monitoring"
{$KUBE.POD.FILTER.LABELS}                       = "app=.*"
{$KUBE.NODE.FILTER.LABELS}                      = "role=worker"
```

### Custom Triggers to Add

The default templates have known gaps. Add these:

1. **OOMKilled detection** -- watch container terminated reason
2. **ImagePullBackOff** -- watch container waiting state
3. **CrashLoopBackOff** -- explicit status string detection ([ZBX-21571](https://support.zabbix.com/browse/ZBX-21571))
4. **Pod eviction** -- monitor eviction events
5. **PV utilization percentage** -- capacity vs. usage

---

## 13. Microsoft 365 Monitoring

### Prerequisites

1. **Azure App Registration** in Microsoft Entra admin center
2. **Internet access** from Zabbix server/proxy to `graph.microsoft.com` and `login.microsoftonline.com`
3. **Entra ID P1/P2 license** (only needed for sign-in logs and risky sign-in monitoring)

### Azure App Registration

1. Go to **Microsoft Entra admin center > App registrations > New registration**
2. Name: "Zabbix M365 Monitoring"
3. Account type: "Accounts in this organizational directory only"
4. No redirect URI needed (client credentials flow)
5. Generate a **client secret** (Certificates & secrets > New client secret)

**Required API Permissions (Application type):**

| Permission | Purpose | Phase |
|------------|---------|-------|
| `Reports.Read.All` | Usage reports (Exchange, Teams, SharePoint, OneDrive) | Core |
| `ServiceHealth.Read.All` | Service health status and incidents | Core |
| `Organization.Read.All` | License/subscription data | Phase 2 |
| `Application.Read.All` | App registration secret expiry | Phase 2 |
| `AuditLog.Read.All` | Sign-in and audit logs | Phase 3 |
| `IdentityRiskEvent.Read.All` | Risk detections | Phase 3 |
| `IdentityRiskyUser.Read.All` | Risky users | Phase 3 |
| `CallRecords.Read.All` | Teams call records | Phase 4 |

**Grant admin consent** after adding permissions.

### Zabbix Configuration

1. Import the official **"Microsoft 365 reports by HTTP"** template
2. Create a host (e.g., "Microsoft 365") -- no agent needed
3. **Assign the host to a proxy group** (e.g., `SiteA-Proxies` or a dedicated cross-site group) so that HTTP checks automatically failover if one site's proxies go down
4. Link the template and set macros:

| Macro | Value |
|-------|-------|
| `{$MS365.TENANT.ID}` | Your Azure tenant ID |
| `{$MS365.APP.ID}` | Application (client) ID |
| `{$MS365.PASSWORD}` | Client secret (store as **secret macro**) |

> **Resilience:** The M365 template uses HTTP agent items (server-side HTTP requests). By assigning the M365 host to a **proxy group** that spans both sites, the HTTP calls automatically fail over to a proxy at the surviving site if one site goes down. Ensure proxies at both sites have outbound HTTPS access to `graph.microsoft.com` and `login.microsoftonline.com`.

### What the Official Template Monitors

| Service | Metrics | Collection Interval |
|---------|---------|---------------------|
| **Exchange Online** | Email activity (send/receive/read), mailbox counts, storage, client app usage | 6 hours |
| **SharePoint** | Active users (via M365 active user counts) | 6 hours |
| **OneDrive** | File activity, user counts, storage usage | 6 hours |
| **Teams** | Active users (via M365 active user counts) | 6 hours |
| **Office Apps** | Per-app user counts (Word, Excel, PowerPoint, Outlook), per-platform usage | 6 hours |
| **Service Health** | Per-service health status (LLD), degradation/interruption triggers | 6 hours |
| **Copilot** (7.4+) | Enabled vs. active users per application | 6 hours |

### Gaps Requiring Custom Implementation

| Capability | Approach | Complexity |
|------------|----------|------------|
| **License usage/compliance** | Custom script item querying `/subscribedSkus` | Low |
| **Sign-in failures** | Custom script item querying `/auditLogs/signIns` | Medium |
| **Risky sign-ins** | Custom script item querying `/identityProtection/riskDetections` | Medium |
| **Teams call quality** | Custom script item querying `/communications/callRecords` | High |
| **Mail flow details** | Custom script item querying `/security/messageTrace` | High |
| **App secret expiry** | Community template from [gerardlemetayerc](https://github.com/gerardlemetayerc/zabbix-microsoft365-monitoring) | Low |

### Graph API Rate Limits

| API Category | Limit | Zabbix Impact |
|-------------|-------|---------------|
| Reports API | 5 req / 10 sec per app per tenant | Official template's 6-hour interval is safe |
| Identity Protection | 1 req / sec per tenant | Very restrictive; space out custom items |
| Global | 130,000 req / 10 sec per app | Not a concern |

**Best practices:**
- Keep the official template's 6-hour interval for reports (data is daily aggregates)
- Service health checks: 5-15 minute intervals are safe
- Sign-in/risk monitoring: no more than every 15-30 minutes
- License checks: every 6-12 hours
- Always use `$select` and `$filter` to reduce payload size

### Recommended Rollout Phases

| Phase | Templates/Items | Permissions | Effort |
|-------|----------------|-------------|--------|
| **1. Core** | Official M365 template | Reports.Read.All, ServiceHealth.Read.All | Low |
| **2. License & Apps** | Community template or custom | Organization.Read.All, Application.Read.All | Low-Medium |
| **3. Security** | Custom sign-in/risk items | AuditLog.Read.All, IdentityRisk*.Read.All | Medium-High |
| **4. Advanced** | Custom mail flow, call quality | CallRecords.Read.All, MessageTrace.Read.All | High |

---

## 14. Agent Deployment & Automation

### Deployment Matrix

| Platform | Agent | Method | Mode |
|----------|-------|--------|------|
| Windows Server | Agent 2 (MSI) | GPO / SCCM / Ansible | Active |
| RHEL / CentOS | Agent 2 (RPM) | Ansible / Puppet / Salt | Active |
| Ubuntu / Debian | Agent 2 (DEB) | Ansible / Puppet / Salt | Active |
| SLES | Agent 2 (RPM) | Ansible / Salt | Active |
| Kubernetes | Agent 2 (container) | Helm chart DaemonSet | Passive (to in-cluster proxy) |

### Auto-Registration Workflow

```
1. Agent starts with HostMetadata describing its role
2. Agent connects to proxy (ServerActive)
3. Proxy forwards registration to Zabbix server
4. Auto-registration action matches metadata patterns
5. Host created, added to groups, templates linked
6. Monitoring begins within seconds
```

### Template Linking Strategy

```
Template Hierarchy:
├── Base OS Templates
│   ├── Linux by Zabbix agent active
│   └── Windows by Zabbix agent active
│
├── Role Templates (linked via auto-registration)
│   ├── Nginx by Zabbix agent 2
│   ├── PostgreSQL by Zabbix agent 2
│   ├── Docker by Zabbix agent 2
│   ├── IIS by Zabbix agent
│   ├── MSSQL by ODBC
│   └── Active Directory DS
│
├── Custom Overlay Templates
│   ├── Custom - Security Monitoring (SSH brute force, account lockouts)
│   ├── Custom - Backup Verification
│   └── Custom - Certificate Expiry
│
└── Environment Threshold Templates (macro overrides)
    ├── Production Thresholds (tighter)
    ├── Staging Thresholds (moderate)
    └── Development Thresholds (relaxed)
```

Use **host-level macro overrides** to tune thresholds per host without duplicating templates:

```
Template default:    {$CPU.UTIL.CRIT} = 90
Host-level override: {$CPU.UTIL.CRIT} = 95   # DB server runs hotter
```

---

## 15. Alerting & Notification

### Media Type Setup

Zabbix 7.0 includes built-in Email, plus importable webhook integrations. Import webhook YAML files from **Alerts > Media types > Import**.

| Media Type | Ships With Zabbix? | Import Path |
|------------|-------------------|-------------|
| Email | Yes (pre-configured) | N/A |
| Microsoft Teams | Import required | `templates/media/msteams-workflow/` |
| Slack | Import required | `templates/media/slack/` |
| PagerDuty | Import required | `templates/media/pagerduty/` |

> 41 webhook integrations are available (Discord, Telegram, Jira, ServiceNow, Opsgenie, etc.).

**Global prerequisite:** Set `{$ZABBIX.URL}` at **Administration > Macros** to your Zabbix frontend URL (e.g., `https://zabbix.example.com`). Webhook messages use this to link back to the problem.

#### Email (Office 365)

**Alerts > Media types > Email:**

| Field | Value |
|-------|-------|
| Email provider | Office 365 relay |
| Email (from) | `Zabbix NOC <zabbix@example.com>` |
| SMTP helo | `example.com` |
| Connection security | STARTTLS |

> Office 365 relay auto-generates the SMTP server from your domain. `SmtpClientAuthentication` must be enabled in Office 365 for the sending mailbox.

#### Microsoft Teams Webhook

1. Import `media_ms_teams_workflow.yaml` from Zabbix media templates
2. In Teams: target channel > Manage channel > Connectors > create incoming webhook > copy URL
3. In Zabbix: **Alerts > Media types > MS Teams Workflow** > paste webhook URL
4. Assign to users: **Users > Users > [user] > Media tab > Add > Type: MS Teams Workflow**

#### Slack

1. Import `media_slack.yaml`
2. Create Slack App at `api.slack.com/apps` with scopes: `chat:write`, `groups:read`, `im:read`, `channels:read`
3. Set `bot_token` parameter in the media type
4. Assign to users with **Send to:** `#channel-name`

#### PagerDuty

1. Import `media_pagerduty.yaml`
2. In PagerDuty: create a "Zabbix Webhook" integration > copy **Integration Key**
3. Set `token` parameter to the Integration Key in the media type
4. Assign to users

### User Groups

| User Group | Frontend Access | Host Permissions | Purpose |
|------------|----------------|------------------|---------|
| Zabbix Admins | Internal/LDAP | Read-write (all) | Full admin |
| NOC / Operations | LDAP | Read (all) | 24/7 first responders |
| App Team - [Name] | LDAP | Read (team host groups) | Per-team scoped visibility |
| Database Admins | LDAP | Read-write (DB groups) | DB-specific monitoring |
| Management | LDAP | Read (all) | Dashboards and reports |

### Escalation Policy

Create separate actions at **Alerts > Actions > Trigger actions** for each severity tier:

**Action 1: "Warning - Email Team"**

| Condition | Operations |
|-----------|------------|
| Severity = Warning | Step 1: Send Email to NOC / Operations |
| | Recovery: Notify all involved |

**Action 2: "High - Page After 15 Min"**

| Condition | Operations |
|-----------|------------|
| Severity = High | Step 1 (immediate): Send Email to NOC / Operations |
| | Step 2 (after 15 min): Send PagerDuty to NOC / Operations |
| | Recovery: Notify all involved |

**Action 3: "Disaster - Immediate Page + Escalate"**

| Condition | Operations |
|-----------|------------|
| Severity = Disaster | Step 1 (immediate): Send PagerDuty + Email to NOC / Operations |
| | Step 3 (after 30 min): Send PagerDuty + Email to Management |
| | Recovery: Notify all involved |

**How escalation steps work:** Set the **Default operation step duration** (e.g., 15 min). Step 1 executes immediately. Step 2 executes after one duration. Step 3 after two durations. Use step ranges `1-1`, `2-2`, `3-0` (3 onwards, indefinite) to control timing.

### Maintenance Windows

**Data collection > Maintenance > Create maintenance period**

| Type | Behavior | When to Use |
|------|----------|-------------|
| **With data collection** | Data collected, triggers evaluated, but **alerts suppressed** | Most maintenance (patching, updates) |
| **No data collection** | Data discarded, triggers skipped | Host fully offline (hardware work) |

- **One-time:** For ad-hoc patching (set exact date/time + duration)
- **Recurring:** For Patch Tuesday, weekly maintenance windows (set day/time + frequency)
- Suppressed problems hidden from dashboards by default. Enable **"Show suppressed problems"** in widgets to view.
- Action escalations pause during maintenance (controlled by **"Pause operations for suppressed problems"** checkbox in each action)

---

## 16. Encryption & Security

### PSK Encryption (Recommended Default)

**Generate unique PSK per host:**
```bash
openssl rand -hex 32 > /etc/zabbix/zabbix_agent2.psk
chmod 640 /etc/zabbix/zabbix_agent2.psk
chown root:zabbix /etc/zabbix/zabbix_agent2.psk
```

**Agent configuration:**
```ini
TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=PSK_<HOSTNAME>
TLSPSKFile=/etc/zabbix/zabbix_agent2.psk
```

**Proxy-to-server encryption:**
```ini
TLSConnect=psk
TLSPSKIdentity=PSK_PROXY_<SITE>_<NUM>
TLSPSKFile=/etc/zabbix/proxy.psk
```

### Certificate-Based Encryption (Enterprise Upgrade Path)

For environments with existing PKI:

```ini
TLSConnect=cert
TLSAccept=cert
TLSCAFile=/etc/zabbix/ssl/ca.crt
TLSCertFile=/etc/zabbix/ssl/agent.crt
TLSKeyFile=/etc/zabbix/ssl/agent.key
TLSServerCertIssuer=CN=Zabbix CA,O=YourOrg
TLSServerCertSubject=CN=zabbix.example.com,O=YourOrg
```

### Security Checklist

- [ ] All agent-server/proxy communication encrypted (PSK or TLS)
- [ ] Unique PSK per host (not shared)
- [ ] PSK values stored in Ansible Vault or equivalent
- [ ] M365 client secret stored as Zabbix secret macro
- [ ] Database passwords not in plaintext configs (use Vault)
- [ ] Frontend behind HTTPS (KEMP SSL offloading)
- [ ] Agent `system.run[*]` disabled (default in Agent 2)
- [ ] Zabbix API access restricted to authorized users
- [ ] Frontend session timeout configured
- [ ] Database replication traffic encrypted

---

## 17. Network & Firewall Requirements

### Core Zabbix Ports

| Port | Protocol | Source | Destination | Purpose |
|------|----------|--------|-------------|---------|
| 10050/TCP | Zabbix | Proxy | Agent | Passive agent checks |
| 10051/TCP | Zabbix | Agent | Proxy/Server | Active agent checks |
| 10051/TCP | Zabbix | Proxy | Server | Active proxy data sync |
| 443/TCP | HTTPS | Users | KEMP VIP | Web frontend (SSL offload) |
| 80/TCP | HTTP | KEMP | Frontend | Backend web (internal) |

### Database Cluster Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 5432/TCP | PostgreSQL | DB client connections |
| 2379/TCP | etcd | Client API (Patroni) |
| 2380/TCP | etcd | Peer communication |
| 8008/TCP | HTTP | Patroni REST API |
| 6973/TCP | KEMP HA Sync | Configuration sync between KEMP HA pair |
| 8443/TCP | HTTPS | KEMP WUI / REST API management |
| 224.0.0.18 (multicast) | CARP | KEMP HA heartbeat (IP protocol 112) |

### Cross-Site Requirements (Minimum)

| Flow | Ports | Purpose |
|------|-------|---------|
| Server Node A <-> DB VIP | 5432 | Database access |
| Server Node B <-> DB VIP | 5432 | Database access |
| PostgreSQL A <-> PostgreSQL B | 5432 | Replication |
| etcd-1 <-> etcd-2 <-> etcd-3 | 2379, 2380 | Consensus |
| Proxy-B <-> Server-A | 10051 | Data sync (when Server-A is active) |
| Proxy-A <-> Server-B | 10051 | Data sync (when Server-B is active) |
| KEMP-A VIP <-> Frontend-B | 80 | Frontend load balancing (if cross-site Real Servers used) |
| KEMP-B VIP <-> Frontend-A | 80 | Frontend load balancing (if cross-site Real Servers used) |

### External Access (Outbound)

| Flow | Destination | Ports | Purpose |
|------|-------------|-------|---------|
| Zabbix Server/Proxy | graph.microsoft.com | 443/TCP | M365 monitoring |
| Zabbix Server/Proxy | login.microsoftonline.com | 443/TCP | M365 OAuth |
| K8s Proxy | Zabbix Server | 10051/TCP | K8s monitoring data |

---

## 18. Backup & Disaster Recovery

### Database Backup with pgBackRest

pgBackRest is the recommended backup tool for Patroni-managed PostgreSQL. It provides incremental backups, parallel restore, WAL archiving, and delta restore (critical for rebuilding replicas without loading the primary).

**pgBackRest configuration (`/etc/pgbackrest.conf`) on each PostgreSQL node:**

> **Critical:** The backup repository must NOT be on the same disk as the database. If the DB VM's disk fails, you lose both the database and the backup. Use NFS, S3, or a dedicated backup server.

**Option A: NFS mount (recommended for on-prem):**
```ini
[global]
repo1-path=/mnt/pgbackrest-nfs           # NFS mount from shared storage
repo1-retention-full=2
repo1-retention-diff=7
process-max=4
start-fast=y
delta=y
compress-type=zst
compress-level=3
log-level-console=info
log-level-file=debug
log-path=/var/log/pgbackrest

[zabbix-cluster]
pg1-path=/var/lib/postgresql/16/main
pg1-port=5432
pg1-user=postgres
pg1-socket-path=/var/run/postgresql
```

Mount the NFS share on both PG nodes:
```bash
# /etc/fstab
nfs-server:/pgbackrest  /mnt/pgbackrest-nfs  nfs  defaults,noatime  0 0
```

**Option B: S3-compatible object store:**
```ini
[global]
repo1-type=s3
repo1-s3-endpoint=s3.your-provider.com
repo1-s3-bucket=pgbackrest-zabbix
repo1-s3-key=<access-key>
repo1-s3-key-secret=<secret-key>
repo1-s3-region=us-east-1
repo1-path=/repo1
repo1-retention-full=2
repo1-retention-diff=7
# ... same tuning params as above

[zabbix-cluster]
pg1-path=/var/lib/postgresql/16/main
pg1-port=5432
pg1-user=postgres
pg1-socket-path=/var/run/postgresql
```

**Patroni integration (`patroni.yml` additions):**

```yaml
postgresql:
  parameters:
    archive_mode: "on"
    archive_command: 'pgbackrest --stanza=zabbix-cluster archive-push %p'
    archive_timeout: 60
  recovery_conf:
    restore_command: pgbackrest --stanza=zabbix-cluster archive-get %f %p
  create_replica_methods:
    - pgbackrest
    - basebackup
  pgbackrest:
    command: pgbackrest --stanza=zabbix-cluster restore --type=none --delta
    keep_data: True
    no_params: True
  basebackup:
    checkpoint: 'fast'
    max-rate: '100M'
```

**Initialize and verify:**

```bash
sudo -iu postgres pgbackrest --stanza=zabbix-cluster stanza-create
sudo -iu postgres pgbackrest --stanza=zabbix-cluster check
```

### Backup Schedule

```cron
# /etc/cron.d/pgbackrest
# Full backup every Sunday at 01:00
0 1 * * 0   postgres   pgbackrest --stanza=zabbix-cluster --type=full backup 2>&1 | logger -t pgbackrest-full

# Differential backup Mon-Sat at 01:00
0 1 * * 1-6 postgres   pgbackrest --stanza=zabbix-cluster --type=diff backup 2>&1 | logger -t pgbackrest-diff
```

**Verification (run weekly):**

```bash
sudo -iu postgres pgbackrest --stanza=zabbix-cluster info     # List backups
sudo -iu postgres pgbackrest --stanza=zabbix-cluster verify   # Check integrity
```

### Zabbix Configuration File Backup

These files/directories must be backed up independently of the database:

| Path | Contents |
|------|----------|
| `/etc/zabbix/` | All server, proxy, agent, and frontend configs |
| `/etc/zabbix/*.psk` | PSK encryption keys (unique per host) |
| `/usr/lib/zabbix/alertscripts/` | Custom alert scripts |
| `/usr/lib/zabbix/externalscripts/` | External check scripts |
| `/etc/patroni/` | Patroni configuration per node |
| `/etc/pgbackrest.conf` | Backup configuration |
| `/etc/etcd/` | etcd configuration and TLS certificates |
| SSL/TLS certificates | Frontend, agent, and etcd certificates |

### Disaster Recovery Procedures

#### DR Scenario 1: Primary Site (A) Completely Lost

**Automatic failover (if etcd-3 is at Site B or third location):**

1. Patroni detects the primary's leader key has expired in etcd (within TTL, ~30s)
2. etcd-2 + etcd-3 form quorum (2 of 3 nodes)
3. Patroni at Site B auto-promotes PostgreSQL to primary
4. KEMP at Site B routes DB traffic to the new primary (Patroni `/primary` health check returns 200)
5. Zabbix Server Node-02 at Site B becomes active (detects writable database)

**Manual steps after automatic failover:**

```bash
# 1. Verify promotion
patronictl -c /etc/patroni/patroni.yml list

# 2. Verify KEMP routing (check WUI or REST API)
curl -k "https://bal:<password>@10.2.1.50:8443/access/showvs?vs=10.2.1.200&port=5432&prot=tcp"

# 3. If using Option A DNS strategy, update DNS:
#    zabbix.example.com -> 10.2.1.100 (Site B KEMP VIP)

# 4. Notify agents/proxies at Site A are offline until site rebuilt
```

**No automatic failover (if etcd-3 is at Site A):**

```bash
# Manual promotion required -- ONLY if you are certain Site A will not return
patronictl -c /etc/patroni/patroni.yml failover zabbix-cluster --candidate pg-siteB --force
```

> **Recommendation:** Place etcd-3 at Site B to enable automatic failover.

#### DR Scenario 2: Database Corruption -- Point-in-Time Recovery

```bash
# 1. Stop Patroni on ALL nodes
ssh pg-siteA "systemctl stop patroni"
ssh pg-siteB "systemctl stop patroni"

# 2. Remove cluster registration from etcd
patronictl -c /etc/patroni/patroni.yml remove zabbix-cluster

# 3. Restore to point before corruption on the target primary
sudo -iu postgres pgbackrest --stanza=zabbix-cluster \
  --delta --type=time \
  --target="2026-02-23 14:30:00+00" \
  --target-action=promote \
  restore

# 4. Start Patroni on restored node, then replica
systemctl start patroni   # On restored node (becomes primary)
# Wait until healthy, then:
systemctl start patroni   # On replica (re-syncs from primary)

# 5. Verify
patronictl -c /etc/patroni/patroni.yml list
```

#### DR Scenario 3: Rebuild a Failed Site

```bash
# 1. Reinstall OS and packages on rebuilt VMs (Section 4)
# 2. Restore config files from backup
# 3. Ensure patroni.yml has the same scope and etcd endpoints
# 4. Ensure data directory is empty
sudo -iu postgres rm -rf /var/lib/postgresql/16/main/*

# 5. Start Patroni -- it auto-discovers the cluster and creates a replica
systemctl start patroni

# 6. If the node has stale data, force a reinit
patronictl -c /etc/patroni/patroni.yml reinit zabbix-cluster pg-siteA --force --wait

# 7. Monitor replica sync progress
patronictl -c /etc/patroni/patroni.yml list
```

### DCS Failsafe Mode

Enable to prevent unnecessary demotions during temporary etcd outages:

```bash
patronictl -c /etc/patroni/patroni.yml edit-config -s failsafe_mode=true
```

When enabled: if etcd is unreachable but the primary can reach all replicas via Patroni REST API, the primary continues operating instead of demoting itself.

---

## 19. Operational Runbook

### Day-1 Deployment Order

1. **DNS records** -- Create all DNS entries (Section 3)
2. **OS & packages** -- Install base OS (Rocky 9), repos, and packages on all VMs (Section 4)
3. **Database schema** -- Create Zabbix DB, import schema, enable TimescaleDB (Section 4)
4. **etcd cluster** -- Deploy and bootstrap 3-node etcd cluster (Section 5)
5. **Patroni** -- Configure and start Patroni on both PostgreSQL nodes (Section 6)
6. **KEMP LoadMaster** -- Deploy VLM HA pairs, configure DB and frontend Virtual Services (Section 8)
7. **Zabbix Server** -- Configure HA nodes, start servers at both sites (Section 7)
8. **Frontends** -- Deploy Nginx + PHP-FPM, configure `zabbix.conf.php` (Section 8)
9. **Proxies** -- Deploy and register proxy groups at both sites (Section 9)
10. **Templates** -- Import all required templates and configure M365 (Sections 10-13)
11. **Alerting** -- Configure media types, user groups, escalation actions (Section 15)
12. **Auto-registration** -- Configure actions for Windows and Linux (Section 14)
13. **Agent rollout** -- Deploy agents via Ansible/GPO (Sections 10-11)
14. **K8s Helm** -- Deploy monitoring into each K8s cluster (Section 12)
15. **Backup** -- Configure pgBackRest, verify first full backup (Section 18)
16. **Validation** -- Verify HA failover, proxy failover, data flow, alerting

### HA Failover Testing

**Zabbix Server failover:**
```bash
# On active node -- simulate graceful failover
systemctl stop zabbix-server
# Verify standby promotes within ~5 seconds
zabbix_server -R ha_status    # Run on the other node

# Simulate ungraceful failure
kill -9 $(pidof zabbix_server)
# Verify failover within failover_delay + 5 seconds
```

**Database failover:**
```bash
# Patroni switchover (graceful)
patronictl -c /etc/patroni/patroni.yml switchover

# Verify via
patronictl -c /etc/patroni/patroni.yml list
```

**Proxy group failover:**
```bash
# Stop one proxy and verify hosts redistribute
systemctl stop zabbix-proxy
# Check in Frontend: Administration > Proxies -- hosts should move
```

### Upgrade Procedure

**Minor version (e.g., 7.0.1 -> 7.0.2):**
1. Upgrade standby node first, restart
2. Upgrade active node, restart (brief failover occurs)

**Major version (e.g., 7.0 -> 7.2):**
1. Stop all HA nodes
2. Comment out `HANodeName` on one node
3. Upgrade that node (runs DB schema migration)
4. Restore `HANodeName`, start it
5. Upgrade remaining nodes, start them

### Self-Monitoring: Infrastructure Stack Templates

Zabbix should monitor the infrastructure it runs on. Create these hosts and link the corresponding templates:

| Host to Create | Template | Key Macros | Notes |
|----------------|----------|------------|-------|
| vCenter Server | **VMware vCenter by HTTP** | `{$VMWARE.URL}`, `{$VMWARE.USER}`, `{$VMWARE.PASSWORD}` | Auto-discovers ESXi hosts, VMs, datastores, clusters |
| (auto-discovered) | **VMware Hypervisor by HTTP** | (inherited) | Per-ESXi CPU, memory, HBA, NIC, datastore |
| (auto-discovered) | **VMware Guest by HTTP** | (inherited) | Per-VM CPU, memory, disk, snapshot age |
| PostgreSQL (Site A) | **PostgreSQL by Zabbix agent 2** | `{$PG.URI}`, `{$PG.USER}`, `{$PG.PASSWORD}` | Connections, replication lag, locks, WAL, dead tuples |
| PostgreSQL (Site B) | **PostgreSQL by Zabbix agent 2** | (same) | Replica-specific: streaming status, replay lag |
| etcd-1, etcd-2, etcd-3 | **etcd by HTTP** | `{$ETCD.URL}` (e.g., `https://10.1.1.30:2379`) | Cluster health, leader changes, DB size, proposal failures |
| KEMP-A, KEMP-B | **SNMP Generic** + custom HTTP checks | SNMP community or HTTP items on port 8443 | VIP status, Real Server health, HA pair state |
| Zabbix Server | **Zabbix server health** | (built-in) | Cache utilization, process busy %, NVPS, queue |

**Patroni monitoring (custom HTTP items):**

Create items querying the Patroni REST API on each PG host:

| Item | URL | What It Returns |
|------|-----|-----------------|
| Cluster role | `http://10.1.1.20:8008/` | JSON with `role`, `state`, `timeline` |
| Is leader | `http://10.1.1.20:8008/leader` | HTTP 200 if leader, 503 if not |
| Replica health | `http://10.1.1.20:8008/health` | HTTP 200 if running |
| Cluster config | `http://10.1.1.20:8008/cluster` | All members, lag, timeline |

Create triggers: **role changed** (leader -> replica), **replication lag > 30s**, **Patroni node unreachable**.

### Monitoring the Monitoring (External Meta-Monitoring)

Set up these checks on an **independent system** (separate from Zabbix -- e.g., simple uptime monitor, Nagios, or cloud-based ping service):
- Zabbix frontend accessibility (HTTP check to KEMP VIP port 443)
- Zabbix server process (TCP check to port 10051)
- Database availability (TCP check to KEMP DB VIP port 5432)
- KEMP LoadMaster WUI accessibility (port 8443)
- Certificate expiry for all TLS endpoints
- M365 app registration secret expiry

### History & Housekeeping

Enable TimescaleDB-based housekeeping in Frontend:
- `Administration > Housekeeping`
- Enable "Override item history period" and "Override item trend period"
- Set periods per requirements (14d history, 365d trends)
- TimescaleDB drops entire partitions instead of row-by-row DELETE

---

## 20. Known Limitations & Gotchas

### Zabbix HA

| Issue | Impact | Mitigation |
|-------|--------|------------|
| Database is the SPOF for HA coordination | Server HA useless if DB is down | Independent DB HA (Patroni/Galera) is mandatory |
| Ungraceful failover delay (default 65s) | No monitoring during gap | Lower to 30s; use proxies to buffer data |
| Frontend hardcoded server address | Breaks HA failover | Never set `$ZBX_SERVER` when HA is enabled |
| `ServerActive` commas vs. semicolons | Data duplication with commas | Always use **semicolons** for `ServerActive` |

### Database

| Issue | Impact | Mitigation |
|-------|--------|------------|
| Cross-site sync replication + high latency | Reduced write throughput (NVPS ceiling) | Consider async replication if > 5ms RTT |
| etcd quorum with 2 sites | Impossible to achieve clean quorum | Deploy etcd-3 at a third location or accept risk |
| TimescaleDB compression immutability | Cannot modify compressed chunks | Plan retention periods carefully |
| TimescaleDB license | Compression requires Community license (not Apache) | Verify license compliance |

### Proxy Groups

| Issue | Impact | Mitigation |
|-------|--------|------------|
| SNMP traps not supported in proxy groups | SNMP trap receivers won't work | Use standalone proxies for SNMP traps |
| Active agent redirect loop on network partition | Agent stuck in infinite redirect | Manual intervention required |
| Pre-7.0 agents don't support redirection | No failover for old agents | Upgrade all agents to 7.0+ |

### Kubernetes

| Issue | Impact | Mitigation |
|-------|--------|------------|
| CrashLoopBackOff not triggering alerts by default | Missed pod failures | Add custom trigger ([ZBX-21571](https://support.zabbix.com/browse/ZBX-21571)) |
| Large clusters overwhelm discovery | Performance degradation | Use namespace/label/annotation filters |
| Control plane metrics may need binding address changes | Scheduler/Controller manager unreachable | Set `--binding-address` to 0.0.0.0 |

### VMware / KEMP LoadMaster

| Issue | Impact | Mitigation |
|-------|--------|------------|
| KEMP VLM license tied to MAC address | VM migration can break license | Assign **static MACs** to all KEMP VM NICs; preserve MACs on migration |
| vSwitch security policies for CARP | KEMP HA failover fails silently if misconfigured | Force **MAC Address Changes: Accept** and **Forged Transmits: Accept** on the port group |
| KEMP HA requires same broadcast domain | Cannot span HA pair across sites | Deploy separate HA pair per site; use DNS or GSLB for cross-site failover |
| Patroni health check on KEMP | KEMP checks port 8008 but serves traffic on 5432 | Verify **Check Port** is set to 8008, not the default service port |
| VMware DRS/vMotion with KEMP | Live migration can change MAC addresses | Pin KEMP VMs to specific hosts or use DRS affinity rules; verify static MACs survive vMotion |
| KEMP WUI default port 8443 | Conflicts if other services use 8443 | Change WUI port if needed; update firewall rules accordingly |

### Microsoft 365

| Issue | Impact | Mitigation |
|-------|--------|------------|
| Report data is daily aggregates (not real-time) | No minute-by-minute M365 metrics | Accept limitation; use for trend monitoring |
| Identity Protection API: 1 req/sec limit | Very restrictive polling | 15-30 min intervals; exponential backoff |
| Entra ID P1/P2 required for sign-in logs | Additional license cost | Phase 3 deployment only where licensed |
| App secret expiration | Monitoring breaks silently | Monitor secret expiry; set 12-24 month rotation |
| Graph API beta version used by default | Potential breaking changes | Pin version; test after Zabbix upgrades |

---

## References

### Official Documentation
- [Zabbix HA Documentation](https://www.zabbix.com/documentation/current/en/manual/concepts/server/ha)
- [Zabbix Proxy Load Balancing and HA](https://www.zabbix.com/documentation/current/en/manual/distributed_monitoring/proxies/ha)
- [Zabbix Requirements](https://www.zabbix.com/documentation/current/en/manual/installation/requirements)
- [Zabbix TimescaleDB Setup](https://www.zabbix.com/documentation/7.0/en/manual/appendix/install/timescaledb)
- [Zabbix 7.0 What's New](https://www.zabbix.com/documentation/7.0/en/manual/introduction/whatsnew700)

### Zabbix Blog
- [Building HA Zabbix with PostgreSQL and Patroni](https://blog.zabbix.com/building-ha-zabbix-with-postgresql-and-patroni/30960/)
- [Running Zabbix with PG Auto Failover](https://blog.zabbix.com/running-zabbix-with-postgresql-and-pg-auto-failover/31026/)
- [Running Zabbix with MariaDB and Galera](https://blog.zabbix.com/running-zabbix-with-mariadb-and-galera-active-active-clustering/31104/)
- [Zabbix 7.0 Proxy Load Balancing](https://blog.zabbix.com/zabbix-7-0-proxy-load-balancing/28173/)
- [Monitoring Kubernetes with Zabbix (3-Part Series)](https://blog.zabbix.com/monitoring-kubernetes-with-zabbix/25055/)
- [Zabbix Proxy Performance Tuning](https://blog.zabbix.com/zabbix-proxy-performance-tuning-and-troubleshooting/14013/)

### Integrations
- [Zabbix Kubernetes Integration](https://www.zabbix.com/integrations/kubernetes)
- [Zabbix Microsoft 365 Integration](https://www.zabbix.com/integrations/ms365)
- [Zabbix etcd Integration](https://www.zabbix.com/integrations/etcd)

### Microsoft Graph API
- [Graph API Throttling Guidance](https://learn.microsoft.com/en-us/graph/throttling)
- [Service Communications API](https://learn.microsoft.com/en-us/graph/api/resources/service-communications-api-overview)
- [Identity Protection APIs](https://learn.microsoft.com/en-us/graph/api/resources/identityprotection-overview)

### KEMP LoadMaster
- [KEMP LoadMaster Product Overview](https://docs.progress.com/bundle/loadmaster-product-overview-progress-kemp-loadmaster-ltsf/page/Welcome.html)
- [KEMP VMware Deployment Guide](https://support.kemptechnologies.com/hc/en-us/articles/203123629-VMware)
- [KEMP HA Configuration](https://docs.progress.com/bundle/loadmaster-product-overview-progress-kemp-loadmaster-ltsf/page/High-Availability-HA-Configuration.html)
- [KEMP Health Checking](https://support.kemptechnologies.com/hc/en-us/articles/202214518-Health-Checking)
- [KEMP REST API Documentation](https://docs.progress.com/bundle/loadmaster-interface-description-restful-api-ga/page/Introduction.html)
- [KEMP PowerShell SDK](https://kemptechnologies.github.io/PowerShell-sdk-vnext/ps-help.html)

### Community
- [Zabbix Community Helm Chart](https://github.com/zabbix-community/helm-zabbix)
- [M365 Community Template](https://github.com/gerardlemetayerc/zabbix-microsoft365-monitoring)
- [Initmax: HA for Zabbix 7.0 (PDF)](https://www.initmax.cz/wp-content/uploads/2024/08/high-availability-for-zabbix-server-and-zabbix-proxy-7.0.pdf)
