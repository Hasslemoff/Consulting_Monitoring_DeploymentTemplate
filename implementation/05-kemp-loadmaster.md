# Phase 5 — KEMP LoadMaster Deployment

> **Objective:** Deploy KEMP VLM HA pairs at both sites and configure three Virtual Services: Frontend HTTPS with SSL acceleration, HTTP-to-HTTPS redirect, and PostgreSQL load balancing with Patroni health checks.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Phase 4 complete | Patroni cluster healthy, primary at Site A, synchronous replica at Site B |
| KEMP OVF images | KEMP LoadMaster VLM OVA/OVF downloaded and accessible to vCenter |
| SSL certificates | `{{SSL_CERT_PATH}}`, `{{SSL_KEY_PATH}}`, and `{{SSL_CA_PATH}}` ready for import |
| VMware access | Administrative access to vCenter at both sites |
| Port group security | Ability to modify port group security policies on `{{PORTGROUP_KEMP_A}}` and `{{PORTGROUP_KEMP_B}}` |
| IP addresses allocated | `{{KEMP_A1_IP}}`, `{{KEMP_A2_IP}}`, `{{KEMP_B1_IP}}`, `{{KEMP_B2_IP}}`, `{{VIP_FRONTEND_A}}`, `{{VIP_FRONTEND_B}}`, `{{VIP_DB_A}}`, `{{VIP_DB_B}}` |

---

## Step 1 — Configure VMware Port Group Security Policies

KEMP HA uses CARP (Common Address Redundancy Protocol) for VIP failover. CARP requires the hypervisor to permit MAC address changes and forged transmits on the port group.

**At Site A — Port Group `{{PORTGROUP_KEMP_A}}`:**

1. In vCenter, navigate to **Networking > Distributed Switch** (or Standard Switch)
2. Select port group `{{PORTGROUP_KEMP_A}}`
3. Edit Settings > **Security**:

| Setting | Value |
|---------|-------|
| Promiscuous Mode | Reject (default) |
| MAC Address Changes | **Accept** |
| Forged Transmits | **Accept** |

**At Site B — Port Group `{{PORTGROUP_KEMP_B}}`:**

Repeat the identical security policy changes on `{{PORTGROUP_KEMP_B}}`.

> **Warning:** If MAC Address Changes or Forged Transmits are set to Reject, CARP failover will silently fail. The passive node will never be able to assume the VIP.

---

## Step 2 — Deploy KEMP OVF Appliances

Deploy **two** KEMP LoadMaster VLM appliances per site (four total).

### VM Configuration

| Parameter | Value |
|-----------|-------|
| NIC Type | VMXNET3 |
| MAC Address | Static (assign via vCenter to avoid conflicts) |
| Disk Provisioning | Thick Provision Eager Zeroed |
| Network | `{{PORTGROUP_KEMP_A}}` (Site A) or `{{PORTGROUP_KEMP_B}}` (Site B) |

### Appliance Inventory

| Appliance | Site | Management IP | Port Group |
|-----------|------|---------------|------------|
| KEMP-VLM-A1 | Site A | `{{KEMP_A1_IP}}` | `{{PORTGROUP_KEMP_A}}` |
| KEMP-VLM-A2 | Site A | `{{KEMP_A2_IP}}` | `{{PORTGROUP_KEMP_A}}` |
| KEMP-VLM-B1 | Site B | `{{KEMP_B1_IP}}` | `{{PORTGROUP_KEMP_B}}` |
| KEMP-VLM-B2 | Site B | `{{KEMP_B2_IP}}` | `{{PORTGROUP_KEMP_B}}` |

**Deploy via vCenter:**

1. Right-click the target cluster > **Deploy OVF Template**
2. Select the KEMP VLM OVA file
3. Name the VM (e.g., `KEMP-VLM-A1`)
4. Select the datastore and choose **Thick Provision Eager Zeroed**
5. Map the network to the correct port group
6. Complete deployment and power on the VM
7. Repeat for all four appliances

---

## Step 3 — Initial Access and Password Change

Access the KEMP Web User Interface (WUI) for each appliance:

```
https://{{KEMP_A1_IP}}:8443
```

Default credentials:

| Field | Value |
|-------|-------|
| Username | `bal` |
| Password | `1fourall` |

**Immediately change the password on every appliance:**

