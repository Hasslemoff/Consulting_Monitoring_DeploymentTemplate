# Phase 14 — Validation and HA Testing

> **Objective:** Perform end-to-end validation of the entire monitoring stack and execute HA failover testing for every component. This phase confirms that all failure scenarios produce the expected automatic recovery behavior before the system enters production.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| All phases 01-13 complete | Every component deployed, configured, and individually verified |
| Maintenance window scheduled | HA testing causes brief service interruptions; schedule accordingly |
| All stakeholders notified | Operations, network, and application teams aware of planned failovers |
| Monitoring dashboard open | Keep the Zabbix frontend open during tests to observe real-time behavior |
| Terminal sessions ready | SSH sessions to both Zabbix servers, both PG nodes, etcd nodes, and proxy nodes |

> **Warning:** These tests intentionally cause service disruptions. Execute only during an approved maintenance window with rollback plans ready.

---

## Step 1 — Zabbix Server HA Failover Test

### 1a. Identify Current Active Node

```bash
# Run on either Zabbix server
zabbix_server -R ha_status
```

Record which node is currently `active` and which is `standby`. The examples below assume `zabbix-node-siteA` is active.

### 1b. Graceful Failover Test

Stop the active Zabbix server cleanly:

```bash
# On zabbix-node-siteA (active node)
systemctl stop zabbix-server
```

**Immediately start a timer.** On the surviving node:

```bash
# On zabbix-node-siteB — monitor for promotion
# The standby should promote within approximately 5 seconds for a graceful stop
zabbix_server -R ha_status
```

**Expected result:**
- `zabbix-node-siteB` becomes `active` within ~5 seconds
- `zabbix-node-siteA` shows as `stopped` or `unavailable`

**Record the actual failover time:** ______ seconds

### 1c. Ungraceful Failover Test

First, restore the stopped node and allow it to rejoin as standby:

```bash
# On zabbix-node-siteA
systemctl start zabbix-server
# Wait 30 seconds for it to register as standby
sleep 30
zabbix_server -R ha_status
# Confirm: siteA = standby, siteB = active
```

Now simulate a crash on the current active node:

```bash
# On zabbix-node-siteB (now active)
kill -9 $(pidof zabbix_server)
```

**Immediately start a timer.** On the surviving node:

```bash
# On zabbix-node-siteA — monitor for promotion
# Ungraceful failover takes up to ha_failover_delay (default 30s) + detection interval (5s) = ~35 seconds
watch -n 2 'zabbix_server -R ha_status'
```

**Expected result:**
- `zabbix-node-siteA` becomes `active` within ~35 seconds
- `zabbix-node-siteB` shows as `unavailable`

**Record the actual failover time:** ______ seconds

### 1d. Restore and Verify

```bash
# On zabbix-node-siteB — restart the crashed node
systemctl start zabbix-server
sleep 30

# Verify both nodes are operational
zabbix_server -R ha_status
# Expected: one active, one standby — both healthy
```

---

## Step 2 — Database (Patroni) Failover Test

### 2a. Pre-Test — Record Cluster State

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Record the current Leader, Sync Standby/Async Standby, and lag values.

### 2b. Graceful Switchover

```bash
# Initiate a planned switchover
# This prompts for confirmation — specify the new leader and current leader
patronictl -c /etc/patroni/patroni.yml switchover \
  --master pg-siteA \
  --candidate pg-siteB \
  --force
```

**Monitor the switchover:**

```bash
# Watch the cluster state in real time
watch -n 2 'patronictl -c /etc/patroni/patroni.yml list'
```

**Expected result:**
- `pg-siteB` becomes `Leader` within 15-30 seconds
- `pg-siteA` becomes `Sync Standby` or `Replica`
- Replication lag returns to 0

### 2c. Ungraceful Database Failure

**Ungraceful database failure:**
```bash
# Simulate crash of primary PostgreSQL
sudo kill -9 $(pidof postgres)

# Verify Patroni detects failure and promotes standby
# Watch: patronictl -c /etc/patroni/patroni.yml list
# Expected: Site B promoted within 30 seconds (TTL expiry)
```

### 2d. Verify KEMP Re-Routes to New Primary

