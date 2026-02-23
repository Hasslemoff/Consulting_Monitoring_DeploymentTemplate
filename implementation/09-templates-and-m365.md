# Phase 9 — Templates and Microsoft 365 Monitoring

## Objective

Import monitoring templates, configure VMware monitoring, set up self-monitoring infrastructure hosts, and establish Microsoft 365 monitoring via the Microsoft Graph API.

## Prerequisites

- [ ] Phase 8 complete — proxy groups online and connected to Zabbix Server
- [ ] vCenter read-only monitoring account created (`{{VCENTER_USER}}`)
- [ ] Azure AD / Entra admin access for app registration
- [ ] Microsoft 365 tenant ID, app registration details available
- [ ] Firewall rules allowing HTTPS (443) from proxies to vCenter endpoints and Microsoft Graph API (`graph.microsoft.com`)

> **Note:** Phases 09, 10, and 12 can run in parallel with different teams. Coordinate to avoid conflicting host group or template changes.

---

## Step 1 — Verify Standard Templates

Zabbix 7.0 LTS ships with these templates pre-installed. Navigate to **Data collection > Templates** and confirm each exists. Do not import duplicates.

| Template Name | Template Group | Used For |
|---------------|----------------|----------|
| Linux by Zabbix agent active | Templates/Operating systems | Linux hosts |
| Windows by Zabbix agent active | Templates/Operating systems | Windows hosts |
| IIS by Zabbix agent | Templates/Applications | IIS web servers |
| MSSQL by ODBC | Templates/Databases | SQL Server instances |
| Active Directory DS by Zabbix agent | Templates/Applications | Domain controllers |
| Nginx by Zabbix agent 2 | Templates/Applications | Nginx reverse proxies |
| PostgreSQL by Zabbix agent 2 | Templates/Databases | PostgreSQL databases |
| Docker by Zabbix agent 2 | Templates/Applications | Docker hosts |
| etcd by HTTP | Templates/Applications | etcd cluster nodes |
| Kubernetes cluster state by HTTP | Templates/Applications | Kubernetes clusters |
| Kubernetes nodes by HTTP | Templates/Applications | Kubernetes node monitoring |
| Kubernetes API server by HTTP | Templates/Applications | Kubernetes API server |
| Kubernetes Controller Manager by HTTP | Templates/Applications | Kubernetes controller |
| Kubernetes Scheduler by HTTP | Templates/Applications | Kubernetes scheduler |
| Kubernetes kubelet by HTTP | Templates/Applications | Kubernetes kubelet |
| VMware vCenter by HTTP | Templates/Cloud | vCenter infrastructure |
| Zabbix server health | Templates/Applications | Zabbix self-monitoring |
| Microsoft 365 reports by HTTP | Templates/Cloud | M365 usage and health |

If any template is missing, it may need to be imported from the Zabbix Git repository:

```
https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates
```

---

## Step 2 — Configure VMware Monitoring

VMware monitoring uses the vCenter SOAP/REST API via HTTP agent items. Zabbix discovers ESXi hosts, VMs, and datastores automatically through the VMware vCenter template.

### 2a. Create Host — vCenter-SiteA

1. Navigate to **Data collection > Hosts > Create host**
2. Configure the host:

| Field | Value |
|-------|-------|
| Host name | `vCenter-SiteA` |
| Visible name | `vCenter — Site A` |
| Host groups | `VMware/vCenter` (create if needed) |
| Interfaces | None required (HTTP agent items use macros) |
| Monitored by proxy group | `SiteA-Proxies` |

3. Link the template:
   - **Data collection > Hosts > vCenter-SiteA > Templates**
   - Link: **VMware vCenter by HTTP**

4. Set host-level macros at **Macros** tab:

| Macro | Value | Type |
|-------|-------|------|
| `{$VMWARE.URL}` | `{{VCENTER_URL_A}}` | Text |
| `{$VMWARE.USER}` | `{{VCENTER_USER}}` | Text |
| `{$VMWARE.PASSWORD}` | `{{VCENTER_PASSWORD}}` | Secret |

5. Click **Add** to save the host.

### 2b. Create Host — vCenter-SiteB

Repeat the steps above with these differences:

| Field | Value |
|-------|-------|
| Host name | `vCenter-SiteB` |
| Visible name | `vCenter — Site B` |
| Monitored by proxy group | `SiteB-Proxies` |
| `{$VMWARE.URL}` | `{{VCENTER_URL_B}}` |

