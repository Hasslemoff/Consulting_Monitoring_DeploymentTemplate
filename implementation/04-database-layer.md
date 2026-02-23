# Phase 4 — Database Layer (Patroni + PostgreSQL + PgBouncer)

## Objective

Configure Patroni-managed PostgreSQL HA with streaming replication across two sites, deploy PgBouncer for connection pooling, tune TimescaleDB chunk intervals, and enable DCS failsafe mode. After this phase, the database layer will automatically fail over if the primary PostgreSQL node becomes unavailable.

## Prerequisites

- [ ] Phase 3 complete — etcd cluster healthy with 3 members, auth enabled, patroni user verified
- [ ] Database schema already imported on `pg-siteA` (Phase 2, Step 8)
- [ ] `postgresql-16` service is **stopped and disabled** on `pg-siteA` (Phase 2, Step 8h)
- [ ] `pg-siteB` has PostgreSQL 16 installed but **not initialized** (no initdb, no data directory)
- [ ] etcd CA certificate available for copy to Patroni nodes
- [ ] Vault credentials ready: `{{DB_ZABBIX_PASSWORD}}`, `{{DB_POSTGRES_PASSWORD}}`, `{{DB_REPLICATOR_PASSWORD}}`, `{{ETCD_PATRONI_PASSWORD}}`, `{{PGBOUNCER_MD5_HASH}}`

---

## Step 1 — Measure Inter-Site RTT and Select Replication Mode

If not already measured in Phase 1, run from `pg-siteA`:

```bash
ping -c 50 {{PG_B_IP}}
```

Record the **average RTT** and compare against the NVPS ceiling table to choose `{{REPLICATION_MODE}}`:

| Inter-Site RTT | Sync Mode NVPS Ceiling | Recommendation |
|----------------|------------------------|----------------|
| < 2 ms (same campus) | 50K+ | **Synchronous** — negligible penalty |
| 2-5 ms (metro area) | 15-25K | **Synchronous** — acceptable for most deployments |
| 5-10 ms | 8-15K | **Evaluate** — sync if `{{EXPECTED_NVPS}}` is below ceiling |
| 10-20 ms | 4-8K | **Asynchronous recommended** — sync caps even small deployments |
| 20 ms+ | < 4K | **Asynchronous required** — sync is not viable |

**Decision:** Set `{{REPLICATION_MODE}}` to `synchronous` or `asynchronous` in your environment variables. The Patroni configuration in Steps 3-4 uses this value.

> **Guideline:** If your `{{EXPECTED_NVPS}}` is within 50% of the ceiling for your RTT, choose **asynchronous** to leave headroom for growth. For Zabbix monitoring data, asynchronous replication risks losing at most a few seconds of metric samples during failover — generally acceptable.

---

## Step 2 — Copy etcd CA Certificate to Patroni Nodes

Patroni connects to etcd over TLS and needs the etcd CA certificate to verify the connection. Copy it to both database VMs.

```bash
# From the certificate generation workstation or etcd-1:
scp /etc/etcd/pki/ca.crt root@{{PG_A_IP}}:/etc/patroni/pki/ca.crt
scp /etc/etcd/pki/ca.crt root@{{PG_B_IP}}:/etc/patroni/pki/ca.crt
```

Set ownership on both database VMs:

```bash
chown postgres:postgres /etc/patroni/pki/ca.crt
chmod 644 /etc/patroni/pki/ca.crt
```

---

## Step 3 — Configure Patroni on Site A Primary

Create `/etc/patroni/patroni.yml` on `pg-siteA`:

