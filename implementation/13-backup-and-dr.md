# Phase 13 — Backup and Disaster Recovery

> **Objective:** Configure pgBackRest for PostgreSQL database backup, etcd snapshots, Zabbix configuration file backup, and document disaster recovery procedures for all failure scenarios.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Phase 4 complete | Patroni cluster healthy with primary and synchronous/asynchronous standby |
| Phase 6 complete | Zabbix Server HA configured and both nodes operational |
| Backup storage available | NFS mount or S3-compatible object storage provisioned per `{{BACKUP_TYPE}}` |
| pgBackRest installed | `dnf install -y pgbackrest` on both PostgreSQL nodes |
| Environment variables | `{{BACKUP_TYPE}}`, `{{BACKUP_RETENTION_FULL}}`, `{{BACKUP_RETENTION_DIFF}}` defined in `00-environment-variables.md` |

> **Note:** Backup configuration can start after Phase 4 (Patroni cluster operational) but should be finalized after Phase 6 (Zabbix Server HA) to ensure all configuration files are captured.

---

## Step 1 — Define Recovery Objectives

Before configuring backups, document the agreed recovery objectives with stakeholders. The following table defines the RPO (Recovery Point Objective) and RTO (Recovery Time Objective) for each failure scenario in this architecture.

| # | Failure Scenario | RPO | RTO | Recovery Method |
|---|------------------|-----|-----|-----------------|
| 1 | Single server failure (Zabbix Server) | 0 | 30-65 seconds | Automatic HA failover; standby promotes |
| 2 | Database failover — synchronous replication | 0 | 15-30 seconds | Patroni auto-promotes sync standby; zero data loss |
| 3 | Database failover — asynchronous replication | < 5 seconds | 15-30 seconds | Patroni auto-promotes async standby; minimal data loss |
| 4 | Full site loss — etcd quorum available | 0 / < 5 seconds | 1-3 minutes | Automatic; Patroni promotes, KEMP re-routes, Zabbix Server activates |
| 5 | Full site loss — etcd quorum lost | 0 / < 5 seconds | 5-15 minutes | Manual intervention; restore etcd quorum, trigger failover |
| 6 | Database corruption — Point-in-Time Recovery | Minutes to hours | 30-60 minutes | pgBackRest PITR to specific timestamp |
| 7 | Full rebuild from backup | Last backup age | 2-4 hours | Reinstall, restore config, restore database from pgBackRest |

> **Important:** Scenarios 1-5 are handled by the HA architecture (Patroni, etcd, KEMP, Zabbix Server HA). Scenarios 6-7 require backup infrastructure configured in this phase.

---

## Step 2 — Configure pgBackRest

pgBackRest provides full, differential, and incremental backups with WAL archiving for point-in-time recovery.

### Option A — NFS Storage

Use this option if `{{BACKUP_TYPE}}` = `nfs`.

#### 2a-1. Mount NFS Storage

On the **primary** PostgreSQL node (`pg-siteA`):

```bash
# Install NFS client utilities
dnf install -y nfs-utils

# Create mount point
mkdir -p {{BACKUP_NFS_MOUNT}}

# Add to /etc/fstab for persistent mount
cat >> /etc/fstab << 'EOF'
{{BACKUP_NFS_SERVER}}:{{BACKUP_NFS_PATH}}  {{BACKUP_NFS_MOUNT}}  nfs  defaults,noatime,nofail  0  0
EOF

# Mount immediately
mount {{BACKUP_NFS_MOUNT}}

# Verify mount
df -h {{BACKUP_NFS_MOUNT}}

# Set ownership for postgres user
chown postgres:postgres {{BACKUP_NFS_MOUNT}}
```

Repeat the NFS mount on the **standby** PostgreSQL node (`pg-siteB`).

#### 2a-2. Create pgBackRest Configuration

Create `/etc/pgbackrest/pgbackrest.conf` on **both** PostgreSQL nodes:

```bash
sudo mkdir -p /etc/pgbackrest
```

