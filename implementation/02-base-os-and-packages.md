# Phase 2 — Base OS and Package Installation

## Objective

Install Rocky Linux 9 base configuration on all VMs, configure package repositories, install role-specific packages, create the Zabbix database, import the schema, and configure SELinux and firewalld.

## Prerequisites

- [ ] Phase 1 complete — all VMs provisioned, DNS verified, NTP synchronized, cross-site connectivity confirmed
- [ ] All VMs running Rocky Linux 9 minimal install
- [ ] Internet access available from VMs (or local mirrors configured for air-gapped environments)
- [ ] `{{DB_ZABBIX_PASSWORD}}` generated and stored in vault

---

## Step 1 — Base OS Configuration (ALL VMs)

Run the following on **every** provisioned VM.

### 1a. Set Hostname

```bash
# Replace with the appropriate hostname for each VM
hostnamectl set-hostname <vm-hostname>.{{DOMAIN}}
```

Use the hostnames from the Phase 1 provisioning tables:

| Site A | Site B |
|--------|--------|
| `zabbix-node-siteA` | `zabbix-node-siteB` |
| `pg-siteA` | `pg-siteB` |
| `etcd-1` | `etcd-2` |
| `etcd-3` | — |
| `frontend-siteA` | `frontend-siteB` |
| `proxy-siteA-01` | `proxy-siteB-01` |
| `proxy-siteA-02` | `proxy-siteB-02` |
| `proxy-siteA-snmptrap` | `proxy-siteB-snmptrap` |

### 1b. Configure Static IP

Edit the appropriate NetworkManager connection. Example for the primary interface:

```bash
# Identify the connection name
nmcli con show

# Configure static IP (Site A example for Zabbix Server)
nmcli con mod "System eth0" \
  ipv4.method manual \
  ipv4.addresses {{ZBX_SERVER_A_IP}}/24 \
  ipv4.gateway {{SITE_A_GATEWAY}} \
  ipv4.dns "{{NTP_SERVER_1}}" \
  connection.autoconnect yes

nmcli con up "System eth0"
```

> **Note:** Adjust the interface name, IP address, subnet mask, and gateway for each VM per the environment variables catalog. Site B VMs use `{{SITE_B_GATEWAY}}`.

### 1c. Configure chrony (if not already done in Phase 1)

```bash
dnf install -y chrony
```

Edit `/etc/chrony.conf`:

```ini
server {{NTP_SERVER_1}} iburst
server {{NTP_SERVER_2}} iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
```

```bash
systemctl enable --now chronyd
chronyc tracking
```

### 1d. Install Common Utilities

```bash
dnf install -y \
  vim \
  wget \
  curl \
  net-tools \
  nmap-ncat \
  bind-utils \
  bash-completion \
  tar \
  unzip \
  policycoreutils-python-utils
```

### 1e. Disable SELinux Only If Required

The `zabbix-selinux-policy` package (installed later) handles most Zabbix-specific SELinux rules. Keep SELinux in **enforcing** mode unless specific issues arise.

```bash
# Verify current mode
getenforce

# Only if absolutely necessary (document the reason):
# setenforce 0
# sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

---

## Step 2 — Database Packages (Database VMs Only)

Run on `pg-siteA` and `pg-siteB`.

### 2a. Install PostgreSQL 16 from PGDG Repository

```bash
# Add PGDG repository
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Disable the built-in PostgreSQL module to avoid conflicts
dnf -qy module disable postgresql

# Install PostgreSQL 16
dnf install -y postgresql16-server postgresql16
```

### 2b. Install TimescaleDB 2.x

```bash
# Add TimescaleDB repository
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

# Install TimescaleDB for PostgreSQL 16
dnf install -y timescaledb-2-postgresql-16 timescaledb-2-loader-postgresql-16
```

### 2c. Install Patroni and patroni-etcd

```bash
# Enable PGDG extras repo
dnf install -y 'dnf-command(config-manager)'
dnf config-manager --enable pgdg-rhel9-extras

# Install Patroni with etcd support
dnf install -y patroni patroni-etcd