```yaml
scope: zabbix-cluster
name: pg-siteA

restapi:
  listen: 0.0.0.0:8008
  connect_address: {{PG_A_IP}}:8008
  authentication:
    username: patroni_api
    password: '{{PATRONI_API_PASSWORD}}'

etcd3:
  hosts: {{ETCD_1_IP}}:2379,{{ETCD_2_IP}}:2379,{{ETCD_3_IP}}:2379
  protocol: https
  cacert: /etc/patroni/pki/ca.crt
  username: patroni
  password: '{{ETCD_PATRONI_PASSWORD}}'

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    synchronous_mode: true          # Set to false if {{REPLICATION_MODE}} is asynchronous
    synchronous_mode_strict: false   # Allow leader to operate alone if replica is down
    postgresql:
      use_pg_rewind: true
      parameters:
        # Connection Settings
        max_connections: 500
        # Memory
        shared_buffers: 8GB
        work_mem: 64MB
        maintenance_work_mem: 1GB
        effective_cache_size: 24GB
        # WAL / Replication
        wal_level: replica
        max_wal_senders: 5
        max_replication_slots: 5
        wal_keep_size: 2GB
        # TimescaleDB
        shared_preload_libraries: timescaledb
        # SSL
        ssl: on
        ssl_cert_file: /etc/patroni/pki/server.crt
        ssl_key_file: /etc/patroni/pki/server.key
        ssl_ca_file: /etc/patroni/pki/ca.crt
        # Logging
        log_min_duration_statement: 1000
        log_checkpoints: 'on'
        log_lock_waits: 'on'
      pg_hba:
        - local   all             postgres                          peer
        - local   all             all                               scram-sha-256
        - host    all             zabbix      {{SITE_A_SUBNET}}     scram-sha-256
        - host    all             zabbix      {{SITE_B_SUBNET}}     scram-sha-256
        - host    all             zabbix      127.0.0.1/32          scram-sha-256
        - hostssl replication     replicator  {{PG_A_IP}}/32        scram-sha-256
        - hostssl replication     replicator  {{PG_B_IP}}/32        scram-sha-256
        - host    all             all         127.0.0.1/32          scram-sha-256

postgresql:
  listen: 0.0.0.0:5432
  connect_address: {{PG_A_IP}}:5432
  bin_dir: /usr/pgsql-16/bin
  data_dir: /var/lib/pgsql/16/data
  authentication:
    replication:
      username: replicator
      password: '{{DB_REPLICATOR_PASSWORD}}'
    superuser:
      username: postgres
      password: '{{DB_POSTGRES_PASSWORD}}'
  parameters:
    unix_socket_directories: '/var/run/postgresql'
```

Set ownership and permissions:

```bash
chown postgres:postgres /etc/patroni/patroni.yml
chmod 600 /etc/patroni/patroni.yml
```

> **Important:** `synchronous_mode: true` in the `bootstrap.dcs` section corresponds to `{{REPLICATION_MODE}}` = `synchronous`. If you chose asynchronous, set this to `false`. The `synchronous_mode_strict: false` setting allows the primary to continue accepting writes if the replica goes down — this prevents a total outage when one site is lost.

---

## Step 4 — Configure Patroni on Site B Replica

Create `/etc/patroni/patroni.yml` on `pg-siteB`:

```yaml
scope: zabbix-cluster
name: pg-siteB

restapi:
  listen: 0.0.0.0:8008
  connect_address: {{PG_B_IP}}:8008
  authentication:
    username: patroni_api
    password: '{{PATRONI_API_PASSWORD}}'

etcd3:
  hosts: {{ETCD_1_IP}}:2379,{{ETCD_2_IP}}:2379,{{ETCD_3_IP}}:2379
  protocol: https
  cacert: /etc/patroni/pki/ca.crt
  username: patroni
  password: '{{ETCD_PATRONI_PASSWORD}}'

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    synchronous_mode: true          # Must match Site A setting
    synchronous_mode_strict: false
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 500
        shared_buffers: 8GB
        work_mem: 64MB
        maintenance_work_mem: 1GB
        effective_cache_size: 24GB
        wal_level: replica
        max_wal_senders: 5
        max_replication_slots: 5
        wal_keep_size: 2GB
        shared_preload_libraries: timescaledb
        ssl: on
        ssl_cert_file: /etc/patroni/pki/server.crt
        ssl_key_file: /etc/patroni/pki/server.key
        ssl_ca_file: /etc/patroni/pki/ca.crt
        log_min_duration_statement: 1000
        log_checkpoints: 'on'
        log_lock_waits: 'on'
      pg_hba:
        - local   all             postgres                          peer
        - local   all             all                               scram-sha-256
        - host    all             zabbix      {{SITE_A_SUBNET}}     scram-sha-256
        - host    all             zabbix      {{SITE_B_SUBNET}}     scram-sha-256
        - host    all             zabbix      127.0.0.1/32          scram-sha-256
        - hostssl replication     replicator  {{PG_A_IP}}/32        scram-sha-256
        - hostssl replication     replicator  {{PG_B_IP}}/32        scram-sha-256
        - host    all             all         127.0.0.1/32          scram-sha-256

postgresql:
  listen: 0.0.0.0:5432
  connect_address: {{PG_B_IP}}:5432
  bin_dir: /usr/pgsql-16/bin
  data_dir: /var/lib/pgsql/16/data
  authentication:
    replication:
      username: replicator
      password: '{{DB_REPLICATOR_PASSWORD}}'
    superuser:
      username: postgres
      password: '{{DB_POSTGRES_PASSWORD}}'
  parameters:
    unix_socket_directories: '/var/run/postgresql'
```

