# Phase 15 — Day-2 Operations

> **Objective:** Configure ongoing operational procedures for the Zabbix monitoring platform: self-monitoring, external meta-monitoring, log rotation, housekeeping, upgrade procedures, maintenance windows, and document known limitations.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Phase 14 complete | All validation and HA testing passed |
| Production handoff approved | Stakeholders have signed off on the deployment |
| Operations team identified | Staff responsible for ongoing platform maintenance |
| External monitoring system available | An independent monitoring platform for meta-monitoring (NOT Zabbix itself) |

---

## Step 1 — Self-Monitoring ("Monitoring the Monitoring")

Zabbix can monitor its own internal health using built-in items and templates.

### 1a. Link the Zabbix Server Health Template

1. In the Zabbix Frontend, navigate to **Data collection > Hosts**
2. Select the Zabbix server host (e.g., `zabbix-node-siteA`)
3. Click **Templates** tab
4. Link the **Zabbix server health** template (included with Zabbix out of the box)
5. Click **Update**
6. Repeat for `zabbix-node-siteB`

This template monitors:
- Cache utilization (value cache, history cache, trend cache)
- Internal process busy percentages (pollers, trappers, preprocessors, etc.)
- New Values Per Second (NVPS)
- Queue length (items waiting to be processed)

### 1b. Key Internal Items to Monitor

Verify the following items are collecting data after linking the template:

| Item Key | Description | Alert Threshold |
|----------|-------------|-----------------|
| `zabbix[wcache,values,*]` | Write cache — values per second (NVPS) | Informational; track trends |
| `zabbix[rcache,buffer,pfree]` | Read cache — % free buffer | Alert when < 20% |
| `zabbix[wcache,buffer,pfree]` | Write cache — % free buffer | Alert when < 20% |
| `zabbix[process,poller,avg,busy]` | Average poller process busy % | Alert when > 75% |
| `zabbix[process,trapper,avg,busy]` | Average trapper process busy % | Alert when > 75% |
| `zabbix[process,preprocessor,avg,busy]` | Average preprocessor busy % | Alert when > 75% |
| `zabbix[process,history syncer,avg,busy]` | Average history syncer busy % | Alert when > 75% |
| `zabbix[queue,10m]` | Items delayed more than 10 minutes | Alert when > 100 |

### 1c. Create Custom Alert Triggers (If Not in Template)

If the default template does not include threshold-based triggers, create them manually:

**Cache Free Buffer Alert:**

- Name: `Zabbix read cache buffer critically low`
- Expression: `last(/zabbix-node-siteA/zabbix[rcache,buffer,pfree])<20`
- Severity: **High**

**Process Busy Alert:**

- Name: `Zabbix pollers overloaded`
- Expression: `avg(/zabbix-node-siteA/zabbix[process,poller,avg,busy],10m)>75`
- Severity: **Warning**

**Queue Alert:**

- Name: `Zabbix queue growing — items delayed`
- Expression: `last(/zabbix-node-siteA/zabbix[queue,10m])>100`
- Severity: **High**

---

## Step 2 — External Meta-Monitoring

Self-monitoring has a critical flaw: if Zabbix itself is down, it cannot alert on its own outage. An independent monitoring system must verify Zabbix availability from outside.

### 2a. Required External Checks

Configure the following checks on an independent monitoring system (e.g., UptimeRobot, PRTG, Nagios, site-specific tooling):

| # | Check Type | Target | Port | Description |
|---|------------|--------|------|-------------|
| 1 | HTTPS | `{{VIP_FRONTEND_A}}` | 443 | Frontend accessible via Site A KEMP VIP |
| 2 | HTTPS | `{{VIP_FRONTEND_B}}` | 443 | Frontend accessible via Site B KEMP VIP |
| 3 | TCP | `{{ZBX_SERVER_A_IP}}` | 10051 | Zabbix Server process — Site A |
| 4 | TCP | `{{ZBX_SERVER_B_IP}}` | 10051 | Zabbix Server process — Site B |
| 5 | TCP | `{{VIP_DB_A}}` | 5432 | Database available via Site A KEMP VIP |
| 6 | TCP | `{{VIP_DB_B}}` | 5432 | Database available via Site B KEMP VIP |
| 7 | HTTPS | `{{KEMP_A1_IP}}` | 8443 | KEMP WUI — Site A unit 1 |
| 8 | HTTPS | `{{KEMP_B1_IP}}` | 8443 | KEMP WUI — Site B unit 1 |