# Verify installation
patroni --version
patronictl version
```

Create the Patroni configuration directory:

```bash
useradd -r -s /sbin/nologin patroni 2>/dev/null || true
mkdir -p /etc/patroni/pki
chown postgres:postgres /etc/patroni
chown postgres:postgres /etc/patroni/pki
chmod 750 /etc/patroni
chmod 750 /etc/patroni/pki
```

### 2d. Install pgBackRest

```bash
dnf install -y pgbackrest

# Create log directory
mkdir -p /var/log/pgbackrest
chown postgres:postgres /var/log/pgbackrest
```

### 2e. Install PgBouncer

```bash
dnf install -y pgbouncer

# Create auth file
touch /etc/pgbouncer/userlist.txt
chown pgbouncer:pgbouncer /etc/pgbouncer/userlist.txt
chmod 640 /etc/pgbouncer/userlist.txt
```

---

## Step 3 — Zabbix Repository (ALL Zabbix VMs)

Run on all Zabbix Server, Frontend, Proxy, and SNMP Trap Proxy VMs.

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rocky/9/x86_64/zabbix-release-latest-7.0.el9.noarch.rpm
dnf clean all
```

### Handle EPEL Conflict

If EPEL is enabled on any VM, exclude Zabbix packages to prevent version conflicts:

```bash
# Check if EPEL is enabled
dnf repolist | grep -i epel

# If EPEL is present, add exclusion
if [ -f /etc/yum.repos.d/epel.repo ]; then
  grep -q "excludepkgs=zabbix\*" /etc/yum.repos.d/epel.repo || \
    sed -i '/^\[epel\]/a excludepkgs=zabbix*' /etc/yum.repos.d/epel.repo
  echo "EPEL exclusion added for zabbix packages"
fi
```

---

## Step 4 — Zabbix Server Packages (Server VMs Only)

Run on `zabbix-node-siteA` and `zabbix-node-siteB`.

```bash
dnf install -y \
  zabbix-server-pgsql \
  zabbix-sql-scripts \
  zabbix-selinux-policy \
  zabbix-agent2
```

Verify:

```bash
rpm -q zabbix-server-pgsql zabbix-sql-scripts zabbix-selinux-policy zabbix-agent2
```

---

## Step 5 — Zabbix Frontend Packages (Frontend VMs Only)

Run on `frontend-siteA` and `frontend-siteB`.

```bash
dnf install -y \
  zabbix-web-pgsql \
  zabbix-nginx-conf \
  zabbix-selinux-policy
```

Verify:

```bash
rpm -q zabbix-web-pgsql zabbix-nginx-conf zabbix-selinux-policy
```

---

## Step 6 — Zabbix Proxy Packages (Proxy VMs Only)

Run on `proxy-siteA-01`, `proxy-siteA-02`, `proxy-siteB-01`, `proxy-siteB-02`, `proxy-siteA-snmptrap`, and `proxy-siteB-snmptrap`.

Choose the backend based on expected proxy NVPS:

```bash
# Option A: SQLite backend (< 1K NVPS per proxy — simpler, no DB management)
dnf install -y \
  zabbix-proxy-sqlite3 \
  zabbix-selinux-policy \
  zabbix-agent2

# Option B: PostgreSQL backend (> 1K NVPS per proxy — better for high volume)
dnf install -y \
  zabbix-proxy-pgsql \
  zabbix-sql-scripts \
  zabbix-selinux-policy \
  zabbix-agent2
```

For SNMP trap proxy VMs, also install the SNMP trap handler:

```bash
# On proxy-siteA-snmptrap and proxy-siteB-snmptrap
dnf install -y \
  zabbix-proxy-sqlite3 \
  zabbix-selinux-policy \
  zabbix-agent2 \
  net-snmp \
  net-snmp-utils \
  net-snmp-perl
```

Verify:

```bash
rpm -q zabbix-proxy-sqlite3 zabbix-selinux-policy zabbix-agent2
# OR
rpm -q zabbix-proxy-pgsql zabbix-sql-scripts zabbix-selinux-policy zabbix-agent2
```

---

## Step 7 — Zabbix Agent 2 (All Monitored Hosts)

For any additional hosts that need Zabbix Agent 2 (beyond the Zabbix infrastructure VMs that already received it in Steps 4-6):

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rocky/9/x86_64/zabbix-release-latest-7.0.el9.noarch.rpm
dnf clean all
dnf install -y zabbix-agent2
```

Optional Agent 2 plugins for database monitoring:

```bash
# PostgreSQL monitoring plugin (install on database VMs)
dnf install -y zabbix-agent2-plugin-postgresql