All other macros (`{$VMWARE.USER}`, `{$VMWARE.PASSWORD}`) remain the same if both sites share a single monitoring account.

### 2c. VMware Auto-Discovery Behavior

Once the hosts are saved and a proxy picks up the configuration:

- **ESXi hosts** are discovered and created as separate Zabbix hosts automatically
- **Virtual machines** are discovered with their performance counters
- **Datastores** are discovered with capacity and latency metrics
- Discovery runs on the interval defined by `{$VMWARE.LLD.INTERVAL}` (default: 1 hour)

> **Tip:** To speed up initial discovery, temporarily set `{$VMWARE.LLD.INTERVAL}` to `300` (5 minutes), then revert after discovery completes.

---

## Step 3 — Configure Self-Monitoring Infrastructure Hosts

These hosts monitor the Zabbix infrastructure components themselves — the database, etcd cluster, load balancers, and Zabbix Server.

### 3a. PostgreSQL — Site A

1. Navigate to **Data collection > Hosts > Create host**
2. Configure:

| Field | Value |
|-------|-------|
| Host name | `pg-siteA` |
| Host groups | `Zabbix Infrastructure/Database` |
| Interfaces | Agent: `{{PG_A_IP}}` port `10050` |
| Monitored by proxy group | `SiteA-Proxies` |

3. Link template: **PostgreSQL by Zabbix agent 2**
4. Set macros:

| Macro | Value | Type |
|-------|-------|------|
| `{$PG.URI}` | `tcp://{{PG_A_IP}}:5432` | Text |
| `{$PG.USER}` | `zbx_monitor` | Text |
| `{$PG.PASSWORD}` | `{{DB_ZABBIX_PASSWORD}}` | Secret |
| `{$PG.DB}` | `zabbix` | Text |

### 3b. PostgreSQL — Site B

Repeat with Site B values:

| Field | Value |
|-------|-------|
| Host name | `pg-siteB` |
| Interfaces | Agent: `{{PG_B_IP}}` port `10050` |
| Monitored by proxy group | `SiteB-Proxies` |
| `{$PG.URI}` | `tcp://{{PG_B_IP}}:5432` |

### 3c. etcd Cluster Nodes

Create three hosts for the etcd cluster. Each uses the **etcd by HTTP** template.

**etcd-1:**

| Field | Value |
|-------|-------|
| Host name | `etcd-1` |
| Host groups | `Zabbix Infrastructure/etcd` |
| Interfaces | None required |
| Monitored by proxy group | `SiteA-Proxies` |

Macros:

| Macro | Value | Type |
|-------|-------|------|
| `{$ETCD.URL}` | `https://{{ETCD_1_IP}}:2379` | Text |

**etcd-2:**

| Field | Value |
|-------|-------|
| Host name | `etcd-2` |
| Monitored by proxy group | `SiteB-Proxies` |
| `{$ETCD.URL}` | `https://{{ETCD_2_IP}}:2379` |

**etcd-3:**

| Field | Value |
|-------|-------|
| Host name | `etcd-3` |
| Monitored by proxy group | `SiteA-Proxies` |
| `{$ETCD.URL}` | `https://{{ETCD_3_IP}}:2379` |

> **Note:** If the etcd template requires TLS client certificates for authentication, configure the `{$ETCD.HTTP.PROXY}`, `{$ETCD.CERT.PATH}`, and `{$ETCD.KEY.PATH}` macros on each host. Alternatively, if etcd is configured with `--client-cert-auth=false` for the monitoring endpoint, the default HTTPS connection will work.

### 3d. KEMP LoadMasters

Create hosts for monitoring KEMP VLM health and VIP status.

**KEMP-A (Site A pair):**

| Field | Value |
|-------|-------|
| Host name | `KEMP-A` |
| Host groups | `Zabbix Infrastructure/Load Balancers` |
| Interfaces | SNMP: `{{KEMP_A1_IP}}` port `161` (if SNMP enabled) |
| Monitored by proxy group | `SiteA-Proxies` |

For KEMP monitoring, use one or both of these approaches:

**Option A — SNMP monitoring:**
- Link the **KEMP LoadMaster SNMP** template (if available) or create custom SNMP items
- Configure SNMPv3 credentials at the host level