### 2b. Certificate Expiry Monitoring

Monitor TLS certificate expiry for all endpoints. Certificates should alert at least 30 days before expiry.

| Endpoint | Certificate Subject |
|----------|---------------------|
| `https://{{FQDN_FRONTEND}}:443` | Frontend SSL certificate |
| `https://{{KEMP_A1_IP}}:8443` | KEMP management certificate |
| `https://{{KEMP_B1_IP}}:8443` | KEMP management certificate |
| etcd peer/client certificates | Internal CA — track expiry manually or via Zabbix agent item |

> **Tip:** If using Zabbix Agent 2 on the servers, the `web.certificate.get` item key can monitor external certificate expiry from within Zabbix. Use this as a secondary check alongside the external monitor.

### 2c. Microsoft 365 App Registration Secret Expiry

Azure app registration client secrets expire on a configurable schedule (typically 1-2 years). Set a reminder to renew before expiry:

| Item | Details |
|------|---------|
| App Registration | `{{M365_APP_ID}}` in tenant `{{M365_TENANT_ID}}` |
| Secret Variable | `{{M365_CLIENT_SECRET}}` |
| Action | Renew in Azure Portal > App registrations > Certificates & secrets |

Create a Zabbix trigger or external calendar reminder at least 30 days before the secret expires.

---

## Step 3 — Log Rotation Configuration

Zabbix components write logs that can grow unbounded without rotation.

### 3a. Zabbix Server Log Rotation

Check if the RPM package already installed a logrotate config:

```bash
ls -la /etc/logrotate.d/zabbix*
```

If no configuration exists, or if the existing one needs adjustment, create or update:

```bash
cat > /etc/logrotate.d/zabbix-server << 'EOF'
/var/log/zabbix/zabbix_server.log {
    weekly
    rotate 12
    copytruncate
    compress
    delaycompress
    missingok
    notifempty
}
EOF
```

### 3b. Zabbix Proxy Log Rotation

```bash
cat > /etc/logrotate.d/zabbix-proxy << 'EOF'
/var/log/zabbix/zabbix_proxy.log {
    weekly
    rotate 12
    compress
    delaycompress
    missingok
    notifempty
    postrotate
        /usr/bin/systemctl reload zabbix-proxy 2>/dev/null || true
    endscript
}
EOF
```

### 3c. Zabbix Agent 2 Log Rotation

```bash
cat > /etc/logrotate.d/zabbix-agent2 << 'EOF'
/var/log/zabbix/zabbix_agent2.log {
    weekly
    rotate 4
    compress
    delaycompress
    missingok
    notifempty
}
EOF
```

> **Note:** The `copytruncate` directive is used for the Zabbix server because it writes to the log file continuously without reopening it on SIGHUP. Proxies and agents support log reopening via reload, so `postrotate` with `systemctl reload` is preferred where applicable.

### 3d. Verify Log Rotation

```bash
# Dry run to verify configuration syntax
logrotate -d /etc/logrotate.d/zabbix-server
logrotate -d /etc/logrotate.d/zabbix-proxy
logrotate -d /etc/logrotate.d/zabbix-agent2
```

---

## Step 4 — History and Housekeeping

Housekeeping controls how long Zabbix retains collected data, events, and trends. Properly configured housekeeping prevents unbounded database growth.

### 4a. Configure Global Housekeeping Settings

In the Zabbix Frontend, navigate to **Administration > Housekeeping**.

**History:**

| Setting | Value |
|---------|-------|
| Override item history period | **Enabled** |
| Data storage period (numeric items) | 14 days |
| Data storage period (character/text/log items) | 7 days |

**Trends:**

| Setting | Value |
|---------|-------|
| Override item trend period | **Enabled** |
| Data storage period | 365 days |

**Events:**