Set ownership and permissions:

```bash
chown postgres:postgres /etc/patroni/patroni.yml
chmod 600 /etc/patroni/patroni.yml
```

---

## Step 5 — Create Patroni systemd Unit

Create `/etc/systemd/system/patroni.service` on **both** `pg-siteA` and `pg-siteB`:

```ini
[Unit]
Description=Patroni - PostgreSQL HA Cluster Manager
Documentation=https://patroni.readthedocs.io
After=network-online.target etcd.service
Wants=network-online.target

[Service]
User=postgres
Group=postgres
Type=simple
ExecStart=/usr/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=10s
TimeoutSec=30
LimitNOFILE=65536
LimitNPROC=65536

[Install]
WantedBy=multi-user.target
```

Reload systemd on both nodes:

```bash
chmod 644 /etc/systemd/system/patroni.service
systemctl daemon-reload
systemctl enable patroni
```

---

## Step 6 — Bootstrap Patroni

> **CRITICAL ORDER:** Start Patroni on Site A **first**. It will detect the existing PostgreSQL data directory (from Phase 2 schema import) and initialize as the primary. Only start Site B after Site A is confirmed healthy.

### 6a. Start Patroni on Site A

```bash
# On pg-siteA
systemctl start patroni
```

Watch the logs to confirm it starts as the leader:

```bash
journalctl -u patroni -f
```

**Expected log messages:**
- `INFO: no action. I am (pg-siteA), the leader with the lock`
- `INFO: Lock owner: pg-siteA; I am pg-siteA`

Wait until Patroni reports it is the leader and PostgreSQL is running before proceeding.

### 6b. Verify Site A is Healthy

```bash
patronictl -c /etc/patroni/patroni.yml list
```

**Expected output:**

```
+----------------+------------+---------+---------+----+-----------+
|    Cluster     |   Member   |  Host   |  Role   | TL | Lag in MB |
+----------------+------------+---------+---------+----+-----------+
| zabbix-cluster | pg-siteA   | <IP>    | Leader  |  1 |           |
+----------------+------------+---------+---------+----+-----------+
```

### 6c. Start Patroni on Site B

```bash
# On pg-siteB
systemctl start patroni
```

Patroni on Site B will automatically:
1. Connect to etcd and discover the existing `zabbix-cluster`
2. Use `pg_basebackup` to create a full copy of the primary
3. Start as a streaming replica

Watch the progress:

```bash
journalctl -u patroni -f
```

**Expected log messages:**
- `INFO: trying to bootstrap from leader 'pg-siteA'`
- `INFO: replica has been created`
- `INFO: no action. I am (pg-siteB), a secondary, and following a leader (pg-siteA)`

> **Note:** The initial `pg_basebackup` can take 10-60 minutes depending on database size and network speed. Do not interrupt it.

---

## Step 7 — Verify the Patroni Cluster

```bash
patronictl -c /etc/patroni/patroni.yml list
```

**Expected output for synchronous mode:**