```ini
# /etc/pgbackrest/pgbackrest.conf

[global]
repo1-path={{BACKUP_NFS_MOUNT}}
repo1-retention-full={{BACKUP_RETENTION_FULL}}
repo1-retention-diff={{BACKUP_RETENTION_DIFF}}
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass={{BACKUP_ENCRYPTION_KEY}}

process-max=4
start-fast=y
delta=y
compress-type=zst
compress-level=3

log-level-console=info
log-level-file=detail

[zabbix-cluster]
pg1-path=/var/lib/pgsql/16/data
pg1-port=5432
```

> **Warning:** The encryption key must be stored securely. If lost, backups cannot be restored.

### Option B — S3 Storage

Use this option if `{{BACKUP_TYPE}}` = `s3`.

Create `/etc/pgbackrest/pgbackrest.conf` on **both** PostgreSQL nodes:

```bash
sudo mkdir -p /etc/pgbackrest
```

```ini
# /etc/pgbackrest/pgbackrest.conf

[global]
repo1-type=s3
repo1-s3-endpoint={{BACKUP_S3_ENDPOINT}}
repo1-s3-bucket={{BACKUP_S3_BUCKET}}
repo1-s3-key={{BACKUP_S3_KEY}}
repo1-s3-key-secret={{BACKUP_S3_SECRET}}
repo1-s3-region={{BACKUP_S3_REGION}}
repo1-path=/pgbackrest
repo1-retention-full={{BACKUP_RETENTION_FULL}}
repo1-retention-diff={{BACKUP_RETENTION_DIFF}}
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass={{BACKUP_ENCRYPTION_KEY}}

process-max=4
start-fast=y
delta=y
compress-type=zst
compress-level=3

log-level-console=info
log-level-file=detail

[zabbix-cluster]
pg1-path=/var/lib/pgsql/16/data
pg1-port=5432
```

> **Warning:** The encryption key must be stored securely. If lost, backups cannot be restored.

> **Security:** For S3 credentials, consider using IAM instance roles or environment variables instead of embedding keys in the configuration file. If keys are stored in `/etc/pgbackrest/pgbackrest.conf`, ensure the file permissions are `0640` owned by `postgres:postgres`.

```bash
chown postgres:postgres /etc/pgbackrest/pgbackrest.conf
chmod 0640 /etc/pgbackrest/pgbackrest.conf
```

---

## Step 3 — Add Patroni Integration for WAL Archiving

WAL (Write-Ahead Log) archiving is required for point-in-time recovery. Patroni manages the PostgreSQL configuration, so WAL archiving must be configured through Patroni.

Edit `/etc/patroni/patroni.yml` on **both** PostgreSQL nodes. Add the following under the `postgresql.parameters` section:

```yaml
bootstrap:
  dcs:
    postgresql:
      parameters:
        archive_mode: "on"
        archive_command: "pgbackrest --stanza=zabbix-cluster archive-push %p"
      recovery_conf:
        restore_command: "pgbackrest --stanza=zabbix-cluster archive-get %f %p"

  method: pgbackrest
  pgbackrest:
    command: pgbackrest --stanza=zabbix-cluster --delta restore
    keep_existing_recovery_conf: true
    no_params: true

postgresql:
  parameters:
    archive_mode: "on"
    archive_command: "pgbackrest --stanza=zabbix-cluster archive-push %p"
  recovery_conf:
    restore_command: "pgbackrest --stanza=zabbix-cluster archive-get %f %p"
  create_replica_methods:
    - pgbackrest
    - basebackup
  pgbackrest:
    command: pgbackrest --stanza=zabbix-cluster --delta restore
    keep_existing_recovery_conf: true
    no_params: true
```

Apply the configuration change:

```bash
# Reload Patroni to pick up the configuration changes
patronictl reload zabbix-cluster
```

Verify archive mode is active:

```bash
psql -U postgres -c "SHOW archive_mode;"
# Expected: on

psql -U postgres -c "SHOW archive_command;"
# Expected: pgbackrest --stanza=zabbix-cluster archive-push %p
```

---

## Step 4 — Initialize pgBackRest

Run the following commands on the **primary** PostgreSQL node as the `postgres` user.

### 4a. Create the Stanza

```bash
sudo -u postgres pgbackrest --stanza=zabbix-cluster stanza-create
```

### 4b. Verify Stanza Configuration