| Setting | Value |
|---------|-------|
| Trigger event and alert storage | 365 days |
| Internal event storage | 7 days |
| Network discovery event storage | 7 days |
| Autoregistration event storage | 7 days |

Click **Update** to apply.

### 4b. TimescaleDB Considerations

If TimescaleDB is installed (as recommended in Phase 2), housekeeping uses `DROP CHUNK` operations to delete entire time-range partitions instead of row-by-row `DELETE` statements. This is significantly faster and generates minimal I/O impact.

Verify TimescaleDB compression and retention policies are active:

```bash
psql -U zabbix -d zabbix -c "SELECT * FROM timescaledb_information.jobs WHERE proc_name = 'policy_retention';"
```

> **Important:** With TimescaleDB, the housekeeping period configured in the frontend controls the retention policy. Zabbix automatically manages the TimescaleDB retention jobs.

---

## Step 5 — Upgrade Procedures

### 5a. Minor Version Upgrade (e.g., 7.0.1 to 7.0.2)

Minor versions contain bug fixes and security patches. No database schema migration is required.

**Procedure:**

1. **Upgrade the standby Zabbix server node first:**
   ```bash
   # On zabbix-node-siteB (standby)
   dnf update -y zabbix-server-pgsql zabbix-web-pgsql zabbix-sql-scripts zabbix-agent2
   systemctl restart zabbix-server
   ```

2. **Verify the standby is healthy:**
   ```bash
   zabbix_server -R ha_status
   # Confirm siteB is standby and connected
   ```

3. **Upgrade the active Zabbix server node:**
   ```bash
   # On zabbix-node-siteA (active)
   dnf update -y zabbix-server-pgsql zabbix-web-pgsql zabbix-sql-scripts zabbix-agent2
   systemctl restart zabbix-server
   ```
   A brief failover occurs (~5 seconds) when the active node restarts. The standby promotes, then the restarted node rejoins as standby.

4. **Upgrade frontend nodes:**
   ```bash
   # On frontend-siteA and frontend-siteB
   dnf update -y zabbix-web-pgsql zabbix-nginx-conf
   systemctl restart php-fpm nginx
   ```

5. **Upgrade proxies (rolling):**
   ```bash
   # On each proxy, one at a time
   dnf update -y zabbix-proxy-sqlite3 zabbix-agent2
   systemctl restart zabbix-proxy
   ```

6. **Verify version:**
   - Frontend: Check bottom-right corner of the Zabbix dashboard
   - Server: `zabbix_server -V`
   - Proxy: `zabbix_proxy -V`

### 5b. Major Version Upgrade (e.g., 7.0 to 7.2)

Major versions include database schema migrations and may have breaking changes.

**Procedure:**

1. **Read the release notes** for the target version. Review breaking changes, deprecated features, and new requirements.

2. **Take a full backup** before proceeding:
   ```bash
   sudo -u postgres pgbackrest --stanza=zabbix-cluster --type=full backup
   ```

3. **Stop ALL Zabbix server HA nodes:**
   ```bash
   systemctl stop zabbix-server   # On zabbix-node-siteA
   systemctl stop zabbix-server   # On zabbix-node-siteB
   ```

4. **On ONE node only, temporarily disable HA** to run the schema migration:
   ```bash
   # On zabbix-node-siteA — edit /etc/zabbix/zabbix_server.conf
   # Comment out HANodeName:
   # HANodeName=zabbix-node-siteA
   ```

5. **Upgrade packages on that node:**
   ```bash
   dnf update -y zabbix-server-pgsql zabbix-web-pgsql zabbix-sql-scripts zabbix-agent2
   ```

6. **Start the server** — it will detect the schema version mismatch and run the migration automatically:
   ```bash
   systemctl start zabbix-server
   # Monitor the log for migration progress
   tail -f /var/log/zabbix/zabbix_server.log
   ```

7. **After migration completes**, restore `HANodeName`:
   ```bash
   # Uncomment HANodeName in /etc/zabbix/zabbix_server.conf
   HANodeName=zabbix-node-siteA
   systemctl restart zabbix-server
   ```

