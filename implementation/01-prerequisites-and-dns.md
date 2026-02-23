# Phase 1 — Prerequisites and DNS

## Objective

Provision all virtual machines, create DNS records, verify NTP synchronization, and confirm cross-site connectivity. This phase establishes the infrastructure foundation that all subsequent phases depend on.

## Prerequisites

- [ ] Phase 0 complete — all values in `00-environment-variables.md` filled in
- [ ] VMware vSphere clusters available at both Site A and Site B
- [ ] IP addresses allocated per the environment variables catalog
- [ ] DNS admin access (or change requests submitted with sufficient lead time)
- [ ] Firewall change requests submitted (refer to Appendix A for full port matrix)
- [ ] Rocky Linux 9 ISO or template available in vSphere content library

---

## Step 1 — VM Provisioning Checklist

Deploy the following VMs at **Site A**. Use thin provisioning unless noted otherwise.

| VM Name | Role | vCPU | RAM | Storage | IP Address | Notes |
|---------|------|------|-----|---------|------------|-------|
| `zabbix-node-siteA` | Zabbix Server | 8 | 32 GB | 100 GB | `{{ZBX_SERVER_A_IP}}` | |
| `pg-siteA` | PostgreSQL + TimescaleDB | 8 | 32 GB | 500 GB SSD | `{{PG_A_IP}}` | SSD/NVMe datastore required |
| `etcd-1` | etcd cluster member | 2 | 4 GB | 20 GB SSD | `{{ETCD_1_IP}}` | SSD datastore required |
| `etcd-3` | etcd arbitrator | 2 | 4 GB | 20 GB SSD | `{{ETCD_3_IP}}` | SSD datastore; place at Site B or third location if automatic DR failover is required |
| `frontend-siteA` | Zabbix Frontend | 2 | 4 GB | 20 GB | `{{FRONTEND_A_IP}}` | |
| `kemp-a1` | KEMP VLM (Active) | 2 | 4 GB | 32 GB **thick** | `{{KEMP_A1_IP}}` | Deploy from OVF; VMXNET3 NICs; static MAC |
| `kemp-a2` | KEMP VLM (Passive) | 2 | 4 GB | 32 GB **thick** | `{{KEMP_A2_IP}}` | Deploy from OVF; VMXNET3 NICs; static MAC |
| `proxy-siteA-01` | Zabbix Proxy | 4 | 8 GB | 50 GB | `{{PROXY_A1_IP}}` | |
| `proxy-siteA-02` | Zabbix Proxy | 4 | 8 GB | 50 GB | `{{PROXY_A2_IP}}` | |
| `proxy-siteA-snmptrap` | SNMP Trap Proxy | 2 | 4 GB | 20 GB | `{{PROXY_A_SNMPTRAP_IP}}` | Standalone; not in proxy group |

Deploy the following VMs at **Site B**:

| VM Name | Role | vCPU | RAM | Storage | IP Address | Notes |
|---------|------|------|-----|---------|------------|-------|
| `zabbix-node-siteB` | Zabbix Server | 8 | 32 GB | 100 GB | `{{ZBX_SERVER_B_IP}}` | |
| `pg-siteB` | PostgreSQL + TimescaleDB | 8 | 32 GB | 500 GB SSD | `{{PG_B_IP}}` | SSD/NVMe datastore required |
| `etcd-2` | etcd cluster member | 2 | 4 GB | 20 GB SSD | `{{ETCD_2_IP}}` | SSD datastore required |
| `frontend-siteB` | Zabbix Frontend | 2 | 4 GB | 20 GB | `{{FRONTEND_B_IP}}` | |
| `kemp-b1` | KEMP VLM (Active) | 2 | 4 GB | 32 GB **thick** | `{{KEMP_B1_IP}}` | Deploy from OVF; VMXNET3 NICs; static MAC |
| `kemp-b2` | KEMP VLM (Passive) | 2 | 4 GB | 32 GB **thick** | `{{KEMP_B2_IP}}` | Deploy from OVF; VMXNET3 NICs; static MAC |
| `proxy-siteB-01` | Zabbix Proxy | 4 | 8 GB | 50 GB | `{{PROXY_B1_IP}}` | |
| `proxy-siteB-02` | Zabbix Proxy | 4 | 8 GB | 50 GB | `{{PROXY_B2_IP}}` | |
| `proxy-siteB-snmptrap` | SNMP Trap Proxy | 2 | 4 GB | 20 GB | `{{PROXY_B_SNMPTRAP_IP}}` | Standalone; not in proxy group |