```bash
sudo -u postgres pgbackrest --stanza=zabbix-cluster check
```

Expected output: `check command end: completed successfully`

### 4c. Run First Full Backup

```bash
sudo -u postgres pgbackrest --stanza=zabbix-cluster --type=full backup
```

This initial full backup may take significant time depending on database size. Monitor progress in the console output.

### 4d. Verify Backup Completed

```bash
sudo -u postgres pgbackrest info
```

Expected output shows the `zabbix-cluster` stanza with one full backup listed, including the timestamp, size, and WAL archive range.

---

## Step 5 — Configure Backup Schedule

Create cron jobs on the **primary** PostgreSQL node. pgBackRest is safe to run on the primary; it uses non-blocking checksums and `start-fast=y` to minimize impact.

### 5a. Create Backup Cron Jobs

```bash
cat > /etc/cron.d/pgbackrest-backup << 'EOF'
# pgBackRest backup schedule for Zabbix database
# Full backup: Sundays at 01:00
0 1 * * 0  postgres  pgbackrest --stanza=zabbix-cluster --type=full backup >> /var/log/pgbackrest/cron-full.log 2>&1

# Differential backup: Monday through Saturday at 01:00
0 1 * * 1-6  postgres  pgbackrest --stanza=zabbix-cluster --type=diff backup >> /var/log/pgbackrest/cron-diff.log 2>&1
EOF
```

### 5b. Create Weekly Verification Cron Job

```bash
cat > /etc/cron.d/pgbackrest-verify << 'EOF'
# pgBackRest weekly verification — Sundays at 06:00 (after full backup completes)
0 6 * * 0  postgres  pgbackrest --stanza=zabbix-cluster info >> /var/log/pgbackrest/cron-verify.log 2>&1
0 7 * * 0  postgres  pgbackrest --stanza=zabbix-cluster verify >> /var/log/pgbackrest/cron-verify.log 2>&1
EOF
```

### 5c. Create Log Directory

```bash
mkdir -p /var/log/pgbackrest
chown postgres:postgres /var/log/pgbackrest
```

---

## Step 6 — etcd Cluster Backup

etcd stores the Patroni DCS state (leader election, cluster configuration). While etcd data can be reconstructed, having snapshots dramatically speeds DR recovery.

### 6a. Create Backup Directory

On **each** etcd node:

```bash
mkdir -p /var/lib/etcd/backup
chown etcd:etcd /var/lib/etcd/backup
```

### 6b. Create etcd Snapshot Script

Create `/usr/local/bin/etcd-backup.sh` on **one** etcd node (e.g., `etcd-1`):

```bash
#!/bin/bash
# etcd cluster snapshot backup
# Saves a snapshot and retains the last 7 days

set -euo pipefail

BACKUP_DIR="/var/lib/etcd/backup"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
SNAPSHOT_FILE="${BACKUP_DIR}/etcd-snapshot-${TIMESTAMP}.db"

# Create snapshot with TLS authentication
ETCDCTL_API=3 etcdctl snapshot save "${SNAPSHOT_FILE}" \
  --endpoints=https://{{ETCD_1_IP}}:2379 \
  --cacert={{ETCD_CA_CERT_PATH}} \
  --cert=/etc/etcd/pki/etcd-1.crt \
  --key=/etc/etcd/pki/etcd-1.key

# Verify the snapshot
ETCDCTL_API=3 etcdctl snapshot status "${SNAPSHOT_FILE}" --write-out=table

# Remove snapshots older than 7 days
find "${BACKUP_DIR}" -name "etcd-snapshot-*.db" -type f -mtime +7 -delete

echo "etcd backup completed: ${SNAPSHOT_FILE}"
```

Set permissions:

```bash
chmod 750 /usr/local/bin/etcd-backup.sh
chown root:etcd /usr/local/bin/etcd-backup.sh
```

### 6c. Schedule Daily etcd Backup

```bash
cat > /etc/cron.d/etcd-backup << 'EOF'
# etcd daily snapshot — 02:00
0 2 * * *  root  /usr/local/bin/etcd-backup.sh >> /var/log/etcd-backup.log 2>&1
EOF
```

### 6d. etcd Snapshot Restore Procedure (DR Reference)