**Option B — HTTP agent items (REST API):**
- Create custom HTTP agent items that query the KEMP REST API
- Example item: `https://{{KEMP_A1_IP}}/access/getvs` to retrieve virtual service status
- Requires KEMP API credentials stored as secret macros

Repeat for **KEMP-B** using `{{KEMP_B1_IP}}` and `SiteB-Proxies`.

### 3e. Zabbix Server Self-Monitoring

1. Navigate to **Data collection > Hosts**
2. Locate the built-in **Zabbix server** host (created automatically during installation)
3. Verify the **Zabbix server health** template is linked
4. Confirm the host interface points to `127.0.0.1` port `10051`

This template monitors internal Zabbix Server metrics: queue size, cache utilization, preprocessing queue, HA cluster status, and more.

---

## Step 4 — Create Patroni Custom Monitoring Items

Patroni exposes a REST API on port 8008 that reports cluster state, node roles, and replication health. Create custom HTTP agent items to monitor this.

### 4a. Patroni Node Health Item — Site A

On host `pg-siteA`, create a new item:

| Field | Value |
|-------|-------|
| Name | `Patroni: Node health` |
| Type | HTTP agent |
| Key | `patroni.node.health` |
| URL | `http://{{PG_A_IP}}:8008/` |
| Request type | GET |
| Type of information | Text |
| Update interval | `30s` |
| History | `7d` |

Create a dependent item to extract the role:

| Field | Value |
|-------|-------|
| Name | `Patroni: Node role` |
| Type | Dependent item |
| Key | `patroni.node.role` |
| Master item | `patroni.node.health` |
| Type of information | Character |
| Preprocessing | JSONPath: `$.role` |

### 4b. Patroni Cluster Info Item — Site A

| Field | Value |
|-------|-------|
| Name | `Patroni: Cluster info` |
| Type | HTTP agent |
| Key | `patroni.cluster.info` |
| URL | `http://{{PG_A_IP}}:8008/cluster` |
| Request type | GET |
| Type of information | Text |
| Update interval | `60s` |
| History | `7d` |

### 4c. Repeat for Site B

Create identical items on host `pg-siteB`, replacing the URL with:
- Node health: `http://{{PG_B_IP}}:8008/`
- Cluster info: `http://{{PG_B_IP}}:8008/cluster`

### 4d. Create Patroni Triggers

Create these triggers on **both** PostgreSQL hosts:

**Trigger 1 — Role Change Detected:**

| Field | Value |
|-------|-------|
| Name | `Patroni: Role changed to {ITEM.LASTVALUE} on {HOST.NAME}` |
| Severity | Warning |
| Expression | `change(/pg-siteA/patroni.node.role)=1` |
| Recovery expression | (none — manual close or auto-resolve) |
| Description | Patroni node role has changed. Verify this was an expected failover. |

**Trigger 2 — Replication Lag Exceeds Threshold:**

Create a dependent item from `patroni.cluster.info` to extract replication lag, then create the trigger:

| Field | Value |
|-------|-------|
| Name | `Patroni: Replication lag > 30s on {HOST.NAME}` |
| Severity | High |
| Expression | `last(/pg-siteA/patroni.replication.lag)>30` |
| Description | Database replication lag exceeds 30 seconds. Investigate network or disk issues. |

**Trigger 3 — Patroni Node Unreachable:**

| Field | Value |
|-------|-------|
| Name | `Patroni: API unreachable on {HOST.NAME}` |
| Severity | Disaster |
| Expression | `nodata(/pg-siteA/patroni.node.health,120s)=1` |
| Description | Patroni REST API on port 8008 has not responded for 2 minutes. |

---

## Step 5 — Microsoft 365 Monitoring Setup

Microsoft 365 monitoring uses the Microsoft Graph API to collect usage reports, service health status, and license utilization data.

### 5a. Azure App Registration

Perform these steps in the **Microsoft Entra admin center** (`https://entra.microsoft.com`):

1. Navigate to **Applications > App registrations > New registration**
2. Configure the application:

| Field | Value |
|-------|-------|
| Name | `Zabbix M365 Monitoring` |
| Supported account types | Accounts in this organizational directory only (Single tenant) |
| Redirect URI | (leave blank) |

3. Click **Register**
4. Record the following values:

| Value | Variable |
|-------|----------|
| Application (client) ID | `{{M365_APP_ID}}` |
| Directory (tenant) ID | `{{M365_TENANT_ID}}` |

