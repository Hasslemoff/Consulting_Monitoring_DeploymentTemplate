# Phase 8 — Proxy Architecture

> **Objective:** Deploy Zabbix proxy groups at both sites for distributed monitoring, configure standalone SNMP trap proxies, and secure all proxy-to-server communication with PSK encryption.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Phase 6 complete | Zabbix Server HA cluster running (one active, one standby) |
| Frontend accessible | Zabbix frontend operational for proxy registration (Phase 7) |
| Proxy packages installed | `zabbix-proxy-sqlite3` (or `zabbix-proxy-pgsql` for heavy proxies) on all proxy VMs |
| Firewall rules | Port 10051/TCP open from proxies to both Zabbix server IPs (`{{ZBX_SERVER_A_IP}}`, `{{ZBX_SERVER_B_IP}}`) |
| SNMP packages | `snmptrapd`, `net-snmp-utils` installed on SNMP trap proxy VMs |
| IP addresses allocated | `{{PROXY_A1_IP}}`, `{{PROXY_A2_IP}}`, `{{PROXY_B1_IP}}`, `{{PROXY_B2_IP}}`, `{{PROXY_A_SNMPTRAP_IP}}`, `{{PROXY_B_SNMPTRAP_IP}}` |

---

## Step 1 — Create Proxy Groups in Zabbix Frontend

Proxy groups provide load balancing and failover for proxies within the same site. If one proxy in a group becomes unavailable, other proxies in the group take over its hosts.

1. Log in to the Zabbix frontend at `https://{{FQDN_FRONTEND}}`
2. Navigate to **Administration > Proxy groups**
3. Click **Create proxy group**

**Proxy Group: Site A**

| Parameter | Value |
|-----------|-------|
| Name | `SiteA-Proxies` |
| Failover period | `60s` |
| Minimum number of proxies | `2` |

4. Click **Add**
5. Create a second proxy group:

**Proxy Group: Site B**

| Parameter | Value |
|-----------|-------|
| Name | `SiteB-Proxies` |
| Failover period | `60s` |
| Minimum number of proxies | `2` |

6. Click **Add**

> **Note on failover period:** This is the time after which a proxy group redistributes hosts from an unavailable proxy to the remaining healthy proxies. 60 seconds is the recommended default. Lower values cause faster failover but may trigger unnecessary redistributions during brief network hiccups.

> **Note on minimum proxies:** Setting this to 2 means the proxy group will report a "degraded" status if fewer than 2 proxies are online. Adjust based on your redundancy requirements.

---

## Step 2 — Generate PSK Values

Generate a unique PSK (Pre-Shared Key) for each proxy. PSK encryption secures all communication between the proxy and the Zabbix server.

**On each proxy VM**, run:

```bash
openssl rand -hex 32 | sudo tee /etc/zabbix/proxy.psk > /dev/null
```

Set restrictive permissions:

```bash
sudo chmod 640 /etc/zabbix/proxy.psk
sudo chown root:zabbix /etc/zabbix/proxy.psk
```

Retrieve the generated PSK value (you will need this when registering the proxy in the frontend):

```bash
cat /etc/zabbix/proxy.psk
```

### PSK Inventory

Record each proxy's PSK value for frontend registration:

| Proxy | PSK Identity | PSK Value Variable |
|-------|-------------|-------------------|
| proxy-siteA-01 | `PSK_PROXY_A1` | `{{PSK_PROXY_A1}}` |
| proxy-siteA-02 | `PSK_PROXY_A2` | `{{PSK_PROXY_A2}}` |
| proxy-siteB-01 | `PSK_PROXY_B1` | `{{PSK_PROXY_B1}}` |
| proxy-siteB-02 | `PSK_PROXY_B2` | `{{PSK_PROXY_B2}}` |
| proxy-siteA-snmptrap | `PSK_PROXY_A_SNMPTRAP` | `{{PSK_PROXY_A_SNMPTRAP}}` |
| proxy-siteB-snmptrap | `PSK_PROXY_B_SNMPTRAP` | `{{PSK_PROXY_B_SNMPTRAP}}` |

