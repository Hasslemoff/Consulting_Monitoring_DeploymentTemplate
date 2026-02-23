# Phase 6 — Zabbix Server HA Cluster

> **Objective:** Configure and start the Zabbix Server native HA cluster across both sites, with one active node at Site A and one standby node at Site B.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Phase 5 complete | KEMP HA pairs operational at both sites |
| Database VIP routing | `{{VIP_DB_A}}` and `{{VIP_DB_B}}` routing to Patroni primary (verify with `psql -h {{VIP_DB_A}} -c "SELECT pg_is_in_recovery();"` returns `f`) |
| Zabbix Server installed | `zabbix-server-pgsql` package installed on both server VMs |
| Database initialized | Zabbix schema imported into `zabbix` database on PostgreSQL primary |
| Firewall rules | Port 10051/TCP open between both Zabbix servers and all proxies/agents |

---

## Step 1 — Configure Node 1 at Site A

Edit the Zabbix Server configuration file:

```bash
sudo cp /etc/zabbix/zabbix_server.conf /etc/zabbix/zabbix_server.conf.orig
sudo vi /etc/zabbix/zabbix_server.conf
```

Set the following parameters in `/etc/zabbix/zabbix_server.conf` on `{{ZBX_SERVER_A_IP}}`:

```ini
### HA Configuration ###
HANodeName=zabbix-node-siteA
NodeAddress={{ZBX_SERVER_A_IP}}:10051

### Database Connection ###
DBHost={{VIP_DB_A}}
DBPort=5432
DBName=zabbix
DBUser=zabbix
DBPassword={{DB_ZABBIX_PASSWORD}}

### General ###
ListenPort=10051
LogFile=/var/log/zabbix/zabbix_server.log
LogFileSize=100
PidFile=/run/zabbix/zabbix_server.pid
SocketDir=/run/zabbix
```

> **Note on DBHost:** Each node connects to its **local site** KEMP Database VIP. This ensures the shortest network path for database queries. Both VIPs route to the same Patroni primary, so data consistency is guaranteed.

> **Note on DBPort:** If PgBouncer is deployed in front of PostgreSQL (configured in Phase 4), set `DBPort=6432` instead of `5432`.

---

## Step 2 — Configure Node 2 at Site B

Edit `/etc/zabbix/zabbix_server.conf` on `{{ZBX_SERVER_B_IP}}`:

```ini
### HA Configuration ###
HANodeName=zabbix-node-siteB
NodeAddress={{ZBX_SERVER_B_IP}}:10051

### Database Connection ###
DBHost={{VIP_DB_B}}
DBPort=5432
DBName=zabbix
DBUser=zabbix
DBPassword={{DB_ZABBIX_PASSWORD}}

### General ###
ListenPort=10051
LogFile=/var/log/zabbix/zabbix_server.log
LogFileSize=100
PidFile=/run/zabbix/zabbix_server.pid
SocketDir=/run/zabbix
```

> **Critical:** The `HANodeName` must be unique across all nodes. If two nodes register with the same name, the second node will fail to start and log a duplicate node error.

---

## Step 3 — Performance Tuning

Add the performance tuning parameters to **both** nodes' configuration files. Select the block that matches your `{{SIZING_TIER}}`.

### Tuning Reference Table