The following procedure restores an etcd cluster from a snapshot. **Use only during disaster recovery.**

```bash
# 1. Stop etcd on ALL nodes
systemctl stop etcd    # Run on etcd-1, etcd-2, and etcd-3

# 2. Restore the snapshot on each node (adjust --name and --initial-advertise-peer-urls per node)
# On etcd-1:
ETCDCTL_API=3 etcdctl snapshot restore /var/lib/etcd/backup/<snapshot-file>.db \
  --name etcd-1 \
  --data-dir /var/lib/etcd/default.etcd \
  --initial-cluster "etcd-1=https://{{ETCD_1_IP}}:2380,etcd-2=https://{{ETCD_2_IP}}:2380,etcd-3=https://{{ETCD_3_IP}}:2380" \
  --initial-advertise-peer-urls https://{{ETCD_1_IP}}:2380

# On etcd-2:
ETCDCTL_API=3 etcdctl snapshot restore /var/lib/etcd/backup/<snapshot-file>.db \
  --name etcd-2 \
  --data-dir /var/lib/etcd/default.etcd \
  --initial-cluster "etcd-1=https://{{ETCD_1_IP}}:2380,etcd-2=https://{{ETCD_2_IP}}:2380,etcd-3=https://{{ETCD_3_IP}}:2380" \
  --initial-advertise-peer-urls https://{{ETCD_2_IP}}:2380

# On etcd-3:
ETCDCTL_API=3 etcdctl snapshot restore /var/lib/etcd/backup/<snapshot-file>.db \
  --name etcd-3 \
  --data-dir /var/lib/etcd/default.etcd \
  --initial-cluster "etcd-1=https://{{ETCD_1_IP}}:2380,etcd-2=https://{{ETCD_2_IP}}:2380,etcd-3=https://{{ETCD_3_IP}}:2380" \
  --initial-advertise-peer-urls https://{{ETCD_3_IP}}:2380

# 3. Start etcd on ALL nodes
systemctl start etcd   # Run on etcd-1, etcd-2, and etcd-3

# 4. Verify cluster health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://{{ETCD_1_IP}}:2379,https://{{ETCD_2_IP}}:2379,https://{{ETCD_3_IP}}:2379 \
  --cacert={{ETCD_CA_CERT_PATH}} \
  --cert=/etc/etcd/pki/etcd-1.crt \
  --key=/etc/etcd/pki/etcd-1.key
```

---

## Step 7 — Zabbix Configuration File Backup

Back up all configuration files that are not stored in the database. These are required for rebuilding servers from scratch.

### 7a. Files to Back Up

| Path | Node(s) | Description |
|------|---------|-------------|
| `/etc/zabbix/` | Zabbix Servers, Frontends, Proxies, Agents | All Zabbix configuration files |
| `/etc/zabbix/zabbix_server.d/*.psk` | Zabbix Servers | Pre-shared key files for proxy encryption |
| `/usr/lib/zabbix/alertscripts/` | Zabbix Servers | Custom alert scripts (Teams, Slack, PagerDuty) |
| `/usr/lib/zabbix/externalscripts/` | Zabbix Servers | External check scripts |
| `/etc/patroni/` | PostgreSQL nodes | Patroni cluster configuration |
| `/etc/pgbackrest/pgbackrest.conf` | PostgreSQL nodes | pgBackRest configuration |
| `/etc/etcd/` | etcd nodes | etcd configuration and TLS certificates |
| `/etc/etcd/pki/` | etcd nodes | etcd TLS certificate and key files |
| `{{SSL_CERT_PATH}}`, `{{SSL_KEY_PATH}}`, `{{SSL_CA_PATH}}` | Management workstation or KEMP | SSL certificates for frontend |

### 7b. Create Configuration Backup Script

Create `/usr/local/bin/zabbix-config-backup.sh` on a management host or each respective server:

```bash
#!/bin/bash
# Zabbix infrastructure configuration backup
# Run from a central management host with SSH access to all nodes

set -euo pipefail

BACKUP_BASE="/var/backups/zabbix-config"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="${BACKUP_BASE}/${TIMESTAMP}"

mkdir -p "${BACKUP_DIR}"

# Local server config backup (run on each server locally, or use rsync/scp from central host)
# Example using rsync from a central management host:

# Zabbix Server configs
rsync -az --relative \
  /etc/zabbix/ \
  /usr/lib/zabbix/alertscripts/ \
  /usr/lib/zabbix/externalscripts/ \
  "${BACKUP_DIR}/zabbix-server/"

# Patroni configs
rsync -az --relative \
  /etc/patroni/ \
  /etc/pgbackrest/ \
  "${BACKUP_DIR}/patroni/"

# etcd configs (including TLS certs)
rsync -az --relative \
  /etc/etcd/ \
  "${BACKUP_DIR}/etcd/"

# Create compressed archive
tar -czf "${BACKUP_BASE}/zabbix-config-${TIMESTAMP}.tar.gz" -C "${BACKUP_BASE}" "${TIMESTAMP}"

# Remove uncompressed directory
rm -rf "${BACKUP_DIR}"

# Retain last 30 days of config backups
find "${BACKUP_BASE}" -name "zabbix-config-*.tar.gz" -type f -mtime +30 -delete

echo "Configuration backup completed: ${BACKUP_BASE}/zabbix-config-${TIMESTAMP}.tar.gz"
```

Set permissions:

```bash
chmod 750 /usr/local/bin/zabbix-config-backup.sh
mkdir -p /var/backups/zabbix-config
```

### 7c. Schedule Configuration Backup

```bash
cat > /etc/cron.d/zabbix-config-backup << 'EOF'
# Zabbix configuration file backup — daily at 03:00
0 3 * * *  root  /usr/local/bin/zabbix-config-backup.sh >> /var/log/zabbix-config-backup.log 2>&1
EOF
```

> **Note:** Adapt this script to your environment. If using a centralized backup agent (Veeam, Commvault, etc.), configure it to capture the paths listed in Step 7a instead of using this script.

---

## Step 8 — Document DR Procedures

### DR Scenario 1 — Primary Site Lost (Automatic Failover)

**Condition:** Site A is completely lost. etcd quorum is maintained (etcd-2 at Site B + etcd-3 at Site B or third location = 2 of 3 members available).

**Automatic actions (no operator intervention required):**

1. **Patroni** detects the primary (`pg-siteA`) is unreachable. After the TTL expires (~30 seconds), Patroni promotes `pg-siteB` to primary.
2. **KEMP** at Site B: The PostgreSQL Virtual Service health check (`/primary` on port 8008) detects `pg-siteB` now returns HTTP 200. Traffic routes to the new primary.
3. **Zabbix Server HA**: `zabbix-node-siteB` detects `zabbix-node-siteA` has missed heartbeats. After `ha_failover_delay` (~30 seconds), it promotes itself to active.
4. **Proxies** at Site B continue operating normally. Proxies at Site A buffer data locally until Site A is restored.

**Manual verification after automatic failover:**

```bash
# Verify Patroni promotion
patronictl -c /etc/patroni/patroni.yml list
# Expected: pg-siteB = Leader, pg-siteA = unknown/unreachable

# Verify Zabbix Server HA status
zabbix_server -R ha_status
# Expected: zabbix-node-siteB = active, zabbix-node-siteA = unavailable

# Verify frontend accessibility
curl -sk -o /dev/null -w "%{http_code}" https://{{VIP_FRONTEND_B}}
# Expected: 200
```

**If using DNS-based frontend access (Option A):** Update `zabbix.{{DOMAIN}}` DNS record to point to `{{VIP_FRONTEND_B}}`. Users accessing `zabbix.{{DOMAIN}}` will be redirected within `{{DNS_TTL}}` seconds.

---

#### DR Scenario 1b: Primary Site Lost — No etcd Quorum

If etcd-3 is at Site A (both etcd-1 and etcd-3 lost), Site B has only 1 of 3 etcd members and **cannot form quorum**. Patroni will NOT auto-promote.

**Manual intervention required:**