```bash
# Check which backend KEMP is routing to
# The Patroni health check should now mark pg-siteB as primary
curl -s -o /dev/null -w "%{http_code}" http://{{PG_A_IP}}:8008/primary
# Expected: 503 (pg-siteA is now standby)

curl -s -o /dev/null -w "%{http_code}" http://{{PG_B_IP}}:8008/primary
# Expected: 200 (pg-siteB is now primary)

# Verify database connectivity through both VIPs
psql -h {{VIP_DB_A}} -p 5432 -U zabbix -d zabbix -c "SELECT pg_is_in_recovery();"
# Expected: f (false — connected to primary via VIP)

psql -h {{VIP_DB_B}} -p 5432 -U zabbix -d zabbix -c "SELECT pg_is_in_recovery();"
# Expected: f (false — both VIPs route to the same primary)
```

### 2e. Verify Zabbix Server Reconnected

```bash
# Check Zabbix server logs for database reconnection
tail -50 /var/log/zabbix/zabbix_server.log | grep -i "database"
# Look for: successful reconnection messages, no persistent errors
```

### 2f. Verify Replication Resumes

```bash
patronictl -c /etc/patroni/patroni.yml list
# Expected: Leader = pg-siteB, Sync Standby = pg-siteA, Lag = 0
```

### 2g. Switch Back (Optional)

If the original topology is desired, perform another switchover:

```bash
patronictl -c /etc/patroni/patroni.yml switchover \
  --master pg-siteB \
  --candidate pg-siteA \
  --force
```

Verify the cluster returns to the original state.

---

## Step 3 — KEMP HA Failover Test

### 3a. Identify Current Active Unit

Access the KEMP WUI at `https://{{KEMP_A1_IP}}:8443` and navigate to **System Configuration > High Availability**. Record which unit is Master and which is Slave.

### 3b. Reboot Active KEMP Unit

```bash
# If SSH is enabled on the KEMP, or use vCenter to reboot the VM
# Alternatively, use the WUI: System Configuration > High Availability > Force Failover
```

From vCenter, reboot the active KEMP VM (e.g., `KEMP-VLM-A1`).

### 3c. Verify Passive Unit Promotes

**Immediately test frontend accessibility:**

```bash
# Test frontend VIP accessibility (should remain reachable through the promoted passive unit)
curl -sk -o /dev/null -w "%{http_code}" https://{{VIP_FRONTEND_A}}
# Expected: 200

# Test database VIP routing
psql -h {{VIP_DB_A}} -p 5432 -U zabbix -d zabbix -c "SELECT 1;"
# Expected: successful query
```

Access the WUI on the promoted unit (`https://{{KEMP_A2_IP}}:8443`):
- Confirm it shows **Master** status
- Confirm all Virtual Services are **Up**

### 3d. Wait for Rebooted Unit to Rejoin

After the rebooted KEMP VM comes back online:

```bash
# Verify HA pair is healthy again
curl -k -s "https://bal:{{KEMP_ADMIN_PASSWORD}}@{{KEMP_A1_IP}}:8443/access/hastatus" | xmllint --format -
```

> **Security note:** Avoid passing passwords in URLs on the command line. Use environment variables (e.g., `curl -k -s "https://bal:${KEMP_ADMIN_PASSWORD}@..."`) or a `.netrc` file to avoid shell history exposure.

Both units should be visible, with one as Master and one as Slave.

### 3e. Repeat for Site B

Repeat Steps 3b-3d for the Site B KEMP HA pair using `{{KEMP_B1_IP}}` and `{{KEMP_B2_IP}}`.

### Step 3b (addendum): etcd Member Loss Test

```bash
# Stop one etcd node (not the leader)
ssh etcd-3 "sudo systemctl stop etcd"

# Verify quorum maintained (2 of 3 nodes)
etcdctl endpoint health \
  --endpoints=https://{{ETCD_1_IP}}:2379,https://{{ETCD_2_IP}}:2379 \
  --cacert={{ETCD_CA_CERT_PATH}}

# Verify Patroni still operational
patronictl -c /etc/patroni/patroni.yml list

# Restore etcd node
ssh etcd-3 "sudo systemctl start etcd"
```

---

## Step 4 — Proxy Group Failover Test

### 4a. Record Current Proxy Assignment

In the Zabbix Frontend, navigate to **Administration > Proxies**. Record which hosts are assigned to each proxy in the group.

### 4b. Stop One Proxy

```bash
# On proxy-siteA-01
systemctl stop zabbix-proxy
```

### 4c. Verify Host Redistribution

Wait 60-120 seconds for the proxy group to redistribute hosts.