| Parameter | Small (<500 hosts) | Medium (500-2,000 hosts) | Large (2,000-10,000 hosts) | Very Large (10,000+ hosts) |
|-----------|---------------------|--------------------------|----------------------------|----------------------------|
| `StartPollers` | 10 | 50 | 100 | 150 |
| `StartPollersUnreachable` | 3 | 10 | 25 | 50 |
| `StartTrappers` | 5 | 10 | 20 | 30 |
| `StartPingers` | 2 | 5 | 10 | 15 |
| `StartDiscoverers` | 2 | 5 | 10 | 15 |
| `StartHTTPPollers` | 2 | 5 | 10 | 15 |
| `StartPreprocessors` | 5 | 15 | 30 | 50 |
| `StartHistoryPollers` | 5 | 10 | 25 | 50 |
| `CacheSize` | 64M | 256M | 512M | 1G |
| `ValueCacheSize` | 64M | 256M | 512M | 1G |
| `HistoryCacheSize` | 32M | 64M | 128M | 256M |
| `HistoryIndexCacheSize` | 16M | 32M | 64M | 128M |
| `TrendCacheSize` | 16M | 32M | 64M | 128M |
| `TrendFunctionCacheSize` | 16M | 32M | 64M | 128M |
| `Timeout` | 10 | 15 | 20 | 30 |
| `HousekeepingFrequency` | 1 | 1 | 1 | 1 |
| `MaxHousekeeperDelete` | 5000 | 10000 | 50000 | 100000 |
| `StartAlerters` | 3 | 5 | 10 | 15 |
| `StartLLDProcessors` | 2 | 5 | 10 | 15 |

### Medium Tier Configuration Block (Default)

Append the following to `/etc/zabbix/zabbix_server.conf` on **both** nodes:

```ini
### Performance Tuning — Medium Tier ({{SIZING_TIER}}) ###
### Estimated: {{EXPECTED_HOSTS}} hosts, {{EXPECTED_NVPS}} NVPS ###

# Pollers
StartPollers=50
StartPollersUnreachable=10
StartTrappers=10
StartPingers=5
StartDiscoverers=5
StartHTTPPollers=5

# Preprocessing
StartPreprocessors=15

# History
StartHistoryPollers=10

# Cache Sizes
CacheSize=256M
ValueCacheSize=256M
HistoryCacheSize=64M
HistoryIndexCacheSize=32M
TrendCacheSize=32M
TrendFunctionCacheSize=32M

# Timeouts
Timeout=15

# Housekeeping
HousekeepingFrequency=1
MaxHousekeeperDelete=10000

# Alerting
StartAlerters=5

# LLD
StartLLDProcessors=5
```

> **Important:** If using a different sizing tier, replace the values with the corresponding column from the tuning reference table above. Apply the **same** tuning block to both nodes — the standby node inherits the same workload when it becomes active.

---

## Step 4 — Set Failover Delay

The failover delay determines how long a standby node waits before promoting itself to active when it detects the current active node is unresponsive.

Add the following to `/etc/zabbix/zabbix_server.conf` on **both** nodes:

```ini
### HA Failover ###
# Minimum: 10s, Maximum: 15m, Recommended: 30s
# Lower values = faster failover but higher risk of split-brain during network blips
HANodeName=zabbix-node-siteA
```

> **Note:** The failover delay is set via a runtime command after the cluster is started (see Step 9). It is stored in the database and applies cluster-wide. You do not need to set it in the configuration file.

---

## Step 5 — Start Zabbix Server at Site A

Start and enable the Zabbix Server service on `{{ZBX_SERVER_A_IP}}`:

```bash
sudo systemctl enable --now zabbix-server
```

Check the service status:

```bash
sudo systemctl status zabbix-server
```

Check the log for startup errors:

```bash
sudo tail -50 /var/log/zabbix/zabbix_server.log
```

Look for these success indicators in the log:

```
HA manager started in active mode
```

---

## Step 6 — Verify Node 1 is Active

Query the HA status:

```bash
sudo zabbix_server -R ha_status
```

Expected output:

```
Zabbix HA cluster information:
  #  ID                        Name                Address                Status     Last Access
  1. xxxxxxxxxxxxxxxxxxxxxxxx  zabbix-node-siteA   {{ZBX_SERVER_A_IP}}:10051  active     0s
```

> **Note:** With only one node running, it automatically becomes the active node.

---

## Step 7 — Start Zabbix Server at Site B

Start and enable the Zabbix Server service on `{{ZBX_SERVER_B_IP}}`:

```bash
sudo systemctl enable --now zabbix-server
```