1. Log in with default credentials
2. Navigate to **System Configuration > System Administration > System User Management**
3. Change the `bal` user password to `{{KEMP_ADMIN_PASSWORD}}`
4. Log out and verify login with the new password

Repeat for `{{KEMP_A2_IP}}`, `{{KEMP_B1_IP}}`, and `{{KEMP_B2_IP}}`.

> **Security:** The default password `1fourall` is publicly documented. Changing it is a mandatory first step before any other configuration.

---

## Step 4 — Configure HA Pair at Site A

On the **first unit** (KEMP-VLM-A1 at `{{KEMP_A1_IP}}`):

1. Navigate to **System Configuration > High Availability**
2. Set **HA Mode** to **First HA**
3. Configure the shared Virtual IP addresses:

| Shared VIP | Purpose | Address |
|------------|---------|---------|
| Frontend VIP | Zabbix web frontend | `{{VIP_FRONTEND_A}}` |
| Database VIP | PostgreSQL / Patroni | `{{VIP_DB_A}}` |

4. Set the **Partner IP Address** to `{{KEMP_A2_IP}}`
5. Set the **HA Inter-Communication Interface** (keep default unless multiple interfaces exist)
6. Click **Set Configuration**

On the **second unit** (KEMP-VLM-A2 at `{{KEMP_A2_IP}}`):

1. Navigate to **System Configuration > High Availability**
2. Set **HA Mode** to **Second HA**
3. Set the **Partner IP Address** to `{{KEMP_A1_IP}}`
4. Click **Set Configuration**

After pairing, the configuration auto-synchronizes between the two units via **TCP port 6973**. All subsequent Virtual Service configuration is performed on the **active** unit only.

**Verify HA status:**

1. Navigate to **System Configuration > High Availability** on either unit
2. Confirm:
   - Unit 1 shows **Master** (active)
   - Unit 2 shows **Slave** (passive)
   - Last sync time is recent

---

## Step 5 — Configure HA Pair at Site B

Repeat the HA pairing process for Site B:

| Parameter | Value |
|-----------|-------|
| First HA unit | `{{KEMP_B1_IP}}` |
| Second HA unit | `{{KEMP_B2_IP}}` |
| Frontend VIP | `{{VIP_FRONTEND_B}}` |
| Database VIP | `{{VIP_DB_B}}` |

Follow the identical steps from Step 4, substituting Site B values.

---

## Step 6 — Import SSL Certificate

On each site's **active** KEMP unit:

1. Navigate to **Certificates & Security > SSL Certificates**
2. Click **Add New Certificate**
3. Choose **Import Certificate**
4. Upload the following files:

| File | Path |
|------|------|
| Certificate | `{{SSL_CERT_PATH}}` |
| Private Key | `{{SSL_KEY_PATH}}` |
| CA Chain (Intermediate) | `{{SSL_CA_PATH}}` |

5. Assign a friendly name (e.g., `zabbix-frontend`)
6. Click **Save**

> **Note:** The certificate must include the full chain (server cert + intermediates). If the CA chain is missing, browsers and KEMP health checks may report untrusted certificate errors.

Repeat on the Site B active unit. HA sync will propagate the certificate to the passive unit automatically.

---

## Step 7 — Virtual Service 1: Zabbix Frontend HTTPS

Create this Virtual Service on the **active** KEMP unit at each site.

### Site A Configuration

Navigate to **Virtual Services > Add New > Standard Virtual Service**.

**Basic Properties:**

| Parameter | Value |
|-----------|-------|
| Virtual Address | `{{VIP_FRONTEND_A}}` |
| Port | `443` |
| Protocol | TCP |
| Service Name | `Zabbix-Frontend-HTTPS` |

**SSL Properties:**

| Parameter | Value |
|-----------|-------|
| SSL Acceleration | Enabled |
| SSL Certificate | `zabbix-frontend` (imported in Step 6) |
| SSL Reencryption | Disabled (offload to HTTP on backend) |

**Scheduling and Persistence:**

| Parameter | Value |
|-----------|-------|
| Scheduling Method | Round Robin |
| Persistence Mode | Active Cookie (or Source IP if cookies not viable) |
| Persistence Timeout | 1800 seconds (30 minutes) |

**Real Servers:**

| Server Address | Port | Weight | Notes |
|----------------|------|--------|-------|
| `{{FRONTEND_A_IP}}` | 80 | 1000 | Primary backend |
| `{{FRONTEND_B_IP}}` | 80 | 0 | Backup only (standby weight) |