In the Zabbix Frontend:
1. Navigate to **Administration > Proxies**
2. Verify that hosts previously assigned to `proxy-siteA-01` have been redistributed to remaining proxies in the group
3. Check that no hosts are in an "unmonitored" state

```bash
# Verify monitoring continuity — check Latest Data for a host that was on the stopped proxy
# Navigate to: Monitoring > Latest data > select a host that was assigned to proxy-siteA-01
# Verify data timestamps are recent (within the last few minutes)
```

### 4d. Restart Proxy and Verify Rebalance

```bash
# On proxy-siteA-01
systemctl start zabbix-proxy
```

Wait 60-120 seconds, then verify in **Administration > Proxies** that hosts have rebalanced across all proxies in the group.

### 4e. Repeat for Site B

Repeat the test by stopping `proxy-siteB-01` and verifying redistribution to `proxy-siteB-02`.

---

## Step 5 — Full Site Failure Simulation

> **Warning:** This is the most disruptive test. Only perform if the environment supports it and stakeholders have approved. This test validates the entire DR chain.

### 5a. Prerequisites

- Verify etcd-3 (arbitrator) is at Site B or a third location — required for automatic failover
- Confirm etcd quorum can survive Site A loss (etcd-2 + etcd-3 = 2 of 3)

### 5b. Simulate Site A Failure

If possible, shut down **all** Site A VMs simultaneously from vCenter:
- `zabbix-node-siteA`
- `pg-siteA`
- `etcd-1`
- `frontend-siteA`
- `kemp-a1`, `kemp-a2`
- `proxy-siteA-01`, `proxy-siteA-02`, `proxy-siteA-snmptrap`

### 5c. Verify Automatic Failover Chain

**Start a stopwatch** when VMs are shut down, then verify each component:

1. **etcd quorum maintained:**
   ```bash
   # From etcd-2 or etcd-3
   ETCDCTL_API=3 etcdctl endpoint health \
     --endpoints=https://{{ETCD_2_IP}}:2379,https://{{ETCD_3_IP}}:2379 \
     --cacert={{ETCD_CA_CERT_PATH}} \
     --cert=/etc/etcd/pki/etcd-2.crt \
     --key=/etc/etcd/pki/etcd-2.key
   # Expected: both endpoints healthy
   ```

2. **Patroni auto-promotes PG at Site B:**
   ```bash
   # From pg-siteB
   patronictl -c /etc/patroni/patroni.yml list
   # Expected: pg-siteB = Leader, pg-siteA = unreachable
   ```

3. **KEMP at Site B routes to new primary:**
   ```bash
   psql -h {{VIP_DB_B}} -p 5432 -U zabbix -d zabbix -c "SELECT pg_is_in_recovery();"
   # Expected: f (false)
   ```

4. **Zabbix Server B becomes active:**
   ```bash
   # From zabbix-node-siteB
   zabbix_server -R ha_status
   # Expected: zabbix-node-siteB = active
   ```

5. **Proxies at Site B continue monitoring:**
   ```bash
   # Check proxy processes
   systemctl status zabbix-proxy   # On proxy-siteB-01 and proxy-siteB-02
   ```

6. **Frontend accessible via Site B:**
   ```bash
   curl -sk -o /dev/null -w "%{http_code}" https://{{VIP_FRONTEND_B}}
   # Expected: 200
   ```

**Record the total time from VM shutdown to full service restoration at Site B:** ______ minutes

### 5d. Restore Site A

Power on all Site A VMs from vCenter. Monitor resynchronization:

```bash
# Wait for VMs to boot (2-5 minutes)

# Verify etcd-1 rejoins the cluster
ETCDCTL_API=3 etcdctl endpoint health --cluster \
  --endpoints=https://{{ETCD_1_IP}}:2379 \
  --cacert={{ETCD_CA_CERT_PATH}} \
  --cert=/etc/etcd/pki/etcd-1.crt \
  --key=/etc/etcd/pki/etcd-1.key

# Verify Patroni replica rebuilds
patronictl -c /etc/patroni/patroni.yml list
# pg-siteA should rejoin as Sync Standby/Replica (may take time for initial sync)

# Verify Zabbix server rejoins HA
zabbix_server -R ha_status
# zabbix-node-siteA should show as standby
```

> **Note:** Depending on how long Site A was down, the PostgreSQL replica rebuild may take significant time as it replays WAL files. Monitor with `patronictl list` until lag returns to 0.

---

## Step 6 — Data Integrity Verification

### 6a. Check for Data Gaps During Failover