Check the log for startup:

```bash
sudo tail -50 /var/log/zabbix/zabbix_server.log
```

Look for this success indicator:

```
HA manager started in standby mode
```

---

## Step 8 — Verify HA Cluster

Query the HA status from **either** node:

```bash
sudo zabbix_server -R ha_status
```

Expected output:

```
Zabbix HA cluster information:
  #  ID                        Name                Address                     Status     Last Access
  1. xxxxxxxxxxxxxxxxxxxxxxxx  zabbix-node-siteA   {{ZBX_SERVER_A_IP}}:10051   active     2s
  2. yyyyyyyyyyyyyyyyyyyyyyyy  zabbix-node-siteB   {{ZBX_SERVER_B_IP}}:10051   standby    4s
```

Verify the cluster from the database directly:

```bash
psql -h {{VIP_DB_A}} -U zabbix -d zabbix -c "SELECT ha_nodeid, name, address, status, lastaccess FROM ha_node ORDER BY ha_nodeid;"
```

Status values:

| Status Code | Meaning |
|-------------|---------|
| 0 | Active |
| 1 | Standby |
| 2 | Unavailable |
| 3 | Stopped |

---

## Step 9 — Set Failover Delay via Runtime Command

Set the failover delay to 30 seconds (recommended):

```bash
sudo zabbix_server -R ha_set_failover_delay=30s
```

Expected output:

```
Failover delay is set to 30 seconds
```

Verify the setting:

```bash
sudo zabbix_server -R ha_status
```

The failover delay is stored in the database and applies to the entire cluster. You only need to run this command once on either node.

> **Tuning guidance:**
> - **30 seconds** — Recommended for most deployments. Balances fast failover with tolerance for brief network interruptions.
> - **10 seconds** — Minimum value. Use only if inter-site latency is very low (`{{INTER_SITE_RTT_MS}}` < 5ms) and network is highly reliable.
> - **60 seconds** — Conservative. Use if the WAN link between sites experiences frequent brief outages.

---

## Verification Checkpoint

| # | Check | Expected Result | Command / Location |
|---|-------|-----------------|--------------------|
| 1 | Both nodes visible in ha_status | 2 nodes listed | `zabbix_server -R ha_status` |
| 2 | Node 1 status | `active` | `zabbix_server -R ha_status` |
| 3 | Node 2 status | `standby` | `zabbix_server -R ha_status` |
| 4 | DB connectivity — Node 1 | Connected (no errors in log) | `tail /var/log/zabbix/zabbix_server.log` on Site A |
| 5 | DB connectivity — Node 2 | Connected (no errors in log) | `tail /var/log/zabbix/zabbix_server.log` on Site B |
| 6 | Failover delay | 30 seconds | `zabbix_server -R ha_status` |
| 7 | Server log — Site A | `HA manager started in active mode`, no errors | `tail -100 /var/log/zabbix/zabbix_server.log` |
| 8 | Server log — Site B | `HA manager started in standby mode`, no errors | `tail -100 /var/log/zabbix/zabbix_server.log` |
| 9 | Service enabled on boot | `enabled` | `systemctl is-enabled zabbix-server` on both nodes |
| 10 | Database HA node table | 2 rows, status 0 and 1 | `psql -h {{VIP_DB_A}} -U zabbix -d zabbix -c "SELECT * FROM ha_node;"` |

---

## Troubleshooting

### Node Stuck in Standby (Never Becomes Active)

**Symptoms:** Both nodes show `standby` in `ha_status`, or the second node never transitions to `standby` (stays `unavailable`).

**Resolution:**

1. Check that Node 1 started correctly and is truly active:
   ```bash
   # On Site A
   sudo zabbix_server -R ha_status
   sudo tail -100 /var/log/zabbix/zabbix_server.log | grep -i "HA"
   ```