> **Security:** Each proxy must have a unique PSK identity and value. Never reuse PSK values across proxies. Store PSK values in your secret manager alongside the other `(vault)` credentials.

---

## Step 3 — Configure Proxy-A1

Edit the Zabbix proxy configuration on `{{PROXY_A1_IP}}`.

**File:** `/etc/zabbix/zabbix_proxy.conf`

```bash
sudo cp /etc/zabbix/zabbix_proxy.conf /etc/zabbix/zabbix_proxy.conf.orig
sudo vi /etc/zabbix/zabbix_proxy.conf
```

```ini
### Proxy Identity ###
Hostname=proxy-siteA-01
ProxyMode=0

### Server Connection ###
# List ALL Zabbix Server HA nodes (semicolon-separated)
# The active proxy discovers which server node is active automatically
Server={{ZBX_SERVER_A_IP}};{{ZBX_SERVER_B_IP}}

### Local Database (SQLite — standard proxies) ###
DBName=/var/lib/zabbix/proxy.db

### Buffering ###
# ProxyLocalBuffer: hours to keep data locally after sending to server
ProxyLocalBuffer=1
# ProxyOfflineBuffer: hours to buffer data when server is unreachable
ProxyOfflineBuffer=24

### Synchronization ###
# ConfigFrequency: how often (seconds) the proxy requests configuration from server
ConfigFrequency=10
# DataSenderFrequency: how often (seconds) the proxy sends collected data to server
DataSenderFrequency=1

### Performance ###
CacheSize=128M
StartPollers=25
StartPollersUnreachable=5
StartTrappers=5
StartPingers=3
StartDiscoverers=3
Timeout=15

### Logging ###
LogFile=/var/log/zabbix/zabbix_proxy.log
LogFileSize=100
PidFile=/run/zabbix/zabbix_proxy.pid

### TLS / PSK Encryption ###
TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=PSK_PROXY_A1
TLSPSKFile=/etc/zabbix/proxy.psk
```

> **Note on Server directive:** List both Zabbix Server HA nodes separated by semicolons. The proxy will connect to whichever node is currently active. This ensures proxy connectivity survives a Zabbix Server failover.

> **Note on SQLite vs PostgreSQL:** SQLite is recommended for standard proxies monitoring up to 2,000 hosts. For heavy proxies (2,000+ hosts or high NVPS), use a local PostgreSQL database instead. Set `DBHost`, `DBPort`, `DBName`, `DBUser`, and `DBPassword` accordingly.

Ensure the SQLite database directory exists:

```bash
sudo mkdir -p /var/lib/zabbix
sudo chown zabbix:zabbix /var/lib/zabbix
```

---

## Step 4 — Configure Remaining Proxies

Configure each remaining proxy using the same template from Step 3, substituting the values below.

### Proxy-A2 (`{{PROXY_A2_IP}}`)

| Parameter | Value |
|-----------|-------|
| `Hostname` | `proxy-siteA-02` |
| `Server` | `{{ZBX_SERVER_A_IP}};{{ZBX_SERVER_B_IP}}` |
| `TLSPSKIdentity` | `PSK_PROXY_A2` |

### Proxy-B1 (`{{PROXY_B1_IP}}`)

| Parameter | Value |
|-----------|-------|
| `Hostname` | `proxy-siteB-01` |
| `Server` | `{{ZBX_SERVER_A_IP}};{{ZBX_SERVER_B_IP}}` |
| `TLSPSKIdentity` | `PSK_PROXY_B1` |

### Proxy-B2 (`{{PROXY_B2_IP}}`)

| Parameter | Value |
|-----------|-------|
| `Hostname` | `proxy-siteB-02` |
| `Server` | `{{ZBX_SERVER_A_IP}};{{ZBX_SERVER_B_IP}}` |
| `TLSPSKIdentity` | `PSK_PROXY_B2` |

All other parameters (buffering, performance, logging, TLS settings) remain identical to Proxy-A1. Only `Hostname`, and `TLSPSKIdentity` change.