**Combined totals:** 19-21 VMs, 70-74 vCPU, 204-212 GB RAM, ~1.7 TB storage.

> **Note:** If etcd-3 (arbitrator) is placed at Site B instead of Site A, adjust the IP address to a Site B allocation and update `{{ETCD_3_IP}}` accordingly. Placing etcd-3 at Site B or a third location enables automatic DR failover when Site A is lost.

---

## Step 2 — DNS Record Creation

Create the following DNS A records. Use the FQDN format `<hostname>.{{DOMAIN}}` for all records.

### Frontend and VIP Records

| # | DNS Record | Type | Value | Purpose |
|---|-----------|------|-------|---------|
| 1 | `zabbix.{{DOMAIN}}` | A | `{{VIP_FRONTEND_A}}` | Primary user-facing frontend URL |
| 2 | `zabbix-b.{{DOMAIN}}` | A | `{{VIP_FRONTEND_B}}` | Secondary / DR frontend URL |
| 3 | `db-vip-a.{{DOMAIN}}` | A | `{{VIP_DB_A}}` | Database VIP — Site A KEMP |
| 4 | `db-vip-b.{{DOMAIN}}` | A | `{{VIP_DB_B}}` | Database VIP — Site B KEMP |

### Zabbix Server Records

| # | DNS Record | Type | Value | Purpose |
|---|-----------|------|-------|---------|
| 5 | `zabbix-node-siteA.{{DOMAIN}}` | A | `{{ZBX_SERVER_A_IP}}` | Zabbix Server HA Node 1 |
| 6 | `zabbix-node-siteB.{{DOMAIN}}` | A | `{{ZBX_SERVER_B_IP}}` | Zabbix Server HA Node 2 |

### Proxy Records

| # | DNS Record | Type | Value | Purpose |
|---|-----------|------|-------|---------|
| 7 | `proxy-siteA-01.{{DOMAIN}}` | A | `{{PROXY_A1_IP}}` | Zabbix Proxy — Site A #1 |
| 8 | `proxy-siteA-02.{{DOMAIN}}` | A | `{{PROXY_A2_IP}}` | Zabbix Proxy — Site A #2 |
| 9 | `proxy-siteB-01.{{DOMAIN}}` | A | `{{PROXY_B1_IP}}` | Zabbix Proxy — Site B #1 |
| 10 | `proxy-siteB-02.{{DOMAIN}}` | A | `{{PROXY_B2_IP}}` | Zabbix Proxy — Site B #2 |
| 11 | `proxy-siteA-snmptrap.{{DOMAIN}}` | A | `{{PROXY_A_SNMPTRAP_IP}}` | SNMP trap receiver — Site A |
| 12 | `proxy-siteB-snmptrap.{{DOMAIN}}` | A | `{{PROXY_B_SNMPTRAP_IP}}` | SNMP trap receiver — Site B |

### Database Records

| # | DNS Record | Type | Value | Purpose |
|---|-----------|------|-------|---------|
| 13 | `pg-siteA.{{DOMAIN}}` | A | `{{PG_A_IP}}` | PostgreSQL node — Site A |
| 14 | `pg-siteB.{{DOMAIN}}` | A | `{{PG_B_IP}}` | PostgreSQL node — Site B |

### etcd Records

| # | DNS Record | Type | Value | Purpose |
|---|-----------|------|-------|---------|
| 15 | `etcd-1.{{DOMAIN}}` | A | `{{ETCD_1_IP}}` | etcd cluster member — Site A |
| 16 | `etcd-2.{{DOMAIN}}` | A | `{{ETCD_2_IP}}` | etcd cluster member — Site B |
| 17 | `etcd-3.{{DOMAIN}}` | A | `{{ETCD_3_IP}}` | etcd arbitrator |

### Frontend Records

| # | DNS Record | Type | Value | Purpose |
|---|-----------|------|-------|---------|
| 18 | `frontend-siteA.{{DOMAIN}}` | A | `{{FRONTEND_A_IP}}` | Zabbix Frontend — Site A |
| 19 | `frontend-siteB.{{DOMAIN}}` | A | `{{FRONTEND_B_IP}}` | Zabbix Frontend — Site B |

### KEMP LoadMaster Records