8. **Upgrade and start the remaining HA node:**
   ```bash
   # On zabbix-node-siteB
   dnf update -y zabbix-server-pgsql zabbix-web-pgsql zabbix-sql-scripts zabbix-agent2
   systemctl start zabbix-server
   ```

9. **Upgrade frontends and proxies** as described in Step 5a, items 4-5.

10. **Verify HA status:**
    ```bash
    zabbix_server -R ha_status
    # Both nodes should be present — one active, one standby
    ```

> **Warning:** Major upgrades are **not reversible** once the database schema migration runs. Ensure the full backup from step 2 is verified before proceeding.

---

## Step 6 — Regular Maintenance Tasks Checklist

### Weekly Tasks

| # | Task | Command / Location | Expected Result |
|---|------|--------------------|-----------------|
| 1 | Verify backup integrity | `sudo -u postgres pgbackrest --stanza=zabbix-cluster verify` | No errors |
| 2 | Check replication lag | `patronictl -c /etc/patroni/patroni.yml list` | Lag = 0 for sync standby |
| 3 | Review Zabbix server queue | Frontend: **Administration > Queue** | All counts at 0 or near 0 |
| 4 | Check for unsupported items | Frontend: **Data collection > Hosts** (filter: Status=Unsupported items) | Investigate new unsupported items |

### Monthly Tasks

| # | Task | Command / Location | Expected Result |
|---|------|--------------------|-----------------|
| 1 | Review disk usage on DB VMs | `df -h /var/lib/pgsql/` | Sufficient free space (> 20%) |
| 2 | Check certificate expiry dates | `openssl s_client -connect {{VIP_FRONTEND_A}}:443 </dev/null 2>/dev/null \| openssl x509 -noout -enddate` | > 30 days remaining |
| 3 | Audit auto-registration | Frontend: **Data collection > Hosts** (sort by name, look for orphans) | Remove decommissioned hosts |
| 4 | Review Zabbix server logs | `grep -i "error\|warning\|critical" /var/log/zabbix/zabbix_server.log \| tail -100` | No persistent errors |
| 5 | Check etcd cluster health | `etcdctl endpoint health --cluster` | All 3 endpoints healthy |

### Quarterly Tasks

| # | Task | Command / Location | Expected Result |
|---|------|--------------------|-----------------|
| 1 | Review NVPS and tune parameters | Frontend: **Reports > System information** (NVPS value) | NVPS within sizing tier ceiling |
| 2 | Test HA failover | Follow Phase 14 Steps 1-2 | Failover within expected times |
| 3 | Update Zabbix agent deployments | Check for new agent versions, deploy via automation | Agents at latest minor version |
| 4 | Review and update alert actions | Frontend: **Alerts > Actions** | Actions aligned with current team structure |
| 5 | Test backup restore | Restore to test VM if available (Phase 14 Step 8) | Restore completes successfully |

---

## Step 7 — Maintenance Windows

Zabbix maintenance windows suppress alert notifications during planned work.

### 7a. Configure a Maintenance Window

In the Zabbix Frontend, navigate to **Data collection > Maintenance**.

Click **Create maintenance period** and configure:

| Parameter | Description |
|-----------|-------------|
| Name | Descriptive name (e.g., `Monthly Patching - Site A`) |
| Maintenance type | See options below |
| Active since / Active till | Date range the maintenance window is available |
| Periods | Specific time windows within the active range |
| Host groups / Hosts | Which hosts are covered |

### 7b. Maintenance Types

| Type | Behavior | Use Case |
|------|----------|----------|
| **With data collection** | Data continues collecting; alerts are suppressed | OS patching, application restarts, rolling upgrades |
| **No data collection** | Data is discarded; alerts are suppressed | Hardware replacement, VM migration, storage maintenance |

### 7c. One-Time vs. Recurring

| Schedule | Configuration |
|----------|---------------|
| One-time | Set a single period with specific start/end times |
| Recurring | Set a daily, weekly, or monthly period (e.g., every Sunday 02:00-06:00) |

### 7d. Viewing Suppressed Problems

By default, problems that occur during maintenance are **hidden** from dashboards and problem views.