On each proxy VM, ensure the PSK file and SQLite directory are configured:

```bash
sudo chmod 640 /etc/zabbix/proxy.psk
sudo chown root:zabbix /etc/zabbix/proxy.psk
sudo mkdir -p /var/lib/zabbix
sudo chown zabbix:zabbix /var/lib/zabbix
```

---

## Step 5 — Configure SNMP Trap Proxies

SNMP trap proxies are **standalone** proxies that handle inbound SNMP traps. They must NOT be added to proxy groups.

> **Critical:** SNMP trap reception is not supported within proxy groups. SNMP traps are sent to a specific IP address, and proxy group load balancing would break trap delivery. These proxies must remain standalone.

### 5a — Install and Configure snmptrapd

On `{{PROXY_A_SNMPTRAP_IP}}` and `{{PROXY_B_SNMPTRAP_IP}}`:

```bash
sudo dnf install -y net-snmp net-snmp-utils
```

Configure snmptrapd to write traps to the Zabbix trap file.

**File:** `/etc/snmp/snmptrapd.conf`

```bash
sudo vi /etc/snmp/snmptrapd.conf
```

```ini
# Accept SNMPv3 traps
createUser -e 0x0102030405 {{SNMPV3_SECURITY_NAME}} {{SNMPV3_AUTH_PROTOCOL}} "{{SNMPV3_AUTH_PASSPHRASE}}" {{SNMPV3_PRIV_PROTOCOL}} "{{SNMPV3_PRIV_PASSPHRASE}}"
authUser log {{SNMPV3_SECURITY_NAME}} priv

# Accept SNMPv2c traps (if needed for legacy devices)
# authCommunity log,execute,net public

# Output format compatible with Zabbix SNMP trapper
format2 %V\n% Agent Address: %A\n Agent Hostname: (%B)\n%v\n
outputOption fts

# Write traps to the Zabbix SNMP trap file
traphandle default /usr/sbin/zabbix_trap_receiver.pl
```

> **Note:** The engine ID must be unique per SNMP trap proxy. Generate a unique value for each: `snmpd -e` or use a hex-encoded string of your choosing. Do NOT use the example value `0x0102030405` in production.

Set restrictive permissions on the snmptrapd configuration:

```bash
sudo chmod 600 /etc/snmp/snmptrapd.conf
sudo chown root:root /etc/snmp/snmptrapd.conf
```

Create the Zabbix SNMP trap receiver script log directory:

```bash
sudo mkdir -p /var/log/snmptrap
sudo touch /var/log/snmptrap/snmptrap.log
sudo chown zabbix:zabbix /var/log/snmptrap/snmptrap.log
```

### 5b — Configure the SNMP Trap Proxy (Zabbix Proxy Configuration)

**File:** `/etc/zabbix/zabbix_proxy.conf` on `{{PROXY_A_SNMPTRAP_IP}}`

```ini
### Proxy Identity ###
Hostname=proxy-siteA-snmptrap
ProxyMode=0

### Server Connection ###
Server={{ZBX_SERVER_A_IP}};{{ZBX_SERVER_B_IP}}

### Local Database ###
DBName=/var/lib/zabbix/proxy.db

### Buffering ###
ProxyLocalBuffer=1
ProxyOfflineBuffer=24

### Synchronization ###
ConfigFrequency=10
DataSenderFrequency=1

### SNMP Trapper ###
SNMPTrapperFile=/var/log/snmptrap/snmptrap.log
StartSNMPTrapper=1

### Performance ###
CacheSize=64M
StartPollers=10
StartPollersUnreachable=3
StartTrappers=5
Timeout=15

### Logging ###
LogFile=/var/log/zabbix/zabbix_proxy.log
LogFileSize=100
PidFile=/run/zabbix/zabbix_proxy.pid

### TLS / PSK Encryption ###
TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=PSK_PROXY_A_SNMPTRAP
TLSPSKFile=/etc/zabbix/proxy.psk
```

### 5c — Configure Site B SNMP Trap Proxy