| # | DNS Record | Type | Value | Purpose |
|---|-----------|------|-------|---------|
| 20 | `kemp-a1.{{DOMAIN}}` | A | `{{KEMP_A1_IP}}` | KEMP VLM-A1 management (Active) |
| 21 | `kemp-a2.{{DOMAIN}}` | A | `{{KEMP_A2_IP}}` | KEMP VLM-A2 management (Passive) |
| 22 | `kemp-b1.{{DOMAIN}}` | A | `{{KEMP_B1_IP}}` | KEMP VLM-B1 management (Active) |
| 23 | `kemp-b2.{{DOMAIN}}` | A | `{{KEMP_B2_IP}}` | KEMP VLM-B2 management (Passive) |

**Total: 23 DNS records.**

---

## Step 3 — Set DNS TTL

Set the TTL to `{{DNS_TTL}}` seconds (recommended: **60 seconds**) for the following records that participate in failover:

- `zabbix.{{DOMAIN}}`
- `zabbix-b.{{DOMAIN}}`
- `db-vip-a.{{DOMAIN}}`
- `db-vip-b.{{DOMAIN}}`
- `zabbix-node-siteA.{{DOMAIN}}`
- `zabbix-node-siteB.{{DOMAIN}}`
- All `proxy-*` records

Infrastructure and management records (etcd, pg, frontend, kemp) can use a standard TTL (e.g., 300-3600 seconds) since they point to static IPs that do not change during failover.

> **Rationale:** Low TTL on failover-participating records ensures DNS clients pick up changes quickly during a site failover event. A 60-second TTL means clients will resolve the new IP within 1-2 minutes.

---

## Step 4 — Verify DNS Resolution

Run the following from **every provisioned VM** to confirm forward DNS resolution. Replace the example domain with your actual `{{DOMAIN}}`.

```bash
# Verify all frontend/VIP records
for RECORD in \
  zabbix.{{DOMAIN}} \
  zabbix-b.{{DOMAIN}} \
  db-vip-a.{{DOMAIN}} \
  db-vip-b.{{DOMAIN}}; do
  echo "--- $RECORD ---"
  dig +short "$RECORD"
done

# Verify all infrastructure records
for RECORD in \
  zabbix-node-siteA.{{DOMAIN}} \
  zabbix-node-siteB.{{DOMAIN}} \
  proxy-siteA-01.{{DOMAIN}} \
  proxy-siteA-02.{{DOMAIN}} \
  proxy-siteB-01.{{DOMAIN}} \
  proxy-siteB-02.{{DOMAIN}} \
  proxy-siteA-snmptrap.{{DOMAIN}} \
  proxy-siteB-snmptrap.{{DOMAIN}} \
  pg-siteA.{{DOMAIN}} \
  pg-siteB.{{DOMAIN}} \
  etcd-1.{{DOMAIN}} \
  etcd-2.{{DOMAIN}} \
  etcd-3.{{DOMAIN}} \
  frontend-siteA.{{DOMAIN}} \
  frontend-siteB.{{DOMAIN}} \
  kemp-a1.{{DOMAIN}} \
  kemp-a2.{{DOMAIN}} \
  kemp-b1.{{DOMAIN}} \
  kemp-b2.{{DOMAIN}}; do
  echo "--- $RECORD ---"
  dig +short "$RECORD"
done
```

Alternatively, use `nslookup` if `dig` is not yet installed:

```bash
nslookup zabbix.{{DOMAIN}}
nslookup db-vip-a.{{DOMAIN}}
# ... repeat for each record
```

**Expected result:** Each record resolves to the IP address listed in Step 2. If any record fails to resolve, do not proceed until DNS is corrected.

---

## Step 5 — Verify NTP Synchronization

Time synchronization is **critical** for etcd elections, Patroni leader TTLs, Zabbix HA heartbeats, and TLS certificate validation. Clock drift greater than 1 second can cause split-brain or false failovers.

### 5a. Install and Configure chrony on ALL VMs

```bash
dnf install -y chrony
```

### 5b. Configure Internal NTP Servers

Edit `/etc/chrony.conf` on each VM. Replace the default pool lines:

```ini
# /etc/chrony.conf
# Remove or comment out any default "pool" or "server" lines, then add:
server {{NTP_SERVER_1}} iburst
server {{NTP_SERVER_2}} iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
```

### 5c. Enable and Start chrony

```bash
systemctl enable --now chronyd
```

### 5d. Verify Synchronization

```bash
# Check synchronization status
chronyc tracking

# Check configured sources and their status
chronyc sources -v
```

**Expected output from `chronyc tracking`:**
- `System time` offset should be **< 10 ms** (0.010 seconds)
- `Leap status` should be `Normal`