```bash
# 1. Verify etcd quorum is lost
etcdctl endpoint health --endpoints=https://{{ETCD_2_IP}}:2379 \
  --cacert={{ETCD_CA_CERT_PATH}}
# Expected: unhealthy (no quorum)

# 2. Force Patroni failover (ONLY if certain Site A will not return)
patronictl -c /etc/patroni/patroni.yml failover zabbix-cluster \
  --candidate pg-siteB --force

# 3. Update DNS if using Option A strategy
# zabbix.{{DOMAIN}} -> {{VIP_FRONTEND_B}}

# 4. Notify: Site A agents/proxies are offline until rebuilt
```

> **Recommendation:** Place etcd-3 at Site B to avoid this scenario entirely.

---

### DR Scenario 2 — Database Corruption (Point-in-Time Recovery)

**Condition:** Data corruption is detected in the Zabbix database. Both primary and replica contain corrupted data (replication propagated the corruption).

**Procedure:**

```bash
# 1. Identify the timestamp BEFORE corruption occurred
#    Check Zabbix server logs, application logs, or user reports to determine
#    the last known good timestamp
#    Example: "2025-03-15 14:30:00"

# 2. Stop Patroni on ALL nodes to prevent automatic recovery
systemctl stop patroni    # Run on pg-siteA
systemctl stop patroni    # Run on pg-siteB

# 3. Remove the cluster state from etcd (allows fresh initialization)
ETCDCTL_API=3 etcdctl del /service/zabbix-cluster --prefix \
  --endpoints=https://{{ETCD_1_IP}}:2379 \
  --cacert={{ETCD_CA_CERT_PATH}} \
  --cert=/etc/etcd/pki/etcd-1.crt \
  --key=/etc/etcd/pki/etcd-1.key \
  --user root:{{ETCD_ROOT_PASSWORD}}

# 4. Restore the database to the target timestamp on the PRIMARY node (pg-siteA)
sudo -u postgres pgbackrest --stanza=zabbix-cluster \
  --type=time \
  --target="2025-03-15 14:30:00" \
  --target-action=promote \
  --delta \
  restore

# 5. Restart Patroni on the restored node FIRST
systemctl start patroni   # Run on pg-siteA (restored node)

# 6. Wait for Patroni to initialize and claim leadership
patronictl -c /etc/patroni/patroni.yml list
# Wait until pg-siteA shows as Leader

# 7. Clear the data directory on the standby and restart Patroni
#    Patroni will automatically create a new replica from the restored primary
systemctl stop patroni    # Run on pg-siteB (if not already stopped)
rm -rf /var/lib/pgsql/16/data/*   # Run on pg-siteB
systemctl start patroni   # Run on pg-siteB

# 8. Monitor replica rebuild
patronictl -c /etc/patroni/patroni.yml list
# Wait until pg-siteB shows as Sync Standby or Async Standby
```

> **Warning:** PITR restores data to the specified timestamp. All data written after that timestamp is permanently lost. Verify the target timestamp carefully with stakeholders before proceeding.

---

### DR Scenario 3 — Rebuild Failed Site

**Condition:** Site A hardware is replaced or VMs are recreated after a site failure. Site B is currently running as primary.

**Procedure:**

1. **Reinstall the operating system** on all Site A VMs (Rocky Linux 9)
2. **Install packages** following Phase 2 (PostgreSQL, Zabbix, etcd as applicable)
3. **Restore configuration files** from the config backup archive:
   ```bash
   # Extract the latest config backup
   tar -xzf /var/backups/zabbix-config/zabbix-config-<latest>.tar.gz -C /tmp/restore/

   # Copy configs to their proper locations
   cp -a /tmp/restore/zabbix-server/etc/zabbix/* /etc/zabbix/
   cp -a /tmp/restore/patroni/etc/patroni/* /etc/patroni/
   cp -a /tmp/restore/etcd/etc/etcd/* /etc/etcd/
   ```
4. **PostgreSQL / Patroni**: Clear the data directory and start Patroni. It will automatically create a replica from the current primary at Site B:
   ```bash
   rm -rf /var/lib/pgsql/16/data/*
   systemctl start patroni
   # Patroni uses pgbackrest or basebackup to create the replica
   ```