Repeat on `{{PROXY_B_SNMPTRAP_IP}}` with these substitutions:

| Parameter | Value |
|-----------|-------|
| `Hostname` | `proxy-siteB-snmptrap` |
| `TLSPSKIdentity` | `PSK_PROXY_B_SNMPTRAP` |

All other parameters remain identical.

### 5d — Set Up File Permissions

On both SNMP trap proxy VMs:

```bash
sudo chmod 640 /etc/zabbix/proxy.psk
sudo chown root:zabbix /etc/zabbix/proxy.psk
sudo mkdir -p /var/lib/zabbix
sudo chown zabbix:zabbix /var/lib/zabbix
```

> **Security:** While Zabbix Agent 2 denies `system.run[*]` by default, add `DenyKey=system.run[*]` explicitly in agent configurations for defense-in-depth.

---

## Step 6 — Register Proxies in Zabbix Frontend

Register each proxy in the Zabbix frontend so the server knows about them and can authenticate their PSK connections.

1. Log in to the Zabbix frontend at `https://{{FQDN_FRONTEND}}`
2. Navigate to **Administration > Proxies**
3. Click **Create proxy**

### Register Standard Proxies (Proxy Group Members)

Create each of the four standard proxies:

**Proxy: proxy-siteA-01**

| Parameter | Value |
|-----------|-------|
| Proxy name | `proxy-siteA-01` |
| Proxy mode | Active |
| Proxy group | `SiteA-Proxies` |
| Proxy address | `{{PROXY_A1_IP}}` |
| Encryption > Connections from proxy | PSK |
| PSK identity | `PSK_PROXY_A1` |
| PSK | `{{PSK_PROXY_A1}}` |

**Proxy: proxy-siteA-02**

| Parameter | Value |
|-----------|-------|
| Proxy name | `proxy-siteA-02` |
| Proxy mode | Active |
| Proxy group | `SiteA-Proxies` |
| Proxy address | `{{PROXY_A2_IP}}` |
| Encryption > Connections from proxy | PSK |
| PSK identity | `PSK_PROXY_A2` |
| PSK | `{{PSK_PROXY_A2}}` |

**Proxy: proxy-siteB-01**

| Parameter | Value |
|-----------|-------|
| Proxy name | `proxy-siteB-01` |
| Proxy mode | Active |
| Proxy group | `SiteB-Proxies` |
| Proxy address | `{{PROXY_B1_IP}}` |
| Encryption > Connections from proxy | PSK |
| PSK identity | `PSK_PROXY_B1` |
| PSK | `{{PSK_PROXY_B1}}` |

**Proxy: proxy-siteB-02**

| Parameter | Value |
|-----------|-------|
| Proxy name | `proxy-siteB-02` |
| Proxy mode | Active |
| Proxy group | `SiteB-Proxies` |
| Proxy address | `{{PROXY_B2_IP}}` |
| Encryption > Connections from proxy | PSK |
| PSK identity | `PSK_PROXY_B2` |
| PSK | `{{PSK_PROXY_B2}}` |

### Register SNMP Trap Proxies (Standalone — No Proxy Group)

**Proxy: proxy-siteA-snmptrap**

| Parameter | Value |
|-----------|-------|
| Proxy name | `proxy-siteA-snmptrap` |
| Proxy mode | Active |
| Proxy group | **(none — leave empty)** |
| Proxy address | `{{PROXY_A_SNMPTRAP_IP}}` |
| Encryption > Connections from proxy | PSK |
| PSK identity | `PSK_PROXY_A_SNMPTRAP` |
| PSK | `{{PSK_PROXY_A_SNMPTRAP}}` |

**Proxy: proxy-siteB-snmptrap**

| Parameter | Value |
|-----------|-------|
| Proxy name | `proxy-siteB-snmptrap` |
| Proxy mode | Active |
| Proxy group | **(none — leave empty)** |
| Proxy address | `{{PROXY_B_SNMPTRAP_IP}}` |
| Encryption > Connections from proxy | PSK |
| PSK identity | `PSK_PROXY_B_SNMPTRAP` |
| PSK | `{{PSK_PROXY_B_SNMPTRAP}}` |