After each failover test, verify that monitoring data has no gaps:

1. In the Zabbix Frontend, navigate to **Monitoring > Latest data**
2. Select several hosts that were actively monitored during the failover
3. Verify the **Last check** timestamps show continuous data collection
4. For hosts monitored through proxies, verify proxy-buffered data was flushed:
   ```bash
   # Check proxy buffer status
   # In the frontend: Administration > Queue
   # Verify no items are significantly delayed
   ```

### 6b. Check History Graphs for Continuity

1. Navigate to **Monitoring > Hosts > select a host > Graphs**
2. Set the time range to cover the failover window
3. Verify the graph shows continuous data points without gaps
4. A brief gap (< 60 seconds) during database failover is acceptable

---

## Step 7 — Alert Delivery Test

### 7a. Create a Test Trigger

Create a temporary item and trigger to generate a test alert:

1. In the Zabbix Frontend, navigate to **Data collection > Hosts**
2. Select any monitored host (e.g., `zabbix-node-siteA`)
3. Create a **Calculated item**:
   - Name: `TEST - Alert delivery test`
   - Key: `test.alert.delivery`
   - Formula: `last(//system.cpu.util)`
   - Type of information: Numeric (float)
   - Update interval: 30s
4. Create a **Trigger**:
   - Name: `TEST - Alert delivery verification`
   - Severity: **High**
   - Expression: `last(/zabbix-node-siteA/test.alert.delivery)>=0`
   - This triggers immediately since CPU utilization is always >= 0

### 7b. Verify Alert Delivery to All Media Types

Confirm alerts are received on each configured media type:

| # | Media Type | Verification Method | Received | Pass |
|---|------------|---------------------|----------|------|
| 1 | Email | Check inbox for `{{SMTP_FROM_EMAIL}}` sender | [ ] | [ ] |
| 2 | Microsoft Teams | Check `{{TEAMS_WEBHOOK_URL}}` channel for Zabbix card | [ ] | [ ] |
| 3 | Slack | Check `{{SLACK_CHANNEL}}` for Zabbix message | [ ] | [ ] |
| 4 | PagerDuty | Check PagerDuty service for incident | [ ] | [ ] |

### 7c. Verify Recovery Notifications

Resolve the test trigger by disabling it:

1. Navigate to the test trigger and **Disable** it
2. Verify recovery notifications are received on all media types

| # | Media Type | Recovery Received | Pass |
|---|------------|-------------------|------|
| 1 | Email | [ ] | [ ] |
| 2 | Microsoft Teams | [ ] | [ ] |
| 3 | Slack | [ ] | [ ] |
| 4 | PagerDuty (auto-resolve) | [ ] | [ ] |

### 7d. Test Escalation

1. Create a new trigger with **High** severity (or reuse the above)
2. Let it remain in PROBLEM state for 15+ minutes (matching the escalation step timing)
3. Verify that PagerDuty escalation fires after the configured delay
4. Verify that the next escalation level is notified

**Record escalation timing:** First notification at ______ minutes, escalation at ______ minutes

### 7e. Clean Up

Delete or disable the test item and trigger after verification is complete.

---

## Step 8 — Backup Restore Test

### 8a. pgBackRest Restore Test

If a test VM is available, perform a trial restore to validate backup integrity:

```bash
# On a separate test VM with PostgreSQL 16 and pgBackRest installed
# Copy /etc/pgbackrest.conf from the production node

# Restore the latest backup
sudo -u postgres pgbackrest --stanza=zabbix-cluster \
  --type=immediate \
  --target-action=promote \
  --delta \
  restore

# Start PostgreSQL
systemctl start postgresql-16

# Verify the database is accessible
psql -U zabbix -d zabbix -c "SELECT count(*) FROM hosts;"
# Expected: returns the count of monitored hosts
```

If no test VM is available, at minimum verify backup metadata:

```bash
sudo -u postgres pgbackrest info
sudo -u postgres pgbackrest --stanza=zabbix-cluster verify
# Expected: no errors
```

### 8b. etcd Snapshot Restore Test

Verify the snapshot is valid (non-destructive check):

```bash
ETCDCTL_API=3 etcdctl snapshot status /var/lib/etcd/backup/<latest-snapshot>.db --write-out=table
# Expected: shows hash, revision, total keys, and total size
```

---

## Step 9 — End-to-End Smoke Tests

Verify data flows from every monitoring source through the complete pipeline.