To see suppressed problems:
1. Open any **Problems** widget on a dashboard
2. Click the widget gear icon (edit)
3. Enable **Show suppressed problems**
4. Click **Apply**

Alternatively, in **Monitoring > Problems**, use the filter to enable **Show suppressed problems**.

> **Important:** Suppressed problems are not lost. They remain in the database and become visible once the maintenance window ends (if the problem persists). However, notifications are permanently skipped for the suppressed period.

---

## Step 8 — Known Limitations Reference

The following table summarizes architectural limitations of this deployment. Reference these during incident triage and capacity planning.

| # | Component | Limitation | Mitigation / Workaround |
|---|-----------|-----------|-------------------------|
| 1 | Zabbix Server HA | Database is a single point of failure (SPOF) for the Zabbix application; HA only covers the server process, not the DB | Patroni + KEMP provide database HA; pgBackRest provides backup for corruption scenarios |
| 2 | Zabbix Server HA | Agents/proxies must NOT hardcode a single `ZBX_SERVER` address | Use KEMP VIP or semicolon-separated `ServerActive` lists (e.g., `ServerActive={{ZBX_SERVER_A_IP}};{{ZBX_SERVER_B_IP}}`) |
| 3 | Database | Synchronous replication + high inter-site latency caps achievable NVPS | Measure RTT (Phase 1); switch to asynchronous if NVPS target exceeds sync ceiling |
| 4 | Proxy Groups | SNMP traps are NOT supported through proxy groups | Use dedicated standalone SNMP trap proxies (`proxy-siteA-snmptrap`, `proxy-siteB-snmptrap`) |
| 5 | Proxy Groups | Proxy group rebalancing can cause brief monitoring gaps during redistribution | Acceptable for most environments; tune `MinOnline` for the group |
| 6 | Kubernetes | `CrashLoopBackOff` pods need custom trigger logic (not in default templates) | Create a custom trigger using the Kubernetes pod status item |
| 7 | KEMP LoadMaster | License is MAC-bound; vMotion can change MAC addresses | Assign static MAC addresses to KEMP VMs in vCenter |
| 8 | KEMP LoadMaster | CARP failover requires VMware port group security policy changes | Ensure `MAC Address Changes` = Accept and `Forged Transmits` = Accept on KEMP port groups |
| 9 | TimescaleDB | Compression policies may delay queries on very old data | Tune compression age based on typical query patterns |
| 10 | Zabbix Frontend | Session persistence required; loss of session during KEMP failover requires re-login | Active Cookie persistence configured on KEMP with 30-minute timeout |

---

## Verification Checkpoint

| # | Check | Command / Location | Expected Result | Pass |
|---|-------|--------------------|-----------------|------|
| 1 | Self-monitoring template linked | Frontend: Host > Templates | "Zabbix server health" linked to both server hosts | [ ] |
| 2 | Internal cache items collecting | Frontend: Monitoring > Latest data (filter: "rcache\|wcache") | Recent values for cache items | [ ] |
| 3 | Process busy items collecting | Frontend: Monitoring > Latest data (filter: "process.*busy") | Recent values for process items | [ ] |
| 4 | External meta-monitoring configured | External monitoring system | HTTPS/TCP checks active for all endpoints listed in Step 2 | [ ] |
| 5 | Log rotation configured | `logrotate -d /etc/logrotate.d/zabbix-server` | No syntax errors | [ ] |
| 6 | Housekeeping settings applied | Frontend: Administration > Housekeeping | History 14d, Trends 365d, Events 365d | [ ] |
| 7 | Minor upgrade procedure documented | Review Step 5a in this document | Rolling upgrade steps documented | [ ] |
| 8 | Major upgrade procedure documented | Review Step 5b in this document | Schema migration steps documented | [ ] |
| 9 | Maintenance window test created | Frontend: Data collection > Maintenance | Test maintenance period visible | [ ] |
| 10 | Known limitations reviewed with team | Team meeting / sign-off | Operations team aware of all limitations | [ ] |

---

## Troubleshooting

### Self-Monitoring Cache Alerts