5. Navigate to **Certificates & secrets > Client secrets > New client secret**
   - Description: `Zabbix monitoring`
   - Expires: 24 months (set a calendar reminder for rotation)
   - Record the secret value as `{{M365_CLIENT_SECRET}}`

> **Warning:** The client secret value is only shown once at creation time. Copy it immediately and store it in your secrets manager.

### 5b. Grant API Permissions

1. Navigate to **API permissions > Add a permission > Microsoft Graph > Application permissions**
2. Add the following permissions:

| Permission | Purpose |
|------------|---------|
| `Reports.Read.All` | Read Microsoft 365 usage reports |
| `ServiceHealth.Read.All` | Read service health and incidents |
| `Organization.Read.All` | Read organization profile and licenses |
| `Application.Read.All` | Read application registration data |

3. Click **Grant admin consent for [Your Organization]**
4. Verify all permissions show a green checkmark under "Status"

> **Important:** Admin consent is required for Application permissions. Without it, the Graph API will return `403 Forbidden` errors.

### 5c. Import the M365 Template

1. Navigate to **Data collection > Templates**
2. Click **Import**
3. Import the **Microsoft 365 reports by HTTP** template
   - If not bundled with your Zabbix version, download from the Zabbix integrations repository:
     ```
     https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/cloud/microsoft_365
     ```

### 5d. Create Microsoft 365 Host

1. Navigate to **Data collection > Hosts > Create host**
2. Configure:

| Field | Value |
|-------|-------|
| Host name | `Microsoft 365` |
| Visible name | `Microsoft 365 — {{M365_TENANT_ID}}` |
| Host groups | `Cloud/Microsoft 365` (create if needed) |
| Interfaces | None required (HTTP agent items query Graph API directly) |
| Monitored by proxy group | `SiteA-Proxies` |

> **Note:** Assigning to a proxy group provides automatic failover. If the active proxy fails, another proxy in the group takes over M365 data collection seamlessly.

3. Link template: **Microsoft 365 reports by HTTP**

4. Set macros:

| Macro | Value | Type |
|-------|-------|------|
| `{$MS365.TENANT.ID}` | `{{M365_TENANT_ID}}` | Text |
| `{$MS365.APP.ID}` | `{{M365_APP_ID}}` | Text |
| `{$MS365.PASSWORD}` | `{{M365_CLIENT_SECRET}}` | Secret |

5. Click **Add** to save.

### 5e. Graph API Rate Limits

Microsoft Graph API enforces rate limits per application per tenant:

| Endpoint Category | Rate Limit | Recommended Interval |
|-------------------|------------|----------------------|
| Reports API | 5 requests per 10 seconds | `6h` (default in template) |
| Service Health API | 10 requests per 10 seconds | `5m` |
| Organization API | 10 requests per 10 seconds | `1h` |

The default template intervals are designed to stay well within these limits. Do **not** reduce polling intervals below the template defaults unless you have confirmed your tenant can handle the additional load.

> **Tip:** If you see `429 Too Many Requests` errors in the item status, increase the polling interval on the affected items.

---

## Verification Checkpoint

Complete all checks before proceeding to the next phase.

| # | Check | How to Verify | Expected Result | Pass |
|---|-------|---------------|-----------------|------|
| 1 | VMware discovery running — Site A | Data collection > Hosts > vCenter-SiteA > Latest data | ESXi hosts appearing in discovery items | [ ] |
| 2 | VMware discovery running — Site B | Data collection > Hosts > vCenter-SiteB > Latest data | ESXi hosts appearing in discovery items | [ ] |
| 3 | Auto-discovered VMs visible | Data collection > Hosts, filter by "VMware" group | VM hosts created with performance counters | [ ] |
| 4 | PostgreSQL Site A collecting data | Monitoring > Latest data > pg-siteA | Connection pool, query stats, replication data | [ ] |
| 5 | PostgreSQL Site B collecting data | Monitoring > Latest data > pg-siteB | Same as above | [ ] |
| 6 | etcd-1 health item green | Monitoring > Latest data > etcd-1 | `{$ETCD.URL}` responding, cluster healthy | [ ] |
| 7 | etcd-2 health item green | Monitoring > Latest data > etcd-2 | Same as above | [ ] |
| 8 | etcd-3 health item green | Monitoring > Latest data > etcd-3 | Same as above | [ ] |
| 9 | Patroni node health items returning | Monitoring > Latest data > pg-siteA | `patroni.node.health` shows JSON response | [ ] |
| 10 | Patroni role items populating | Monitoring > Latest data > pg-siteA | `patroni.node.role` shows `master` or `replica` | [ ] |
| 11 | Zabbix server health active | Monitoring > Latest data > Zabbix server | Queue size, cache stats, HA status items | [ ] |
| 12 | M365 template collecting data | Monitoring > Latest data > Microsoft 365 | Usage report items show values (may take 6h) | [ ] |
| 13 | M365 service health items active | Monitoring > Latest data > Microsoft 365 | Service health status items populated | [ ] |
| 14 | No unsupported items | Monitoring > Hosts > filter "Unsupported items" | Zero unsupported items on new hosts | [ ] |