> **Note:** Setting the remote site frontend to weight 0 means it only receives traffic if the local frontend is down. Adjust weights based on your traffic distribution requirements.

**Health Check:**

| Parameter | Value |
|-----------|-------|
| Check Type | HTTP |
| Check Port | 80 |
| Check URL | `/` |
| Expected Response Code | 200 |
| Check Interval | 10 seconds |
| Check Timeout | 5 seconds |

Click **Save** to create the Virtual Service.

### Site B Configuration

Repeat the above on the Site B active KEMP unit with these substitutions:

| Parameter | Site B Value |
|-----------|-------------|
| Virtual Address | `{{VIP_FRONTEND_B}}` |
| Primary Real Server | `{{FRONTEND_B_IP}}:80` (weight 1000) |
| Backup Real Server | `{{FRONTEND_A_IP}}:80` (weight 0) |

---

## Step 8 — Virtual Service 2: HTTP-to-HTTPS Redirect

Create this Virtual Service to redirect all HTTP traffic to HTTPS.

### Site A Configuration

Navigate to **Virtual Services > Add New > Standard Virtual Service**.

**Basic Properties:**

| Parameter | Value |
|-----------|-------|
| Virtual Address | `{{VIP_FRONTEND_A}}` |
| Port | `80` |
| Protocol | TCP |
| Service Name | `Zabbix-HTTP-Redirect` |

**Redirect Configuration:**

1. Under **Standard Options**, locate the **Add HTTP Redirect** section
2. Set **Redirect URL** to:
   ```
   https://{{FQDN_FRONTEND}}
   ```
3. Set **Redirect Status Code** to `301` (Permanent Redirect)
4. Enable the redirect rule

> **Note:** Do NOT add real servers to this Virtual Service. All traffic is redirected before reaching a backend.

### Site B Configuration

Repeat on the Site B active KEMP unit:

| Parameter | Site B Value |
|-----------|-------------|
| Virtual Address | `{{VIP_FRONTEND_B}}` |
| Redirect URL | `https://{{FQDN_FRONTEND}}` |

---

## Step 9 — Virtual Service 3: PostgreSQL Primary (Patroni Health Check)

This is the critical database routing service. It uses the Patroni REST API to ensure traffic is **only** sent to the current PostgreSQL primary node.

### Site A Configuration

Navigate to **Virtual Services > Add New > Standard Virtual Service**.

**Basic Properties:**

| Parameter | Value |
|-----------|-------|
| Virtual Address | `{{VIP_DB_A}}` |
| Port | `5432` |
| Protocol | TCP |
| Service Name | `PostgreSQL-Primary` |
| Force L7 | **Disabled** (pure L4 mode) |

> **Critical:** Force L7 must be disabled. PostgreSQL uses a binary protocol that is incompatible with L7 inspection.

**Scheduling and Persistence:**

| Parameter | Value |
|-----------|-------|
| Scheduling Method | Fixed Weighted (Chained Failover) |
| Persistence Mode | Source IP |
| Persistence Timeout | 300 seconds |
| Idle Timeout | 3600 seconds |

> **Note:** Fixed Weighted with Chained Failover means the highest-weight healthy server receives all traffic. If it fails the health check, traffic moves to the next server. This is the correct behavior for a database primary.

**Real Servers:**

| Server Address | Port | Weight | Notes |
|----------------|------|--------|-------|
| `{{PG_A_IP}}` | 5432 | 1000 | PostgreSQL node at Site A |
| `{{PG_B_IP}}` | 5432 | 1000 | PostgreSQL node at Site B |

> **Important:** Both servers have equal weight because the **health check** determines which server receives traffic, not the weight. Only the Patroni primary responds HTTP 200 on `/primary`. The standby returns HTTP 503, so KEMP marks it as down and routes all traffic to the true primary.

**Health Check (Patroni REST API):**

| Parameter | Value |
|-----------|-------|
| Check Type | HTTP |
| Check Port | `8008` |
| Check URL | `/primary` |
| Expected Response Code | `200` |
| Check Interval | 5 seconds |
| Check Timeout | 3 seconds |

**How the health check works:**