# Other available plugins
# dnf install -y zabbix-agent2-plugin-mongodb
# dnf install -y zabbix-agent2-plugin-mssql
```

---

## Step 8 — Database Creation and Schema Import

> **CRITICAL:** This step runs on the **initial primary PostgreSQL node only** (`pg-siteA`). It must be completed **BEFORE** Patroni takes over PostgreSQL management. Do NOT run this on `pg-siteB`.

### 8a. Initialize PostgreSQL (Standalone Mode)

```bash
/usr/pgsql-16/bin/postgresql-16-setup initdb
systemctl start postgresql-16
```

### 8b. Create Zabbix User and Database

```bash
sudo -u postgres createuser --pwprompt zabbix
# Enter {{DB_ZABBIX_PASSWORD}} when prompted

sudo -u postgres createdb -O zabbix -E Unicode -T template0 zabbix
```

### 8c. Import Zabbix Schema

```bash
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix
```

> **Note:** This import takes 1-5 minutes depending on disk speed. Do not interrupt it. You will see no output during the import — this is normal.

### 8d. Configure TimescaleDB

```bash
# Run TimescaleDB tuning (adjusts postgresql.conf for TimescaleDB)
timescaledb-tune --pg-config=/usr/pgsql-16/bin/pg_config --quiet --yes

# Restart PostgreSQL to apply tuning changes
systemctl restart postgresql-16
```

### 8e. Enable TimescaleDB Extension

```bash
echo "CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;" | sudo -u postgres psql zabbix
```

### 8f. Import Zabbix TimescaleDB Migration

```bash
cat /usr/share/zabbix-sql-scripts/postgresql/timescaledb/schema.sql | sudo -u zabbix psql zabbix
```

### 8g. Verify Schema Import

```bash
# Verify tables exist
sudo -u postgres psql zabbix -c "\dt" | head -20

# Verify TimescaleDB extension is active
sudo -u postgres psql zabbix -c "SELECT extname, extversion FROM pg_extension WHERE extname = 'timescaledb';"