---

## Troubleshooting

### VMware Discovery Returns Empty Results

**Symptom:** vCenter host shows "Supported" status for the template but no ESXi hosts or VMs are discovered after 1+ hours.

**Resolution:**
1. Verify the vCenter URL format — it must include `/sdk`:
   ```
   https://vcenter-a.example.com/sdk
   ```
2. Test the credentials manually from a proxy:
   ```bash
   curl -k "{{VCENTER_URL_A}}"
   ```
   You should receive an XML response (not a 401 or connection error).
3. Check the proxy logs for VMware-related errors:
   ```bash
   grep -i vmware /var/log/zabbix/zabbix_proxy.log | tail -20
   ```
4. Verify the monitoring account has at least **Read-only** role at the vCenter root level.
5. If using a self-signed certificate on vCenter, verify the proxy can reach it over HTTPS (port 443).

### Microsoft 365 Authentication Errors

**Symptom:** M365 items show "HTTP error 401" or "HTTP error 403" in the item status.

**Resolution:**
1. Verify the tenant ID is correct:
   - Navigate to Entra admin center > Overview
   - Confirm the tenant ID matches `{{M365_TENANT_ID}}`
2. Verify the application (client) ID matches `{{M365_APP_ID}}`
3. Confirm admin consent was granted:
   - Entra admin center > App registrations > Zabbix M365 Monitoring > API permissions
   - All permissions should show a green checkmark under "Status"
4. Verify the client secret has not expired:
   - Entra admin center > App registrations > Zabbix M365 Monitoring > Certificates & secrets
   - Check the "Expires" column
5. If the secret has expired, generate a new one and update `{$MS365.PASSWORD}` on the Zabbix host.
6. Test the token endpoint manually from the proxy:
   ```bash
   curl -X POST "https://login.microsoftonline.com/{{M365_TENANT_ID}}/oauth2/v2.0/token" \
     -d "client_id={{M365_APP_ID}}" \
     -d "client_secret={{M365_CLIENT_SECRET}}" \
     -d "scope=https://graph.microsoft.com/.default" \
     -d "grant_type=client_credentials"
   ```
   A successful response returns an `access_token`. An error response indicates which parameter is wrong.

### Patroni HTTP Items Timeout

**Symptom:** `patroni.node.health` item shows "Timeout" or "Connection refused."

**Resolution:**
1. Verify Patroni REST API is running on the target host:
   ```bash
   curl http://{{PG_A_IP}}:8008/
   ```
   Expected: JSON response with `role`, `state`, `timeline` fields.
2. If connection refused, check that Patroni is configured to listen on all interfaces:
   ```yaml
   # patroni.yml
   restapi:
     listen: 0.0.0.0:8008
   ```
3. Verify firewall allows port 8008 from the proxy to the PostgreSQL host:
   ```bash
   # From the proxy
   nc -zv {{PG_A_IP}} 8008
   ```
4. If using HTTPS for the Patroni REST API, update the item URL to use `https://` and configure any required certificate macros.

### etcd Items Show Unsupported

**Symptom:** etcd template items show "Unsupported" status with TLS-related errors.

**Resolution:**
1. If etcd requires client TLS authentication, the HTTP agent items need client certificates configured.
2. Verify etcd is accessible over HTTPS from the proxy:
   ```bash
   curl --cacert {{ETCD_CA_CERT_PATH}} https://{{ETCD_1_IP}}:2379/health
   ```
3. If etcd does not require client certs, verify the `--client-cert-auth` setting in the etcd configuration.
4. Check proxy logs for TLS handshake errors:
   ```bash
   grep -i etcd /var/log/zabbix/zabbix_proxy.log | tail -20
   ```