> **Warning:** If any VM shows a System time offset greater than 100 ms, investigate and resolve before proceeding with etcd or Patroni setup. Common causes: NTP server unreachable, firewall blocking UDP 123, or VM time sync interfering with chrony (disable VMware Tools time sync if using chrony).

---

## Step 6 — Verify Cross-Site Connectivity

### 6a. Ping Tests Between Key IPs

From a Site A VM, ping key Site B hosts:

```bash
# From any Site A VM
ping -c 10 {{ZBX_SERVER_B_IP}}
ping -c 10 {{PG_B_IP}}
ping -c 10 {{ETCD_2_IP}}
ping -c 10 {{FRONTEND_B_IP}}
ping -c 10 {{PROXY_B1_IP}}
```

From a Site B VM, ping key Site A hosts:

```bash
# From any Site B VM
ping -c 10 {{ZBX_SERVER_A_IP}}
ping -c 10 {{PG_A_IP}}
ping -c 10 {{ETCD_1_IP}}
ping -c 10 {{ETCD_3_IP}}
ping -c 10 {{FRONTEND_A_IP}}
ping -c 10 {{PROXY_A1_IP}}
```

**Expected result:** 0% packet loss on all tests.

### 6b. Measure Inter-Site RTT for Replication Mode Decision

This measurement determines whether to use synchronous or asynchronous database replication (Phase 4).

```bash
# Run from pg-siteA to pg-siteB — use 50 pings for a stable average
ping -c 50 {{PG_B_IP}}
```

Record the **average RTT** value and compare against the replication mode decision table:

| Inter-Site RTT | Sync Mode NVPS Ceiling | Recommendation |
|----------------|------------------------|----------------|
| < 2 ms (same campus) | 50K+ | **Synchronous** — negligible penalty |
| 2-5 ms (metro area) | 15-25K | **Synchronous** — acceptable for most deployments |
| 5-10 ms | 8-15K | **Evaluate** — sync if NVPS target is below ceiling |
| 10-20 ms | 4-8K | **Asynchronous recommended** |
| 20 ms+ | < 4K | **Asynchronous required** |

Record the measured RTT in your environment variables as `{{INTER_SITE_RTT_MS}}` and set `{{REPLICATION_MODE}}` to `synchronous` or `asynchronous` accordingly.

---

## Step 7 — Verify Firewall Prerequisites

Confirm that all required firewall rules are in place before proceeding. Refer to the **Appendix A — Firewall Port Matrix** for the complete list.

### Quick Connectivity Spot-Checks

Test critical paths from the relevant source VM:

```bash
# From Zabbix Server (Site A) to Database VIP (Site A)
nc -zv {{VIP_DB_A}} 5432

# From Zabbix Server (Site A) to Database VIP (Site B) — cross-site fallback
nc -zv {{VIP_DB_B}} 5432

# From etcd-1 to etcd-2 — peer communication
nc -zv {{ETCD_2_IP}} 2380

# From etcd-1 to etcd-3 — peer communication
nc -zv {{ETCD_3_IP}} 2380

# From pg-siteA to etcd-1 — Patroni client API
nc -zv {{ETCD_1_IP}} 2379

# From Frontend (Site A) to Zabbix Server (Site A)
nc -zv {{ZBX_SERVER_A_IP}} 10051

# From Proxy (Site A) to Zabbix Server (Site A)
nc -zv {{ZBX_SERVER_A_IP}} 10051

# From KEMP (Site A) to Frontend (Site A) — HTTP backend
nc -zv {{FRONTEND_A_IP}} 80

# From KEMP (Site A) to Database (Site A) — Patroni health check
nc -zv {{PG_A_IP}} 8008
```

If `nc` (netcat) is not available, install it:

```bash
dnf install -y nmap-ncat
```

**Expected result:** All connections succeed (`Connection to <IP> <PORT> port [tcp/*] succeeded!`). If any connection fails, escalate to the network/firewall team with the specific source IP, destination IP, and port before proceeding.

---

## Verification Checkpoint

Complete all checks before proceeding to Phase 2.