```
+----------------+------------+---------+----------------+----+-----------+
|    Cluster     |   Member   |  Host   |      Role      | TL | Lag in MB |
+----------------+------------+---------+----------------+----+-----------+
| zabbix-cluster | pg-siteA   | <IP>    | Leader         |  1 |           |
| zabbix-cluster | pg-siteB   | <IP>    | Sync Standby   |  1 |         0 |
+----------------+------------+---------+----------------+----+-----------+
```

**Expected output for asynchronous mode:**

```
+----------------+------------+---------+----------------+----+-----------+
|    Cluster     |   Member   |  Host   |      Role      | TL | Lag in MB |
+----------------+------------+---------+----------------+----+-----------+
| zabbix-cluster | pg-siteA   | <IP>    | Leader         |  1 |           |
| zabbix-cluster | pg-siteB   | <IP>    | Replica        |  1 |         0 |
+----------------+------------+---------+----------------+----+-----------+
```

Verify replication is working:

```bash
# On the leader (pg-siteA), check replication status
sudo -u postgres psql -c "SELECT client_addr, state, sync_state, sent_lsn, write_lsn, flush_lsn, replay_lsn FROM pg_stat_replication;"
```

**Expected:** One row showing `state: streaming` and `sync_state: sync` (or `async`).

Verify the Patroni REST API is accessible (KEMP will use this):

```bash
# From pg-siteA — should return 200 with role info
curl -s http://{{PG_A_IP}}:8008/patroni | python3 -m json.tool

# Check primary status (returns 200 on primary, 503 on replica)
curl -s -o /dev/null -w "%{http_code}" http://{{PG_A_IP}}:8008/primary
# Expected: 200

curl -s -o /dev/null -w "%{http_code}" http://{{PG_B_IP}}:8008/primary
# Expected: 503
```

> **Security hardening:** The Patroni REST API is shown using HTTP for initial setup. For production, enable HTTPS by adding `certfile`, `keyfile`, and `cafile` to the `restapi` section of `patroni.yml`. Update KEMP health checks (Phase 05) to use HTTPS on port 8008 accordingly.

---

## Step 8 — TimescaleDB Chunk Tuning

Connect to the primary database and configure chunk intervals for trend tables. This improves trend update performance on high-item-count deployments.

```bash
sudo -u postgres psql zabbix << 'SQL'
-- Set 7-day chunk interval for trend tables (default may be 1 day)
SELECT set_chunk_time_interval('trends', INTERVAL '7 days');
SELECT set_chunk_time_interval('trends_uint', INTERVAL '7 days');

-- Verify the settings
SELECT hypertable_name, column_name,
       (SELECT value FROM timescaledb_information.dimensions d
        WHERE d.hypertable_name = h.hypertable_name
        AND d.column_name = h.column_name)
FROM timescaledb_information.dimensions h
WHERE hypertable_name IN ('trends', 'trends_uint');

-- Verify all Zabbix hypertables
SELECT hypertable_name, num_chunks, compression_enabled
FROM timescaledb_information.hypertables
ORDER BY hypertable_name;
SQL
```

---

## Step 9 — Configure PgBouncer

Configure PgBouncer on **both** `pg-siteA` and `pg-siteB` to provide connection pooling between Zabbix components and PostgreSQL.

### 9a. Create PgBouncer Configuration

Edit `/etc/pgbouncer/pgbouncer.ini` on both database VMs:

```ini
;;;
;;; PgBouncer configuration for Zabbix
;;;

[databases]
zabbix = host=127.0.0.1 port=5432 dbname=zabbix

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

;; Pool mode: transaction is required for Zabbix 7.0+
pool_mode = transaction
max_client_conn = 500
default_pool_size = 50
reserve_pool_size = 10
reserve_pool_timeout = 3

;; Required for transaction mode
server_reset_query = DISCARD ALL

;; Timeouts
server_connect_timeout = 15
server_idle_timeout = 600
client_idle_timeout = 0
client_login_timeout = 60

;; Logging (minimize noise in production)
log_connections = 0
log_disconnections = 0
log_pooler_errors = 1

;; Admin access (for monitoring PgBouncer itself)
admin_users = pgbouncer_admin
stats_users = postgres,zabbix

;; Paths
pidfile = /var/run/pgbouncer/pgbouncer.pid
logfile = /var/log/pgbouncer/pgbouncer.log
unix_socket_dir = /var/run/pgbouncer
```