2. Verify both nodes can reach the database:
   ```bash
   # On each node
   psql -h {{VIP_DB_A}} -U zabbix -d zabbix -c "SELECT 1;"
   psql -h {{VIP_DB_B}} -U zabbix -d zabbix -c "SELECT 1;"
   ```
3. Verify the `ha_node` table has both entries:
   ```bash
   psql -h {{VIP_DB_A}} -U zabbix -d zabbix -c "SELECT name, address, status, lastaccess FROM ha_node;"
   ```
4. If a node's `lastaccess` is stale (more than the failover delay ago), the node may have lost database connectivity. Check KEMP VIP status and Patroni health.
5. Verify the Zabbix server process is running:
   ```bash
   ps aux | grep zabbix_server
   ```

### Database Connection Refused

**Symptoms:** Zabbix server log shows `cannot connect to database` or `connection refused`.

**Resolution:**

1. Verify the KEMP Database VIP is healthy:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" http://{{PG_A_IP}}:8008/primary
   ```
2. Test direct database connectivity:
   ```bash
   psql -h {{VIP_DB_A}} -p 5432 -U zabbix -d zabbix -c "SELECT 1;"
   ```
3. If using PgBouncer (DBPort=6432), verify PgBouncer is running:
   ```bash
   psql -h {{VIP_DB_A}} -p 6432 -U zabbix -d pgbouncer -c "SHOW POOLS;"
   ```
4. Check `pg_hba.conf` allows connections from the Zabbix server IPs:
   ```bash
   # On the PostgreSQL primary
   sudo grep -i "zabbix" /etc/patroni/pg_hba.conf
   ```
   Ensure entries exist for `{{ZBX_SERVER_A_IP}}` and `{{ZBX_SERVER_B_IP}}`
5. Verify the `zabbix` database user password matches `{{DB_ZABBIX_PASSWORD}}`:
   ```bash
   PGPASSWORD='{{DB_ZABBIX_PASSWORD}}' psql -h {{VIP_DB_A}} -U zabbix -d zabbix -c "SELECT 1;"
   ```

### Duplicate HANodeName Error

**Symptoms:** Second node fails to start with an error about duplicate node name or conflicting HA node registration.

**Resolution:**

1. Verify each node has a **unique** `HANodeName`:
   ```bash
   # On Site A
   grep HANodeName /etc/zabbix/zabbix_server.conf
   # Expected: HANodeName=zabbix-node-siteA

   # On Site B
   grep HANodeName /etc/zabbix/zabbix_server.conf
   # Expected: HANodeName=zabbix-node-siteB
   ```
2. If a previous failed start left a stale entry, clean it up:
   ```bash
   psql -h {{VIP_DB_A}} -U zabbix -d zabbix -c "SELECT ha_nodeid, name, status FROM ha_node;"
   ```
3. Remove the stale entry if one exists with the wrong name:
   ```bash
   # Only if absolutely necessary — use the ha_nodeid from the SELECT above
   psql -h {{VIP_DB_A}} -U zabbix -d zabbix -c "DELETE FROM ha_node WHERE name='<stale_node_name>';"
   ```
4. Restart the Zabbix server on the affected node:
   ```bash
   sudo systemctl restart zabbix-server
   ```

### Failover Takes Too Long or Does Not Occur

**Symptoms:** Active node goes down but standby does not promote within the expected failover delay.

**Resolution:**

1. Verify the failover delay setting:
   ```bash
   sudo zabbix_server -R ha_status
   ```
2. Check that the standby node can still reach the database:
   ```bash
   # On the standby node
   psql -h {{VIP_DB_B}} -U zabbix -d zabbix -c "SELECT 1;"
   ```
3. The standby node promotes itself only when the active node's `lastaccess` in the `ha_node` table exceeds the failover delay. If the database VIP is also down (e.g., the entire site failed), the standby must be able to reach the database via its own site's VIP.
4. Check the standby node's log for failover activity:
   ```bash
   sudo tail -200 /var/log/zabbix/zabbix_server.log | grep -i "failover\|active\|standby"
   ```