| # | Check | Command | Expected Result | Pass |
|---|-------|---------|-----------------|------|
| 1 | All Site A VMs provisioned and powered on | vSphere console | 10 VMs running | [ ] |
| 2 | All Site B VMs provisioned and powered on | vSphere console | 9 VMs running | [ ] |
| 3 | DNS forward lookup — `zabbix.{{DOMAIN}}` | `dig +short zabbix.{{DOMAIN}}` | `{{VIP_FRONTEND_A}}` | [ ] |
| 4 | DNS forward lookup — `zabbix-b.{{DOMAIN}}` | `dig +short zabbix-b.{{DOMAIN}}` | `{{VIP_FRONTEND_B}}` | [ ] |
| 5 | DNS forward lookup — `db-vip-a.{{DOMAIN}}` | `dig +short db-vip-a.{{DOMAIN}}` | `{{VIP_DB_A}}` | [ ] |
| 6 | DNS forward lookup — `db-vip-b.{{DOMAIN}}` | `dig +short db-vip-b.{{DOMAIN}}` | `{{VIP_DB_B}}` | [ ] |
| 7 | DNS forward lookup — all 23 records resolve | Step 4 script | All resolve to correct IPs | [ ] |
| 8 | NTP sync on all VMs — offset < 10 ms | `chronyc tracking` | System time < 0.010 s | [ ] |
| 9 | Cross-site ping — Site A to Site B | `ping -c 10 {{PG_B_IP}}` | 0% packet loss | [ ] |
| 10 | Cross-site ping — Site B to Site A | `ping -c 10 {{PG_A_IP}}` | 0% packet loss | [ ] |
| 11 | Inter-site RTT measured and recorded | `ping -c 50 {{PG_B_IP}}` | RTT recorded in `{{INTER_SITE_RTT_MS}}` | [ ] |
| 12 | Replication mode decision made | Compare RTT to ceiling table | `{{REPLICATION_MODE}}` set | [ ] |
| 13 | Firewall spot-checks pass | Step 7 nc commands | All connections succeed | [ ] |

---

## Troubleshooting

### DNS Propagation Delays

**Symptom:** `dig` returns correct results from the DNS server but VMs still resolve old or empty records.

**Resolution:**
1. Flush the local DNS cache on the VM:
   ```bash
   # If systemd-resolved is running
   resolvectl flush-caches

   # If nscd is running
   nscd -i hosts
   ```
2. Verify the VM is using the correct DNS servers:
   ```bash
   cat /etc/resolv.conf
   ```
3. Test resolution directly against the authoritative DNS server:
   ```bash
   dig @<dns-server-ip> zabbix.{{DOMAIN}}
   ```
4. If using Active Directory DNS, allow up to 15 minutes for AD replication between domain controllers.

### NTP Drift

**Symptom:** `chronyc tracking` shows System time offset > 100 ms or `Leap status: Not synchronised`.

**Resolution:**
1. Verify NTP servers are reachable:
   ```bash
   chronyc sources -v
   # Look for '*' (selected) or '+' (candidate) next to source
   # '?' means unreachable
   ```
2. Check that UDP port 123 is open to the NTP servers:
   ```bash
   nc -zuv {{NTP_SERVER_1}} 123
   ```
3. If VMware Tools time sync conflicts with chrony, disable it:
   ```bash
   vmware-toolbox-cmd timesync disable
   ```
4. Force an immediate sync:
   ```bash
   chronyc makestep
   ```
5. Wait 30 seconds and re-check:
   ```bash
   chronyc tracking
   ```

### Firewall Blocks

**Symptom:** `nc -zv` times out or returns "Connection refused" for a required port.

**Resolution:**
1. Verify that `firewalld` on the **destination** VM has the port open:
   ```bash
   # On the destination VM
   firewall-cmd --list-all
   ```
2. If the port is open locally but the connection still fails, the block is at a network firewall between sites. Provide the network team with:
   - Source IP and hostname
   - Destination IP and hostname
   - Port and protocol (TCP/UDP)
   - Direction (Site A to Site B, or vice versa)
3. For KEMP VLM VMs that are not yet configured, ICMP ping may work but TCP ports will not respond until KEMP is initialized (Phase 5). Mark KEMP connectivity checks as deferred.

### VM Provisioning Issues

**Symptom:** KEMP VLM fails to deploy from OVF.

**Resolution:**
1. Ensure the vSphere port group specified in `{{PORTGROUP_KEMP_A}}` / `{{PORTGROUP_KEMP_B}}` exists and is accessible to the target ESXi host.
2. KEMP VLMs require **thick provisioned** disks and **VMXNET3** network adapters. Verify these settings in the OVF deployment wizard.
3. Assign static MAC addresses to KEMP VLMs to ensure CARP (HA) functions correctly after vMotion events.