> **Critical Reminder:** SNMP trap proxies must NOT be assigned to a proxy group. Network devices send traps to a specific IP address. If the proxy were part of a group, host redistribution during failover would break trap delivery because the device is still sending traps to the original proxy IP.

---

## Step 7 — Start All Proxy Services

Start the Zabbix proxy service on **all six proxy VMs**.

**Standard proxies:**

```bash
# On {{PROXY_A1_IP}}, {{PROXY_A2_IP}}, {{PROXY_B1_IP}}, {{PROXY_B2_IP}}
sudo systemctl enable --now zabbix-proxy
```

**SNMP trap proxies (start both snmptrapd and zabbix-proxy):**

```bash
# On {{PROXY_A_SNMPTRAP_IP}} and {{PROXY_B_SNMPTRAP_IP}}
sudo systemctl enable --now snmptrapd
sudo systemctl enable --now zabbix-proxy
```

Verify the services on each VM:

```bash
sudo systemctl status zabbix-proxy
```

Check the proxy log for successful startup and server connection:

```bash
sudo tail -50 /var/log/zabbix/zabbix_proxy.log
```

Look for these success indicators:

```
proxy #0 started [main process]
sending configuration data to server at "{{ZBX_SERVER_A_IP}}" ...
received configuration data from server at "{{ZBX_SERVER_A_IP}}", datalen XXXXX
```

---

## Step 8 — Verify Proxy Group Assignment

After all proxies are started and connected, verify the proxy group status in the frontend.

1. Navigate to **Administration > Proxy groups**
2. Verify:

| Proxy Group | Expected Status | Expected Proxy Count |
|-------------|----------------|---------------------|
| `SiteA-Proxies` | Online | 2 (proxy-siteA-01, proxy-siteA-02) |
| `SiteB-Proxies` | Online | 2 (proxy-siteB-01, proxy-siteB-02) |

3. Navigate to **Administration > Proxies**
4. Verify all six proxies show status **Online**:

| Proxy | Mode | Proxy Group | Expected Status |
|-------|------|-------------|----------------|
| proxy-siteA-01 | Active | SiteA-Proxies | Online |
| proxy-siteA-02 | Active | SiteA-Proxies | Online |
| proxy-siteB-01 | Active | SiteB-Proxies | Online |
| proxy-siteB-02 | Active | SiteB-Proxies | Online |
| proxy-siteA-snmptrap | Active | (none) | Online |
| proxy-siteB-snmptrap | Active | (none) | Online |

---

## Step 9 — Configure SNMPv3 Global Macros

Set SNMPv3 credentials as global macros so they can be referenced across all SNMP templates and host groups.

1. Navigate to **Administration > Macros**
2. Add the following global macros:

| Macro | Value | Type |
|-------|-------|------|
| `{$SNMPV3_SECURITYNAME}` | `{{SNMPV3_SECURITY_NAME}}` | Text |
| `{$SNMPV3_AUTHPROTOCOL}` | `{{SNMPV3_AUTH_PROTOCOL}}` | Text |
| `{$SNMPV3_AUTHPASSPHRASE}` | `{{SNMPV3_AUTH_PASSPHRASE}}` | Secret |
| `{$SNMPV3_PRIVPROTOCOL}` | `{{SNMPV3_PRIV_PROTOCOL}}` | Text |
| `{$SNMPV3_PRIVPASSPHRASE}` | `{{SNMPV3_PRIV_PASSPHRASE}}` | Secret |
| `{$SNMPV3_CONTEXTNAME}` | *(leave empty unless required by your devices)* | Text |
| `{$SNMPV3_SECURITYLEVEL}` | `authPriv` | Text |

3. Click **Update** to save

> **Note:** Use macro type **Secret** for passphrases. Secret macros are write-only in the frontend — once saved, their values cannot be viewed, only overwritten. This prevents accidental exposure through the UI.

---

## Verification Checkpoint