- Patroni exposes a REST API on port 8008 on each PostgreSQL node
- `GET /primary` returns HTTP 200 **only** on the current primary node
- `GET /primary` returns HTTP 503 on standby/replica nodes
- KEMP marks the primary as "Up" and the standby as "Down"
- During a Patroni failover, the new primary starts responding 200 and the old primary stops — KEMP automatically routes to the new primary

> **Note:** If Patroni REST API authentication is enabled (Phase 04), configure the KEMP health check to include Basic Auth credentials. In the KEMP WUI: set the health check **Username** to `patroni_api` and **Password** to `{{PATRONI_API_PASSWORD}}`.

Click **Save** to create the Virtual Service.

### Site B Configuration

Repeat on the Site B active KEMP unit:

| Parameter | Site B Value |
|-----------|-------------|
| Virtual Address | `{{VIP_DB_B}}` |
| Real Servers | `{{PG_A_IP}}:5432` (weight 1000), `{{PG_B_IP}}:5432` (weight 1000) |

> **Note:** Both sites have identical real server configurations. Each site's KEMP independently discovers the current Patroni primary via the health check. This means both `{{VIP_DB_A}}` and `{{VIP_DB_B}}` always route to the same PostgreSQL primary node.

---

## Step 10 — Enable REST API for Automation

On each site's **active** KEMP unit:

1. Navigate to **System Configuration > Miscellaneous Options**
2. Locate **Remote Access** (API Access)
3. Set **Enable API Interface** to `Yes`
4. Restrict API access by source IP to the management network:
   - Add `{{ZBX_SERVER_A_IP}}` and `{{ZBX_SERVER_B_IP}}` to the allowed list
5. Click **Save**

> **Note:** The REST API is used for automated health monitoring and configuration validation. It listens on the same management port (8443).

---

## Step 11 — Verify Deployment via REST API

Run these verification commands from a management workstation or the Zabbix server.

**List all Virtual Services:**

```bash
curl -k -s "https://bal:{{KEMP_ADMIN_PASSWORD}}@{{KEMP_A1_IP}}:8443/access/listvs" | xmllint --format -
```

> **Security note:** The curl commands above embed credentials in the URL for clarity. In production, use environment variables to avoid shell history exposure:
> ```bash
> export KEMP_PASS='{{KEMP_ADMIN_PASSWORD}}'
> curl -k -s -u "bal:${KEMP_PASS}" "https://{{KEMP_A1_IP}}:8443/access/listvs"
> ```

**Check HA status:**

```bash
curl -k -s "https://bal:{{KEMP_ADMIN_PASSWORD}}@{{KEMP_A1_IP}}:8443/access/hastatus" | xmllint --format -
```

**Check specific Virtual Service status (PostgreSQL):**

```bash
curl -k -s "https://bal:{{KEMP_ADMIN_PASSWORD}}@{{KEMP_A1_IP}}:8443/access/showvs?vs={{VIP_DB_A}}&port=5432&prot=tcp" | xmllint --format -
```

**Verify Patroni health check is working (direct test):**

```bash
# Should return 200 on the primary node
curl -s -o /dev/null -w "%{http_code}" http://{{PG_A_IP}}:8008/primary

# Should return 503 on the standby node
curl -s -o /dev/null -w "%{http_code}" http://{{PG_B_IP}}:8008/primary
```

**Verify database connectivity through VIP:**

```bash
psql -h {{VIP_DB_A}} -p 5432 -U zabbix -d zabbix -c "SELECT pg_is_in_recovery();"
# Expected: f (false, meaning this is the primary)
```

Repeat all verification commands for Site B using `{{KEMP_B1_IP}}` and `{{VIP_DB_B}}`.

---

## Verification Checkpoint