### 9a. Windows Agent Data Flow

```bash
# In the Zabbix Frontend: Monitoring > Latest data
# Filter by a Windows host monitored through a proxy
# Verify items like "CPU utilization", "Available memory", "Disk space" have recent values
```

**Host tested:** ______________________ **Data flowing:** [ ] Yes [ ] No

### 9b. Linux Agent Data Flow

```bash
# Filter by a Linux host monitored through a proxy
# Verify items like "CPU utilization", "Available memory", "Network interfaces" have recent values
```

**Host tested:** ______________________ **Data flowing:** [ ] Yes [ ] No

### 9c. Kubernetes Cluster Data Flow

```bash
# Filter by the Kubernetes cluster host ({{K8S_CLUSTER_NAME}})
# Verify items from the Kubernetes templates:
#   - Node count, pod count, namespace metrics
#   - Container CPU/memory usage
```

**Cluster tested:** ______________________ **Data flowing:** [ ] Yes [ ] No

### 9d. Microsoft 365 Data Collection

```bash
# Filter by the Microsoft 365 host
# Verify M365 items are collecting:
#   - Service health, license counts, mailbox statistics
```

**Tenant tested:** ______________________ **Data flowing:** [ ] Yes [ ] No

### 9e. VMware Discovery

```bash
# Filter by the vCenter host
# Verify VMware discovery has run and populated:
#   - ESXi hosts, datastores, VMs discovered via LLD
#   - CPU/memory/storage metrics collecting
```

**vCenter tested:** ______________________ **Data flowing:** [ ] Yes [ ] No

---

## Verification Checkpoint

| # | Check | Expected Result | Pass |
|---|-------|-----------------|------|
| 1 | Zabbix Server HA — graceful failover | Standby promotes within ~5 seconds | [ ] |
| 2 | Zabbix Server HA — ungraceful failover | Standby promotes within ~35 seconds | [ ] |
| 3 | Zabbix Server HA — node rejoins as standby | Restarted node becomes standby | [ ] |
| 4 | Patroni — graceful switchover | New primary within 15-30 seconds, replication resumes | [ ] |
| 5 | Patroni — KEMP re-routes to new primary | VIP connects to new primary (pg_is_in_recovery = f) | [ ] |
| 6 | Patroni — Zabbix server reconnects | No persistent database errors in Zabbix logs | [ ] |
| 7 | KEMP HA — passive promotes on active reboot | VIPs remain accessible during KEMP failover | [ ] |
| 8 | Proxy group — hosts redistribute on proxy stop | No unmonitored hosts during proxy outage | [ ] |
| 9 | Proxy group — hosts rebalance on proxy restart | Even distribution restored after proxy restarts | [ ] |
| 10 | Full site failure — automatic failover (if tested) | All services operational at Site B within 1-3 minutes | [ ] |
| 11 | Full site failure — resynchronization after restore | All components rejoin and sync after Site A restored | [ ] |
| 12 | Data continuity — no gaps during failover | Latest data timestamps show continuous collection | [ ] |
| 13 | Data continuity — proxy buffer flush | Queued items delivered after server recovery | [ ] |
| 14 | Alert delivery — email | Alert and recovery received | [ ] |
| 15 | Alert delivery — Microsoft Teams | Alert and recovery received | [ ] |
| 16 | Alert delivery — Slack | Alert and recovery received | [ ] |
| 17 | Alert delivery — PagerDuty | Alert, recovery, and escalation working | [ ] |
| 18 | Backup restore — pgBackRest verified | Backup info and verify pass without errors | [ ] |
| 19 | Backup restore — etcd snapshot verified | Snapshot status shows valid data | [ ] |
| 20 | Smoke test — Windows agent data flowing | Recent data visible in Latest data | [ ] |
| 21 | Smoke test — Linux agent data flowing | Recent data visible in Latest data | [ ] |
| 22 | Smoke test — Kubernetes data flowing | Cluster metrics collecting | [ ] |
| 23 | Smoke test — M365 data collecting | Service health and license data present | [ ] |
| 24 | Smoke test — VMware discovery running | Hosts, VMs, and datastores discovered | [ ] |

---

## Troubleshooting

### Failover Takes Too Long

**Symptom:** Zabbix Server HA failover exceeds 60 seconds, or Patroni failover exceeds 30 seconds.

**Resolution:**