| # | Check | Expected Result | Command / Location |
|---|-------|-----------------|--------------------|
| 1 | All proxies online | 6 proxies show "Online" | **Administration > Proxies** |
| 2 | Proxy group SiteA-Proxies | Status: Online, 2 proxies | **Administration > Proxy groups** |
| 3 | Proxy group SiteB-Proxies | Status: Online, 2 proxies | **Administration > Proxy groups** |
| 4 | SNMP trap proxies standalone | Not assigned to any proxy group | **Administration > Proxies** |
| 5 | PSK connections verified | `TLS PSK` shown in proxy details | **Administration > Proxies > (proxy name)** |
| 6 | Proxy-to-server data flow | `lastaccess` updating in real-time | **Administration > Proxies** (check "Last seen" column) |
| 7 | SNMP trap proxy receiving traps | Trap log growing | `tail -f /var/log/snmptrap/snmptrap.log` on SNMP trap proxy |
| 8 | Server log — PSK connections | `connection from proxy "proxy-siteA-01" ... TLS PSK` | `tail /var/log/zabbix/zabbix_server.log` on active server |
| 9 | snmptrapd running | `active (running)` | `systemctl status snmptrapd` on SNMP trap proxies |
| 10 | SNMPv3 macros configured | All 7 macros present | **Administration > Macros** |
| 11 | All proxy services enabled on boot | `enabled` | `systemctl is-enabled zabbix-proxy` on all 6 proxy VMs |

---

## Troubleshooting

### Proxy Stays Offline (Not Connecting to Server)

**Symptoms:** Proxy shows "Offline" or "Unknown" status in Administration > Proxies. Last seen timestamp is stale or blank.

**Resolution:**

1. Check the proxy log for connection errors:
   ```bash
   sudo tail -100 /var/log/zabbix/zabbix_proxy.log | grep -i "error\|cannot\|refused\|timeout"
   ```

2. Verify the proxy can reach both Zabbix server IPs:
   ```bash
   nc -zv {{ZBX_SERVER_A_IP}} 10051
   nc -zv {{ZBX_SERVER_B_IP}} 10051
   ```

3. **PSK mismatch** — The most common cause. Verify the PSK identity and value match between the proxy config and the frontend registration:
   ```bash
   # On the proxy VM
   grep TLSPSKIdentity /etc/zabbix/zabbix_proxy.conf
   cat /etc/zabbix/proxy.psk
   ```
   Compare with the values entered in **Administration > Proxies > (proxy name) > Encryption tab**.

4. **Hostname mismatch** — The `Hostname` in `zabbix_proxy.conf` must exactly match the proxy name registered in the frontend:
   ```bash
   grep "^Hostname=" /etc/zabbix/zabbix_proxy.conf
   ```

5. **Wrong Server address** — Verify the `Server` directive lists the actual Zabbix server IPs, not VIPs or FQDNs:
   ```bash
   grep "^Server=" /etc/zabbix/zabbix_proxy.conf
   ```

6. Check the Zabbix server log for rejected connections:
   ```bash
   sudo tail -100 /var/log/zabbix/zabbix_server.log | grep -i "proxy\|psk\|tls"
   ```

### Proxy Group Shows Degraded Status

**Symptoms:** Proxy group status shows "Degraded" instead of "Online" in Administration > Proxy groups.

**Resolution:**

1. A proxy group enters "Degraded" state when fewer than the minimum number of proxies are online.
2. Check which proxy is offline:
   - Navigate to **Administration > Proxies**
   - Identify proxies assigned to the degraded group that show "Offline"
3. Follow the "Proxy Stays Offline" troubleshooting steps above for the affected proxy.
4. If a proxy was intentionally taken offline for maintenance, the degraded state is expected and will resolve when the proxy returns.
5. Hosts assigned to a degraded proxy group will be redistributed to the remaining healthy proxy(ies). Verify this at **Administration > Proxies > (proxy name) > Hosts**.

### SNMP Trap Proxy Not Receiving Traps

**Symptoms:** No data appearing in `/var/log/snmptrap/snmptrap.log`. SNMP trap items show no data.