> **Note:** PgBouncer 1.21+ is required for SCRAM-SHA-256 authentication support. Verify your installed version with `pgbouncer --version`.

### 9b. Create PgBouncer Auth File

Generate the MD5 hash for the zabbix user. The hash format is `md5` + MD5(`password` + `username`):

```bash
# Generate the hash (replace <password> with {{DB_ZABBIX_PASSWORD}})
echo -n "md5$(echo -n '{{DB_ZABBIX_PASSWORD}}zabbix' | md5sum | cut -d' ' -f1)"
```

Write the auth file `/etc/pgbouncer/userlist.txt` on both database VMs:

```bash
cat > /etc/pgbouncer/userlist.txt << 'EOF'
"zabbix" "{{PGBOUNCER_MD5_HASH}}"
EOF

chown pgbouncer:pgbouncer /etc/pgbouncer/userlist.txt
chmod 640 /etc/pgbouncer/userlist.txt
```

> **Note:** `{{PGBOUNCER_MD5_HASH}}` is the full hash string including the `md5` prefix (e.g., `md5a1b2c3d4e5f6...`).

### 9c. Set PgBouncer File Permissions

```bash
chown pgbouncer:pgbouncer /etc/pgbouncer/pgbouncer.ini
chmod 640 /etc/pgbouncer/pgbouncer.ini

# Ensure log directory exists
mkdir -p /var/log/pgbouncer
chown pgbouncer:pgbouncer /var/log/pgbouncer

# Ensure PID directory exists
mkdir -p /var/run/pgbouncer
chown pgbouncer:pgbouncer /var/run/pgbouncer
```

---

## Step 10 — Start and Enable PgBouncer

Run on **both** `pg-siteA` and `pg-siteB`:

```bash
systemctl enable --now pgbouncer
```

Verify PgBouncer is running and accepting connections:

```bash
# Check service status
systemctl status pgbouncer

# Test connection through PgBouncer
psql -h 127.0.0.1 -p 6432 -U zabbix -d zabbix -c "SELECT version();"
# Enter {{DB_ZABBIX_PASSWORD}} when prompted

# Check PgBouncer stats
psql -h 127.0.0.1 -p 6432 -U postgres -d pgbouncer -c "SHOW pools;"
# Enter {{DB_POSTGRES_PASSWORD}} when prompted
```

> **Note:** PgBouncer on the **replica** node (pg-siteB) will only serve read-only queries. During a Patroni failover, the new primary's PgBouncer will automatically start serving read-write traffic because the local PostgreSQL instance becomes the primary.

---

## Step 11 — Enable DCS Failsafe Mode

Failsafe mode prevents Patroni from demoting the primary if etcd becomes unavailable. Without failsafe mode, losing the etcd cluster would cause Patroni to shut down PostgreSQL — taking down the entire monitoring platform.

```bash
patronictl -c /etc/patroni/patroni.yml edit-config -s failsafe_mode=true
```

Verify the setting:

```bash
patronictl -c /etc/patroni/patroni.yml show-config | grep failsafe
```

**Expected output:** `failsafe_mode: true`

> **How failsafe works:** If etcd becomes unreachable, the primary keeps running instead of shutting down. The standby also keeps running as a replica. No automatic failover can occur while etcd is down (because there is no consensus mechanism), but the existing primary/replica relationship is preserved. Once etcd recovers, Patroni resumes normal operation.

---

## Verification Checkpoint

Complete all checks before proceeding to Phase 5.