**Symptom:** Triggers fire for low cache buffer free percentage (`rcache,buffer,pfree < 20%` or `wcache,buffer,pfree < 20%`).

**Resolution:**

1. Identify which cache is constrained:
   ```bash
   zabbix_server -R diaginfo
   # Review the cache utilization section
   ```
2. Increase the relevant cache size in `/etc/zabbix/zabbix_server.conf`:
   ```ini
   # History cache (default 16M)
   HistoryCacheSize=64M

   # Trend cache (default 4M)
   TrendCacheSize=16M

   # Value cache (default 8M)
   ValueCacheSize=64M
   ```
3. Restart the Zabbix server:
   ```bash
   systemctl restart zabbix-server
   ```
4. If process busy alerts fire, increase the number of that process type:
   ```ini
   # Example: increase pollers from default 5
   StartPollers=10

   # Example: increase history syncers
   StartDBSyncers=6
   ```

### Disk Filling on Database VMs

**Symptom:** `/var/lib/pgsql/` approaching full, database performance degrading.

**Resolution:**

1. Check current disk usage:
   ```bash
   df -h /var/lib/pgsql/
   du -sh /var/lib/pgsql/16/data/
   ```
2. Verify housekeeping is running:
   ```bash
   # Check PostgreSQL for recent housekeeping activity
   psql -U zabbix -d zabbix -c "SELECT * FROM pg_stat_user_tables WHERE relname LIKE 'history%' ORDER BY n_tup_del DESC LIMIT 10;"
   ```
3. If using TimescaleDB, verify chunk drops are occurring:
   ```bash
   psql -U zabbix -d zabbix -c "SELECT * FROM timescaledb_information.jobs WHERE proc_name = 'policy_retention';"
   ```
4. Check if WAL files are accumulating (archive lag):
   ```bash
   ls -la /var/lib/pgsql/16/data/pg_wal/ | wc -l
   # If WAL file count is very high, check archive_command status
   psql -U postgres -c "SELECT * FROM pg_stat_archiver;"
   ```
5. Consider reducing history retention if disk cannot be expanded:
   - Reduce numeric history from 14d to 7d
   - Reduce character/text/log history from 7d to 3d
6. As a last resort, add disk space to the VM and extend the filesystem

### Log Rotation Not Working

**Symptom:** Log files grow continuously beyond expected size.

**Resolution:**

1. Verify logrotate configuration syntax:
   ```bash
   logrotate -d /etc/logrotate.d/zabbix-server
   ```
2. Manually force a rotation to test:
   ```bash
   logrotate -f /etc/logrotate.d/zabbix-server
   ```
3. Check that the logrotate cron job is running:
   ```bash
   systemctl status crond
   cat /etc/cron.daily/logrotate
   ```
4. Verify the Zabbix log path matches the logrotate configuration:
   ```bash
   grep "^LogFile" /etc/zabbix/zabbix_server.conf
   # Must match the path in /etc/logrotate.d/zabbix-server
   ```

### Upgrade Schema Migration Fails

**Symptom:** Zabbix server fails to start after a major version upgrade, with database schema errors in the log.

**Resolution:**

1. Check the Zabbix server log for the specific error:
   ```bash
   tail -100 /var/log/zabbix/zabbix_server.log
   ```
2. Verify database connectivity:
   ```bash
   psql -h {{VIP_DB_A}} -p 5432 -U zabbix -d zabbix -c "SELECT 1;"
   ```
3. Verify only ONE Zabbix server node is running the migration (the one with `HANodeName` commented out):
   ```bash
   # All other Zabbix server nodes must be STOPPED
   systemctl status zabbix-server   # Check on all nodes
   ```
4. If the migration partially completed and is now stuck:
   - Check the `dbversion` table: `psql -U zabbix -d zabbix -c "SELECT * FROM dbversion;"`
   - Consult the Zabbix upgrade documentation for the specific version
   - **Do NOT manually modify the schema** — restore from backup if the migration is unrecoverable
5. Restore from the pre-upgrade backup if needed:
   ```bash
   sudo -u postgres pgbackrest --stanza=zabbix-cluster \
     --type=immediate \
     --target-action=promote \
     --delta \
     restore
   ```