5. **etcd**: Rejoin the etcd cluster:
   ```bash
   # Add the member back from an existing etcd node
   ETCDCTL_API=3 etcdctl member add etcd-1 \
     --peer-urls=https://{{ETCD_1_IP}}:2380 \
     --endpoints=https://{{ETCD_2_IP}}:2379 \
     --cacert={{ETCD_CA_CERT_PATH}} \
     --cert=/etc/etcd/pki/etcd-2.crt \
     --key=/etc/etcd/pki/etcd-2.key

   # Start etcd on the rebuilt node
   systemctl start etcd
   ```
6. **Zabbix Server**: Start the Zabbix server. It will automatically join HA as standby:
   ```bash
   systemctl start zabbix-server
   ```
7. **Verify full resynchronization**:
   ```bash
   patronictl -c /etc/patroni/patroni.yml list
   # pg-siteA should show as Sync Standby or Async Standby

   zabbix_server -R ha_status
   # zabbix-node-siteA should show as standby

   ETCDCTL_API=3 etcdctl endpoint health --cluster \
     --endpoints=https://{{ETCD_1_IP}}:2379 \
     --cacert={{ETCD_CA_CERT_PATH}} \
     --cert=/etc/etcd/pki/etcd-1.crt \
     --key=/etc/etcd/pki/etcd-1.key
   ```

---

## Step 9 — Enable DCS Failsafe Mode

If not already enabled in Phase 4, enable the Patroni DCS failsafe mode. This prevents Patroni from demoting the primary if all etcd nodes become unreachable simultaneously (network partition protecting the database from unnecessary demotion).

```bash
patronictl -c /etc/patroni/patroni.yml edit-config -s failsafe_mode=true
```

Verify the setting:

```bash
patronictl -c /etc/patroni/patroni.yml show-config | grep failsafe
# Expected: failsafe_mode: true
```

> **Rationale:** Without failsafe mode, a total etcd outage causes Patroni to demote the primary (because it cannot confirm it is still the leader). With failsafe mode enabled, the primary continues operating in a degraded state until etcd recovers, preventing unnecessary downtime.

---

## Verification Checkpoint

| # | Check | Command | Expected Result | Pass |
|---|-------|---------|-----------------|------|
| 1 | First full backup completed | `sudo -u postgres pgbackrest info` | Full backup listed with timestamp and size | [ ] |
| 2 | pgBackRest stanza is healthy | `sudo -u postgres pgbackrest --stanza=zabbix-cluster check` | `completed successfully` | [ ] |
| 3 | pgBackRest verify passes | `sudo -u postgres pgbackrest --stanza=zabbix-cluster verify` | No errors reported | [ ] |
| 4 | WAL archiving active | `psql -U postgres -c "SELECT * FROM pg_stat_archiver;"` | `archived_count` > 0, `last_failed_wal` is empty | [ ] |
| 5 | etcd snapshot created | `ls -la /var/lib/etcd/backup/` | At least one `.db` snapshot file present | [ ] |
| 6 | etcd snapshot verified | `etcdctl snapshot status /var/lib/etcd/backup/<latest>.db` | Table output shows hash, revision, total keys | [ ] |
| 7 | Config files backed up | `ls -la /var/backups/zabbix-config/` | At least one `.tar.gz` archive present | [ ] |
| 8 | Backup cron jobs installed | `cat /etc/cron.d/pgbackrest-backup` | Full on Sunday, diff Mon-Sat at 01:00 | [ ] |
| 9 | etcd backup cron installed | `cat /etc/cron.d/etcd-backup` | Daily at 02:00 | [ ] |
| 10 | DCS failsafe mode enabled | `patronictl show-config \| grep failsafe` | `failsafe_mode: true` | [ ] |
| 11 | DR procedures documented | Review this document | Scenarios 1-3 documented with step-by-step commands | [ ] |

---

## Troubleshooting

### pgBackRest Stanza-Create Fails

**Symptom:** `pgbackrest --stanza=zabbix-cluster stanza-create` returns a permission error or path not found.

**Resolution:**

1. Verify the backup repository path exists and is writable:
   ```bash
   # NFS
   ls -la {{BACKUP_NFS_MOUNT}}
   touch {{BACKUP_NFS_MOUNT}}/test && rm {{BACKUP_NFS_MOUNT}}/test

   # S3
   pgbackrest --stanza=zabbix-cluster repo-ls
   ```