**Resolution:**

1. Verify snmptrapd is running:
   ```bash
   sudo systemctl status snmptrapd
   ```

2. Check if the trap port (UDP 162) is listening:
   ```bash
   sudo ss -ulnp | grep 162
   ```

3. Verify the firewall allows inbound UDP 162:
   ```bash
   sudo firewall-cmd --list-ports
   # If not listed:
   sudo firewall-cmd --permanent --add-port=162/udp
   sudo firewall-cmd --reload
   ```

4. Send a test trap to verify the pipeline:
   ```bash
   # From a test machine
   snmptrap -v 3 -e 0x0102030405 -u {{SNMPV3_SECURITY_NAME}} -a {{SNMPV3_AUTH_PROTOCOL}} \
     -A "{{SNMPV3_AUTH_PASSPHRASE}}" -x {{SNMPV3_PRIV_PROTOCOL}} \
     -X "{{SNMPV3_PRIV_PASSPHRASE}}" -l authPriv \
     {{PROXY_A_SNMPTRAP_IP}} '' .1.3.6.1.4.1.8072.2.3.0.1 \
     .1.3.6.1.4.1.8072.2.3.2.1 s "Test trap"
   ```

5. Check the snmptrapd log for received traps:
   ```bash
   sudo journalctl -u snmptrapd --no-pager -n 50
   ```

6. Verify the `SNMPTrapperFile` path in `zabbix_proxy.conf` matches the actual trap log:
   ```bash
   grep SNMPTrapperFile /etc/zabbix/zabbix_proxy.conf
   ls -la /var/log/snmptrap/snmptrap.log
   ```

7. Ensure `StartSNMPTrapper=1` is set in `zabbix_proxy.conf`:
   ```bash
   grep StartSNMPTrapper /etc/zabbix/zabbix_proxy.conf
   ```

### PSK Authentication Errors in Server Log

**Symptoms:** Server log shows TLS errors like `TLS handshake returned error code 1` or `PSK identity not found`.

**Resolution:**

1. Confirm the PSK identity in the proxy config matches the frontend exactly (case-sensitive):
   ```bash
   grep TLSPSKIdentity /etc/zabbix/zabbix_proxy.conf
   ```

2. Confirm the PSK value (hex string) is exactly 64 characters:
   ```bash
   wc -c /etc/zabbix/proxy.psk
   # Expected: 65 (64 hex chars + newline)
   ```

3. Verify the PSK file contains only the hex string with no extra whitespace or line breaks:
   ```bash
   xxd /etc/zabbix/proxy.psk | tail -1
   ```

4. Regenerate the PSK if corrupted:
   ```bash
   openssl rand -hex 32 | sudo tee /etc/zabbix/proxy.psk > /dev/null
   sudo chmod 640 /etc/zabbix/proxy.psk
   sudo chown root:zabbix /etc/zabbix/proxy.psk
   ```
   Then update the PSK value in **Administration > Proxies > (proxy name) > Encryption tab** to match.

5. Restart the proxy:
   ```bash
   sudo systemctl restart zabbix-proxy
   ```

### Proxy Cannot Reach Server After HA Failover

**Symptoms:** After a Zabbix Server HA failover, proxy temporarily loses connectivity and shows offline.

**Resolution:**

1. This is typically transient. The proxy should reconnect within `ConfigFrequency` seconds (default: 10s) after the new active server starts accepting connections.
2. Verify both server IPs are listed in the proxy's `Server` directive:
   ```bash
   grep "^Server=" /etc/zabbix/zabbix_proxy.conf
   # Expected: Server={{ZBX_SERVER_A_IP}};{{ZBX_SERVER_B_IP}}
   ```
3. If only one server IP was listed, add both and restart the proxy:
   ```bash
   sudo systemctl restart zabbix-proxy
   ```
4. Check the proxy log to confirm it reconnected to the new active server:
   ```bash
   sudo tail -50 /var/log/zabbix/zabbix_proxy.log | grep "sending configuration"
   ```