1. Check Zabbix Server HA failover delay:
   ```bash
   grep -i "ha_failover_delay" /etc/zabbix/zabbix_server.conf
   # Default is 30 seconds for ungraceful; reduce to 15 if acceptable
   ```
2. Check Patroni TTL and loop_wait settings:
   ```bash
   patronictl -c /etc/patroni/patroni.yml show-config | grep -E "ttl|loop_wait|retry_timeout"
   # ttl=30, loop_wait=10, retry_timeout=10 are typical defaults
   ```
3. Check etcd leader election performance:
   ```bash
   ETCDCTL_API=3 etcdctl endpoint status --write-out=table \
     --endpoints=https://{{ETCD_1_IP}}:2379,https://{{ETCD_2_IP}}:2379,https://{{ETCD_3_IP}}:2379 \
     --cacert={{ETCD_CA_CERT_PATH}} \
     --cert=/etc/etcd/pki/etcd-1.crt \
     --key=/etc/etcd/pki/etcd-1.key
   ```
4. Verify network latency between sites is within expected range:
   ```bash
   ping -c 20 {{PG_B_IP}}
   # Compare with {{INTER_SITE_RTT_MS}} from Phase 1
   ```

### Data Gaps After Failover

**Symptom:** Monitoring > Latest data shows missing data points during the failover window.

**Resolution:**

1. Check proxy buffer settings — proxies should buffer data during server unavailability:
   ```bash
   grep -i "ProxyLocalBuffer\|ProxyOfflineBuffer" /etc/zabbix/zabbix_proxy.conf
   # ProxyLocalBuffer=0 (hours of local buffering; 0 = use ProxyOfflineBuffer only)
   # ProxyOfflineBuffer=1 (hours of offline buffering)
   ```
2. Verify the Zabbix server queue is draining after recovery:
   - In Frontend: **Administration > Queue**
   - All queue counts should return to 0 within 5-10 minutes
3. If gaps persist for agent-monitored (non-proxy) items, the gap equals the failover time — this is expected behavior for direct agent connections
4. For proxy-monitored items, increase `ProxyOfflineBuffer` if longer outages are anticipated

### Alerts Not Delivered

**Symptom:** Test trigger fires but no notification is received on one or more media types.

**Resolution:**

1. Check the alert log in Frontend: **Reports > Action log**
   - Look for the test trigger action
   - Check the **Status** column for errors
   - Click on failed entries to see the error message
2. Verify user media configuration:
   - **Administration > Users > select user > Media tab**
   - Confirm the correct email/endpoint is configured
   - Confirm severity filter includes **High** (or the test trigger severity)
3. Verify action configuration:
   - **Alerts > Actions > Trigger actions**
   - Confirm the action is enabled
   - Confirm the conditions match the test trigger
   - Confirm operations are configured for the correct media type and user/group
4. Test media type connectivity:
   - **Alerts > Media types > select media type > Test**
   - Send a test message directly from the media type configuration
5. Check Zabbix server logs for alert delivery errors:
   ```bash
   grep -i "alert\|media\|cannot" /var/log/zabbix/zabbix_server.log | tail -50
   ```

### Proxy Hosts Do Not Redistribute

**Symptom:** After stopping a proxy, its hosts do not move to another proxy in the group.

**Resolution:**

1. Confirm the proxies are in a **Proxy group** (not standalone):
   - Frontend: **Administration > Proxy groups**
   - Verify the test proxy is a member of the group
2. Check proxy group failover settings:
   - `MinOnline` must be satisfied by the remaining proxies
3. Verify the remaining proxies are healthy:
   - Frontend: **Administration > Proxies**
   - Check the `Last seen` timestamp is recent
4. Allow sufficient time — proxy group redistribution may take up to 2 minutes
5. Check Zabbix server logs for proxy group events:
   ```bash
   grep -i "proxy group\|proxy.*offline\|reassign" /var/log/zabbix/zabbix_server.log | tail -30
   ```

### KEMP VIP Does Not Migrate

**Symptom:** After rebooting the active KEMP unit, the VIP becomes unreachable.

**Resolution:**

1. Verify VMware port group security policies (Step 1 of Phase 5):
   - **MAC Address Changes** must be **Accept**
   - **Forged Transmits** must be **Accept**
2. Check that the passive unit has the correct HA configuration:
   ```
   WUI > System Configuration > High Availability
   ```
3. Verify TCP port 6973 is open between the two KEMP units (HA sync port)
4. If the VIP is still unreachable, log into the surviving KEMP unit directly via its management IP to diagnose