| # | Check | Expected Result | Command / Location |
|---|-------|-----------------|--------------------|
| 1 | HA pair status — Site A | A1 = Master, A2 = Slave | WUI > System Configuration > High Availability |
| 2 | HA pair status — Site B | B1 = Master, B2 = Slave | WUI > System Configuration > High Availability |
| 3 | VS: Zabbix-Frontend-HTTPS | Status: Up, Real servers healthy | WUI > Virtual Services > View/Modify |
| 4 | VS: Zabbix-HTTP-Redirect | Status: Up (redirect active) | WUI > Virtual Services > View/Modify |
| 5 | VS: PostgreSQL-Primary | Status: Up, 1 server Up / 1 server Down | WUI > Virtual Services > View/Modify |
| 6 | SSL certificate valid | Not expired, correct FQDN | WUI > Certificates & Security > SSL Certificates |
| 7 | Patroni health check | Primary = 200, Standby = 503 | `curl http://{{PG_A_IP}}:8008/primary` |
| 8 | Frontend via VIP | HTTP 200 on `https://{{FQDN_FRONTEND}}` | `curl -k https://{{VIP_FRONTEND_A}}` |
| 9 | DB via VIP | Connected to primary (`pg_is_in_recovery = f`) | `psql -h {{VIP_DB_A}} -U zabbix -c "SELECT pg_is_in_recovery();"` |
| 10 | HTTP redirect | 301 redirect to HTTPS | `curl -I http://{{VIP_FRONTEND_A}}` |

---

## Troubleshooting

### CARP Failover Not Working (VIP Does Not Move to Passive Unit)

**Symptoms:** Active unit goes down, but passive unit does not assume the VIP. Clients lose connectivity.

**Root Cause:** VMware port group security policies are rejecting the MAC address change during failover.

**Resolution:**

1. Verify port group security settings:
   ```
   vCenter > Networking > {{PORTGROUP_KEMP_A}} > Edit > Security
   ```
2. Confirm **MAC Address Changes** = Accept and **Forged Transmits** = Accept
3. If using a Distributed Switch, check that the port group override is not blocked by the switch-level policy
4. Verify with a manual failover test:
   - On the active KEMP: **System Configuration > High Availability > Force Failover**
   - Monitor the VIP migration

### Patroni Health Check Failing (All Real Servers Marked Down)

**Symptoms:** PostgreSQL Virtual Service shows all backends as "Down" even though Patroni is running.

**Resolution:**

1. Verify Patroni REST API is accessible:
   ```bash
   curl -v http://{{PG_A_IP}}:8008/primary
   ```
2. Check that the KEMP health check uses port `8008` (not `5432`)
3. Ensure the health check URL is `/primary` (not `/health` or `/`)
4. Verify no firewall is blocking KEMP-to-Patroni traffic on port 8008:
   ```bash
   # From the KEMP appliance (if SSH is enabled)
   curl http://{{PG_A_IP}}:8008/primary
   ```
5. If Patroni is running but not responding on 8008, check `/etc/patroni/patroni.yml`:
   ```yaml
   restapi:
     listen: 0.0.0.0:8008
     connect_address: {{PG_A_IP}}:8008
   ```

### SSL Certificate Issues (Browser Warnings or Health Check Failures)

**Symptoms:** Browser shows certificate warning when accessing `https://{{FQDN_FRONTEND}}`. KEMP HTTPS health checks may also fail.

**Resolution:**

1. Verify the certificate chain is complete:
   ```bash
   openssl s_client -connect {{VIP_FRONTEND_A}}:443 -servername {{FQDN_FRONTEND}} </dev/null 2>/dev/null | openssl x509 -noout -dates -subject
   ```
2. Ensure the certificate Subject Alternative Name (SAN) includes `{{FQDN_FRONTEND}}`
3. In KEMP WUI, navigate to **Certificates & Security > SSL Certificates** and verify:
   - Certificate is not expired
   - CA chain is present (intermediate certificates included)
4. Re-import the certificate with the full chain if the intermediate is missing

### Virtual Service Shows "Down" Despite Healthy Backend

**Symptoms:** Real server is accessible directly but KEMP marks it as down.

**Resolution:**

1. Check the health check configuration matches the backend service:
   - Frontend: HTTP on port 80, URL `/`, expect 200
   - PostgreSQL: HTTP on port 8008, URL `/primary`, expect 200
2. Test the health check manually from the KEMP management network:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" http://{{FRONTEND_A_IP}}:80/
   ```
3. If using internal-only DNS, ensure the KEMP can resolve the real server addresses
4. Check the KEMP system logs: **System Configuration > Logging Options > View Logs**

### HA Configuration Not Syncing Between Units

**Symptoms:** Changes made on the active unit do not appear on the passive unit.

**Resolution:**

1. Verify TCP port 6973 is open between the two KEMP units
2. On the active unit, navigate to **System Configuration > High Availability**
3. Check the **Last Sync** timestamp
4. Force a manual sync: **System Configuration > High Availability > Force Sync**
5. If sync continues to fail, verify both units are running the same firmware version