| # | Check | Command | Expected Result | Pass |
|---|-------|---------|-----------------|------|
| 1 | Patroni cluster healthy | `patronictl -c /etc/patroni/patroni.yml list` | 2 members: 1 Leader + 1 Sync Standby (or Replica) | [ ] |
| 2 | pg-siteA is Leader | `patronictl -c /etc/patroni/patroni.yml list` | pg-siteA shows `Leader` role | [ ] |
| 3 | pg-siteB is Standby | `patronictl -c /etc/patroni/patroni.yml list` | pg-siteB shows `Sync Standby` or `Replica` | [ ] |
| 4 | Replication lag < 1 MB | `patronictl -c /etc/patroni/patroni.yml list` | `Lag in MB` column shows 0 | [ ] |
| 5 | Streaming replication active | `sudo -u postgres psql -c "SELECT state, sync_state FROM pg_stat_replication;"` | `streaming` / `sync` or `async` | [ ] |
| 6 | Patroni REST API — primary returns 200 | `curl -s -o /dev/null -w "%{http_code}" http://{{PG_A_IP}}:8008/primary` | `200` | [ ] |
| 7 | Patroni REST API — replica returns 503 | `curl -s -o /dev/null -w "%{http_code}" http://{{PG_B_IP}}:8008/primary` | `503` | [ ] |
| 8 | PgBouncer running — Site A | `systemctl is-active pgbouncer` (on pg-siteA) | `active` | [ ] |
| 9 | PgBouncer running — Site B | `systemctl is-active pgbouncer` (on pg-siteB) | `active` | [ ] |
| 10 | PgBouncer connection — Site A | `psql -h 127.0.0.1 -p 6432 -U zabbix -d zabbix -c "SELECT 1;"` | Returns `1` | [ ] |
| 11 | TimescaleDB extension active | `sudo -u postgres psql zabbix -c "SELECT extname, extversion FROM pg_extension WHERE extname='timescaledb';"` | `timescaledb` with version | [ ] |
| 12 | TimescaleDB hypertables present | `sudo -u postgres psql zabbix -c "SELECT count(*) FROM timescaledb_information.hypertables;"` | Count > 0 | [ ] |
| 13 | Failsafe mode enabled | `patronictl -c /etc/patroni/patroni.yml show-config \| grep failsafe` | `failsafe_mode: true` | [ ] |
| 14 | Replication mode matches design | `patronictl -c /etc/patroni/patroni.yml show-config \| grep synchronous_mode` | Matches `{{REPLICATION_MODE}}` | [ ] |

---

## Troubleshooting

### Patroni Bootstrap Fails on Site A

**Symptom:** `systemctl start patroni` fails or logs show `could not connect to etcd` or `failed to initialize`.

**Resolution:**
1. Verify etcd is reachable:
   ```bash
   curl -s --cacert /etc/patroni/pki/ca.crt \
     https://{{ETCD_1_IP}}:2379/health
   ```
2. Verify the etcd CA cert is correct:
   ```bash
   md5sum /etc/patroni/pki/ca.crt
   md5sum /etc/etcd/pki/ca.crt  # Should match (on an etcd node)
   ```
3. Verify the patroni etcd credentials:
   ```bash
   etcdctl --endpoints=https://{{ETCD_1_IP}}:2379 \
     --cacert=/etc/etcd/pki/ca.crt \
     --user patroni:{{ETCD_PATRONI_PASSWORD}} \
     get /service/ --prefix --keys-only
   ```
4. Check that the PostgreSQL data directory exists and contains data:
   ```bash
   ls -la /var/lib/pgsql/16/data/
   # Should contain PG_VERSION, postgresql.conf, base/, etc.
   ```
5. Verify ownership:
   ```bash
   ls -la /var/lib/pgsql/16/
   # data/ should be owned by postgres:postgres
   ```
6. Check Patroni config syntax:
   ```bash
   python3 -c "import yaml; yaml.safe_load(open('/etc/patroni/patroni.yml'))"
   ```
7. Check detailed logs:
   ```bash
   journalctl -u patroni --no-pager -n 100
   ```

### Replica Won't Sync (Site B Bootstrap Fails)

**Symptom:** Patroni on Site B logs `pg_basebackup` errors, or the replica never appears in `patronictl list`.

**Resolution:**
1. Verify the primary is accepting replication connections:
   ```bash
   # On pg-siteA
   sudo -u postgres psql -c "SELECT * FROM pg_hba_file_rules WHERE database = '{replication}';"
   ```
2. Verify the replicator user exists and has the correct password:
   ```bash
   # On pg-siteA
   sudo -u postgres psql -c "SELECT usename, usesuper, userepl FROM pg_user WHERE usename='replicator';"
   ```
   If the replicator user does not exist, Patroni creates it automatically from the config. Check if the password in `patroni.yml` is correct.