# Verify TimescaleDB hypertables were created
sudo -u postgres psql zabbix -c "SELECT hypertable_name FROM timescaledb_information.hypertables;"
```

### 8h. Stop and Disable PostgreSQL

Patroni will manage PostgreSQL from this point forward. Stop and disable the standalone service:

```bash
systemctl stop postgresql-16
systemctl disable postgresql-16
```

> **WARNING:** After this step, do NOT start `postgresql-16` directly. Patroni manages the PostgreSQL process. Starting it manually will cause conflicts.

---

## Step 9 — SELinux Booleans

Run on the appropriate VMs based on role.

### Frontend VMs

```bash
setsebool -P httpd_can_connect_zabbix on
setsebool -P httpd_can_network_connect_db on
setsebool -P zabbix_can_network on
```

### Server VMs

```bash
setsebool -P zabbix_can_network on
```

### Proxy VMs

```bash
setsebool -P zabbix_can_network on
```

Verify booleans are set:

```bash
getsebool httpd_can_connect_zabbix httpd_can_network_connect_db zabbix_can_network
```

---

## Step 10 — Firewalld Configuration

Rocky Linux 9 ships with `firewalld` enabled. Open ports according to each VM's role.

> **CRITICAL:** Only open the ports relevant to each VM's role. Do NOT run all commands on every VM.

### Database VMs (`pg-siteA`, `pg-siteB`)

```bash
firewall-cmd --permanent --add-port=5432/tcp     # PostgreSQL
firewall-cmd --permanent --add-port=6432/tcp     # PgBouncer
firewall-cmd --permanent --add-port=8008/tcp     # Patroni REST API (KEMP health checks)
firewall-cmd --permanent --add-port=10050/tcp    # Zabbix agent (self-monitoring)
firewall-cmd --reload
```

### etcd VMs (`etcd-1`, `etcd-2`, `etcd-3`)

```bash
firewall-cmd --permanent --add-port=2379/tcp     # etcd client API
firewall-cmd --permanent --add-port=2380/tcp     # etcd peer communication
firewall-cmd --permanent --add-port=10050/tcp    # Zabbix agent (self-monitoring)
firewall-cmd --reload
```

### Zabbix Server VMs (`zabbix-node-siteA`, `zabbix-node-siteB`)

```bash
firewall-cmd --permanent --add-port=10051/tcp    # Zabbix trapper (agent/proxy connections)
firewall-cmd --permanent --add-port=10050/tcp    # Zabbix agent (self-monitoring)
firewall-cmd --reload
```

### Frontend VMs (`frontend-siteA`, `frontend-siteB`)

```bash
firewall-cmd --permanent --add-port=80/tcp       # Nginx HTTP (KEMP backend)
firewall-cmd --permanent --add-port=443/tcp      # Nginx HTTPS (if not using KEMP SSL offload)
firewall-cmd --permanent --add-port=10050/tcp    # Zabbix agent (self-monitoring)
firewall-cmd --reload
```

### Proxy VMs (`proxy-siteA-01`, `proxy-siteA-02`, `proxy-siteB-01`, `proxy-siteB-02`)

```bash
firewall-cmd --permanent --add-port=10051/tcp    # Zabbix trapper (agent connections)
firewall-cmd --permanent --add-port=10050/tcp    # Zabbix agent (self-monitoring)
firewall-cmd --reload
```

### SNMP Trap Proxy VMs (`proxy-siteA-snmptrap`, `proxy-siteB-snmptrap`)

```bash
firewall-cmd --permanent --add-port=162/udp      # SNMP traps inbound
firewall-cmd --permanent --add-port=10051/tcp    # Zabbix trapper
firewall-cmd --permanent --add-port=10050/tcp    # Zabbix agent (self-monitoring)
firewall-cmd --reload
```

### Verify Firewalld on Any VM

```bash
firewall-cmd --list-all
```

---

## Verification Checkpoint

Complete all checks before proceeding to Phase 3.

| # | Check | VM(s) | Command | Expected Result | Pass |
|---|-------|-------|---------|-----------------|------|
| 1 | PostgreSQL 16 installed | `pg-siteA`, `pg-siteB` | `rpm -q postgresql16-server` | Package version returned | [ ] |
| 2 | TimescaleDB installed | `pg-siteA`, `pg-siteB` | `rpm -q timescaledb-2-postgresql-16` | Package version returned | [ ] |
| 3 | Patroni installed | `pg-siteA`, `pg-siteB` | `patroni --version` | Version returned | [ ] |
| 4 | pgBackRest installed | `pg-siteA`, `pg-siteB` | `rpm -q pgbackrest` | Package version returned | [ ] |
| 5 | PgBouncer installed | `pg-siteA`, `pg-siteB` | `rpm -q pgbouncer` | Package version returned | [ ] |
| 6 | Zabbix Server packages | `zabbix-node-siteA/B` | `rpm -q zabbix-server-pgsql` | Package version returned | [ ] |
| 7 | Zabbix Frontend packages | `frontend-siteA/B` | `rpm -q zabbix-web-pgsql zabbix-nginx-conf` | Both packages returned | [ ] |
| 8 | Zabbix Proxy packages | All proxy VMs | `rpm -q zabbix-proxy-sqlite3` or `zabbix-proxy-pgsql` | Package version returned | [ ] |
| 9 | Zabbix Agent 2 packages | All Zabbix VMs | `rpm -q zabbix-agent2` | Package version returned | [ ] |
| 10 | PostgreSQL schema imported | `pg-siteA` | `sudo -u postgres psql zabbix -c "\dt" \| wc -l` | 100+ tables | [ ] |
| 11 | TimescaleDB extension active | `pg-siteA` | `sudo -u postgres psql zabbix -c "SELECT extname FROM pg_extension WHERE extname='timescaledb';"` | `timescaledb` returned | [ ] |
| 12 | TimescaleDB hypertables created | `pg-siteA` | `sudo -u postgres psql zabbix -c "SELECT count(*) FROM timescaledb_information.hypertables;"` | Count > 0 | [ ] |
| 13 | PostgreSQL stopped and disabled | `pg-siteA` | `systemctl is-active postgresql-16 && systemctl is-enabled postgresql-16` | `inactive` and `disabled` | [ ] |
| 14 | SELinux booleans set | Frontend VMs | `getsebool httpd_can_connect_zabbix` | `on` | [ ] |
| 15 | Firewalld ports open — Database | `pg-siteA` | `firewall-cmd --list-ports` | `5432/tcp 6432/tcp 8008/tcp 10050/tcp` | [ ] |
| 16 | Firewalld ports open — etcd | `etcd-1` | `firewall-cmd --list-ports` | `2379/tcp 2380/tcp 10050/tcp` | [ ] |
| 17 | Firewalld ports open — Server | `zabbix-node-siteA` | `firewall-cmd --list-ports` | `10051/tcp 10050/tcp` | [ ] |
| 18 | Firewalld ports open — Frontend | `frontend-siteA` | `firewall-cmd --list-ports` | `80/tcp 443/tcp 10050/tcp` | [ ] |
| 19 | Firewalld ports open — Proxy | `proxy-siteA-01` | `firewall-cmd --list-ports` | `10051/tcp 10050/tcp` | [ ] |
| 20 | Firewalld ports open — SNMP Trap | `proxy-siteA-snmptrap` | `firewall-cmd --list-ports` | `162/udp 10051/tcp 10050/tcp` | [ ] |

---

## Troubleshooting

### Repository Conflicts

**Symptom:** `dnf install` fails with package conflict errors or dependency resolution failures.

**Resolution:**
1. Check for conflicting repositories:
   ```bash
   dnf repolist
   ```
2. If AppStream provides a conflicting `postgresql` module:
   ```bash
   dnf -qy module disable postgresql
   ```
3. If multiple Zabbix repositories are enabled:
   ```bash
   dnf repolist | grep -i zabbix
   # Remove duplicates
   rpm -qa | grep zabbix-release
   ```

### EPEL Zabbix Package Conflict

**Symptom:** `dnf install zabbix-server-pgsql` installs an old EPEL version instead of the official Zabbix 7.0 package.

**Resolution:**
1. Add the EPEL exclusion:
   ```bash
   sed -i '/^\[epel\]/a excludepkgs=zabbix*' /etc/yum.repos.d/epel.repo
   ```
2. Clean the cache and retry:
   ```bash
   dnf clean all
   dnf install -y zabbix-server-pgsql
   ```
3. Verify the correct version:
   ```bash
   rpm -qi zabbix-server-pgsql | grep -E "Version|Release|Vendor"
   # Vendor should be "Zabbix LLC"
   ```

### TimescaleDB Version Mismatch

**Symptom:** `CREATE EXTENSION timescaledb` fails with an error about incompatible PostgreSQL version.

**Resolution:**
1. Verify you installed the correct TimescaleDB package for PostgreSQL 16:
   ```bash
   rpm -qa | grep timescaledb
   # Should show: timescaledb-2-postgresql-16-*
   ```
2. If the wrong version is installed:
   ```bash
   dnf remove timescaledb-2-postgresql-15  # Remove wrong version
   dnf install -y timescaledb-2-postgresql-16
   ```
3. Ensure `shared_preload_libraries` includes `timescaledb` in `postgresql.conf`:
   ```bash
   grep shared_preload_libraries /var/lib/pgsql/16/data/postgresql.conf
   ```

### Schema Import Errors

**Symptom:** `zcat server.sql.gz | psql` produces errors or the zabbix database has missing tables.

**Resolution:**
1. Verify the schema file exists:
   ```bash
   ls -la /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz
   ```
2. If the file is missing, the `zabbix-sql-scripts` package may not be installed:
   ```bash
   dnf install -y zabbix-sql-scripts
   ```
3. If the import partially completed, drop and recreate the database:
   ```bash
   sudo -u postgres dropdb zabbix
   sudo -u postgres createdb -O zabbix -E Unicode -T template0 zabbix
   zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix
   ```
4. Check for disk space issues:
   ```bash
   df -h /var/lib/pgsql
   ```

### PostgreSQL Won't Stop

**Symptom:** `systemctl stop postgresql-16` hangs or fails.

**Resolution:**
1. Check for active connections:
   ```bash
   sudo -u postgres psql -c "SELECT pid, usename, application_name, state FROM pg_stat_activity;"
   ```
2. Use fast shutdown:
   ```bash
   sudo -u postgres /usr/pgsql-16/bin/pg_ctl stop -D /var/lib/pgsql/16/data -m fast
   ```
3. Verify it stopped:
   ```bash
   systemctl status postgresql-16
   ```