2. Ensure the command is run as the `postgres` user:
   ```bash
   sudo -u postgres pgbackrest --stanza=zabbix-cluster stanza-create
   ```
3. Verify `pg1-path` in `/etc/pgbackrest/pgbackrest.conf` matches the actual PostgreSQL data directory:
   ```bash
   psql -U postgres -c "SHOW data_directory;"
   ```
4. Confirm PostgreSQL is running and accessible on `pg1-port`:
   ```bash
   pg_isready -p 5432
   ```

### NFS Mount Issues

**Symptom:** Mount fails or backup hangs during write operations.

**Resolution:**

1. Verify the NFS server is reachable:
   ```bash
   showmount -e {{BACKUP_NFS_SERVER}}
   ```
2. Check that the NFS export path exists and allows write access from the PostgreSQL node IPs:
   ```bash
   # On the NFS server, check /etc/exports
   exportfs -v
   ```
3. If mounts hang, check NFS version compatibility and firewall rules (TCP 2049):
   ```bash
   nc -zv {{BACKUP_NFS_SERVER}} 2049
   ```
4. Test with a specific NFS version if needed:
   ```bash
   mount -t nfs -o vers=4.1 {{BACKUP_NFS_SERVER}}:{{BACKUP_NFS_PATH}} {{BACKUP_NFS_MOUNT}}
   ```

### S3 Authentication Errors

**Symptom:** pgBackRest fails with S3 authorization or access denied errors.

**Resolution:**

1. Verify S3 credentials are correct:
   ```bash
   # Test connectivity to the S3 endpoint
   curl -I https://{{BACKUP_S3_ENDPOINT}}
   ```
2. Check that the bucket exists and the credentials have read/write access:
   ```bash
   # If AWS CLI is available
   aws s3 ls s3://{{BACKUP_S3_BUCKET}} --endpoint-url https://{{BACKUP_S3_ENDPOINT}}
   ```
3. Verify the region is correct in `/etc/pgbackrest/pgbackrest.conf`
4. If using a non-AWS S3-compatible service, confirm `repo1-s3-uri-style` is set correctly (path-style vs virtual-hosted):
   ```ini
   # Add to [global] section if using path-style URLs (MinIO, Ceph, etc.)
   repo1-s3-uri-style=path
   ```

### etcd Snapshot Fails

**Symptom:** `etcdctl snapshot save` returns a connection or TLS error.

**Resolution:**

1. Verify the TLS certificate paths are correct in the backup script:
   ```bash
   ls -la {{ETCD_CA_CERT_PATH}}
   ls -la /etc/etcd/pki/etcd-1.crt
   ls -la /etc/etcd/pki/etcd-1.key
   ```
2. Test etcd connectivity manually:
   ```bash
   ETCDCTL_API=3 etcdctl endpoint health \
     --endpoints=https://{{ETCD_1_IP}}:2379 \
     --cacert={{ETCD_CA_CERT_PATH}} \
     --cert=/etc/etcd/pki/etcd-1.crt \
     --key=/etc/etcd/pki/etcd-1.key
   ```
3. Ensure the backup directory exists and is writable:
   ```bash
   ls -la /var/lib/etcd/backup/
   ```
4. Check that the `etcdctl` binary is installed and in the PATH:
   ```bash
   which etcdctl
   etcdctl version
   ```

### WAL Archiving Not Working

**Symptom:** `pg_stat_archiver` shows `last_failed_wal` is populated or `archived_count` is not increasing.

**Resolution:**

1. Check the archive status:
   ```bash
   psql -U postgres -c "SELECT * FROM pg_stat_archiver;"
   ```
2. Verify the archive command can run manually:
   ```bash
   sudo -u postgres pgbackrest --stanza=zabbix-cluster archive-push /var/lib/pgsql/16/data/pg_wal/<wal-file>
   ```
3. Check pgBackRest logs for detailed errors:
   ```bash
   ls -lt /var/log/pgbackrest/
   cat /var/log/pgbackrest/zabbix-cluster-archive-push-async.log
   ```
4. Ensure Patroni applied the `archive_command` setting (not overridden):
   ```bash
   psql -U postgres -c "SHOW archive_command;"
   ```