3. Verify network connectivity from Site B to Site A on port 5432:
   ```bash
   nc -zv {{PG_A_IP}} 5432
   ```
4. If `pg_basebackup` failed partway through, clean up and retry:
   ```bash
   # On pg-siteB
   systemctl stop patroni
   rm -rf /var/lib/pgsql/16/data/*
   systemctl start patroni
   ```
5. Check for WAL segment issues on the primary:
   ```bash
   # On pg-siteA
   sudo -u postgres psql -c "SELECT slot_name, active, restart_lsn FROM pg_replication_slots;"
   ```

### PgBouncer Auth Errors

**Symptom:** `psql -h 127.0.0.1 -p 6432 -U zabbix -d zabbix` returns `ERROR: password authentication failed`.

**Resolution:**
1. Verify the MD5 hash in `/etc/pgbouncer/userlist.txt`:
   ```bash
   cat /etc/pgbouncer/userlist.txt
   ```
2. Regenerate the hash and compare:
   ```bash
   echo -n "md5$(echo -n '{{DB_ZABBIX_PASSWORD}}zabbix' | md5sum | cut -d' ' -f1)"
   # This output should match the hash in userlist.txt
   ```
3. Verify the same password works directly against PostgreSQL:
   ```bash
   psql -h 127.0.0.1 -p 5432 -U zabbix -d zabbix -c "SELECT 1;"
   ```
4. Verify `auth_type` in `pgbouncer.ini` is `md5` (not `scram-sha-256`):
   ```bash
   grep auth_type /etc/pgbouncer/pgbouncer.ini
   ```
5. If PostgreSQL uses SCRAM authentication, you need to set `auth_type = scram-sha-256` in PgBouncer and use the `auth_query` method instead of `auth_file`. Consult the PgBouncer documentation for SCRAM setup.
6. After making changes, restart PgBouncer:
   ```bash
   systemctl restart pgbouncer
   ```

### Synchronous Replication Timeout

**Symptom:** Write operations on the primary hang or time out. `patronictl list` shows the replica with high lag or the replica is `Replica` instead of `Sync Standby`.

**Resolution:**
1. Check the replica status:
   ```bash
   sudo -u postgres psql -c "SELECT client_addr, state, sync_state, sent_lsn, write_lsn, flush_lsn, replay_lsn FROM pg_stat_replication;"
   ```
2. If `sync_state` is `async` instead of `sync`, the replica may have fallen behind. Check for network issues:
   ```bash
   ping -c 10 {{PG_B_IP}}
   ```
3. If the inter-site link is degraded, temporarily switch to asynchronous mode:
   ```bash
   patronictl -c /etc/patroni/patroni.yml edit-config -s synchronous_mode=false
   ```
4. Monitor the switch:
   ```bash
   patronictl -c /etc/patroni/patroni.yml list
   # Role should change from "Sync Standby" to "Replica"
   ```
5. When the link recovers, switch back to synchronous:
   ```bash
   patronictl -c /etc/patroni/patroni.yml edit-config -s synchronous_mode=true
   ```
6. If the problem persists and your NVPS is close to the ceiling for your RTT, you may need to permanently use asynchronous mode. Refer to the NVPS ceiling table in Step 1.

### Patroni Failover Testing

After all checks pass, optionally test a controlled failover:

```bash
# Switchover (planned, zero-downtime for sync mode)
patronictl -c /etc/patroni/patroni.yml switchover

# Follow the prompts:
#   Master: pg-siteA
#   Candidate: pg-siteB
#   When: now

# Verify the new state
patronictl -c /etc/patroni/patroni.yml list
# pg-siteB should now be Leader, pg-siteA should be Sync Standby/Replica

# Switch back when ready
patronictl -c /etc/patroni/patroni.yml switchover
```

> **WARNING:** Only test failover during a maintenance window. In synchronous mode, the switchover is near-instantaneous. In asynchronous mode, there may be a brief period (< 1 second) where recent writes are not yet replicated.
