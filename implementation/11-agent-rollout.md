# Phase 11 — Agent Rollout (Windows and Linux)

## Objective

Deploy Zabbix Agent 2 to all monitored hosts, configure auto-registration actions to automatically assign host groups and link templates based on host metadata, and implement a layered template linking strategy for scalable configuration management.

## Prerequisites

- [ ] Phase 8 complete — proxy groups online and accepting connections
- [ ] Phase 9 complete — all templates verified as present in Zabbix
- [ ] Zabbix Agent 2 MSI installer available for Windows deployment (download from `https://www.zabbix.com/download_agents`)
- [ ] Zabbix Agent 2 RPM/DEB packages available for Linux deployment (Zabbix repository configured)
- [ ] Deployment tool available: GPO/SCCM/Intune for Windows, Ansible for Linux (or equivalent)
- [ ] PSK generation process defined — each host receives a unique 256-bit (64 hex character) pre-shared key

---

## Step 1 — Configure Auto-Registration Actions

Auto-registration actions automatically process hosts when an unknown agent connects to a proxy or server for the first time. The agent's `HostMetadata` string determines which groups and templates are assigned.

Navigate to **Alerts > Actions > Autoregistration actions > Create action**.

### 1a. Windows Server — Base Registration

| Field | Value |
|-------|-------|
| Name | `Auto-Register: Windows Servers` |
| Conditions | Host metadata contains `Windows` |
| Enabled | Yes |

**Operations:**

| Operation | Details |
|-----------|---------|
| Add to host groups | `Windows Servers` |
| Link to templates | `Windows by Zabbix agent active` |
| Set host inventory mode | Automatic |

### 1b. IIS Web Server

| Field | Value |
|-------|-------|
| Name | `Auto-Register: IIS Web Servers` |
| Conditions | Host metadata contains `IIS` |

**Operations:**

| Operation | Details |
|-----------|---------|
| Add to host groups | `Web Servers` |
| Link to templates | `IIS by Zabbix agent` |

### 1c. SQL Server

| Field | Value |
|-------|-------|
| Name | `Auto-Register: SQL Server` |
| Conditions | Host metadata contains `SQLServer` |

**Operations:**

| Operation | Details |
|-----------|---------|
| Add to host groups | `Database Servers` |
| Link to templates | `MSSQL by ODBC` |

### 1d. Active Directory Domain Controller

| Field | Value |
|-------|-------|
| Name | `Auto-Register: Domain Controllers` |
| Conditions | Host metadata contains `DC` |

**Operations:**

| Operation | Details |
|-----------|---------|
| Add to host groups | `Domain Controllers` |
| Link to templates | `Active Directory DS by Zabbix agent` |

### 1e. Linux Server — Base Registration

| Field | Value |
|-------|-------|
| Name | `Auto-Register: Linux Servers` |
| Conditions | Host metadata contains `Linux` |

**Operations:**

| Operation | Details |
|-----------|---------|
| Add to host groups | `Linux Servers` |
| Link to templates | `Linux by Zabbix agent active` |
| Set host inventory mode | Automatic |

### 1f. Nginx

| Field | Value |
|-------|-------|
| Name | `Auto-Register: Nginx` |
| Conditions | Host metadata contains `Nginx` |

**Operations:**

| Operation | Details |
|-----------|---------|
| Add to host groups | `Web Servers` |
| Link to templates | `Nginx by Zabbix agent 2` |

### 1g. PostgreSQL

| Field | Value |
|-------|-------|
| Name | `Auto-Register: PostgreSQL` |
| Conditions | Host metadata contains `PostgreSQL` |

**Operations:**

| Operation | Details |
|-----------|---------|
| Add to host groups | `Database Servers` |
| Link to templates | `PostgreSQL by Zabbix agent 2` |

### 1h. Docker

| Field | Value |
|-------|-------|
| Name | `Auto-Register: Docker Hosts` |
| Conditions | Host metadata contains `Docker` |

**Operations:**

| Operation | Details |
|-----------|---------|
| Add to host groups | `Container Hosts` |
| Link to templates | `Docker by Zabbix agent 2` |

> **Important:** Auto-registration actions are evaluated independently. A host with `HostMetadata=Linux Nginx Docker production` will match actions 1e, 1f, and 1h, resulting in three host groups and three templates being applied. This is the intended behavior — roles stack.

> **IMPORTANT — PSK gap in auto-registration:** Auto-registration creates the host but does NOT configure PSK encryption on the server side. After auto-registration, an administrator must manually configure the PSK identity and value for each host via the Zabbix frontend or API. For automated PSK provisioning, use the Zabbix API to update the host's PSK settings as part of your deployment pipeline:
> ```bash
> # Example: Set PSK on auto-registered host via API
> curl -s -X POST "{{ZABBIX_FRONTEND_URL}}/api_jsonrpc.php" \
>   -H "Content-Type: application/json" \
>   -d '{"jsonrpc":"2.0","method":"host.update","params":{"hostid":"<HOST_ID>","tls_connect":2,"tls_accept":2,"tls_psk_identity":"PSK_<HOSTNAME>","tls_psk":"<64-char-hex>"},"id":1,"auth":"<API_TOKEN>"}'
> ```

### 1i. Create Required Host Groups

Before agents start connecting, create the following host groups at **Data collection > Host groups > Create host group**:

| Host Group | Description |
|------------|-------------|
| `Windows Servers` | All Windows hosts |
| `Linux Servers` | All Linux hosts |
| `Web Servers` | IIS and Nginx hosts |
| `Database Servers` | SQL Server, PostgreSQL hosts |
| `Domain Controllers` | Active Directory DCs |
| `Container Hosts` | Docker and container runtime hosts |

---

## Step 2 — Windows Agent Deployment

### 2a. Download the Installer

Download the Zabbix Agent 2 MSI installer for Windows from the official Zabbix download page:

```
https://www.zabbix.com/download_agents?version=7.0+LTS&release=7.0.0&os=Windows&os_version=Any&hardware=amd64&encryption=OpenSSL&packaging=MSI&show_legacy=0
```

Place the MSI file on a network share accessible to the deployment tool.

### 2b. Agent Configuration File

Create the configuration file `zabbix_agent2.conf` with the following content. Use your deployment tool to template the host-specific values.

```ini
# /etc/zabbix/zabbix_agent2.conf (or C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf)

# --- Server Connection ---
# Semicolons for active checks (agent initiates); commas for passive (server initiates)
ServerActive={{PROXY_A1_IP}};{{PROXY_A2_IP}}
Server={{PROXY_A1_IP}},{{PROXY_A2_IP}}

# --- Host Identity ---
Hostname=<HOSTNAME>
HostnameItem=system.hostname
HostMetadata=Windows <ROLES> <ENVIRONMENT>

# --- TLS / PSK Encryption ---
TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=PSK_<HOSTNAME>
TLSPSKFile=C:\Program Files\Zabbix Agent 2\zabbix_agent2.psk

# --- Logging ---
LogFile=C:\Program Files\Zabbix Agent 2\zabbix_agent2.log
LogFileSize=10
DebugLevel=3

# --- Security ---
DenyKey=system.run[*]

# --- Performance ---
BufferSend=5
BufferSize=100
```

> **Windows file permissions:** Restrict the PSK file ACL to SYSTEM and Administrators only:
> ```powershell
> icacls "C:\Program Files\Zabbix Agent 2\zabbix_agent2.psk" /inheritance:r /grant:r "SYSTEM:(R)" /grant:r "Administrators:(R)"
> ```

**HostMetadata construction:**

The `HostMetadata` string must contain keywords that match the auto-registration action conditions. Build it based on the server's roles:

| Server Role | HostMetadata Value |
|-------------|-------------------|
| Basic Windows server | `Windows production` |
| IIS web server | `Windows IIS production` |
| SQL Server database | `Windows SQLServer production` |
| Domain Controller | `Windows DC production` |
| IIS + SQL Server (multi-role) | `Windows IIS SQLServer production` |

Replace `production` with the appropriate environment: `production`, `staging`, `development`, `test`.

> **Note:** For Site B hosts, replace `{{PROXY_A1_IP}};{{PROXY_A2_IP}}` with `{{PROXY_B1_IP}};{{PROXY_B2_IP}}` in the `ServerActive` and `Server` directives.

### 2c. PSK Key Generation

Generate a unique PSK for each host. The PSK must be a 256-bit (64 hex character) random string:

```powershell
# PowerShell — generate a random PSK
$psk = -join ((0..63) | ForEach-Object { '{0:x}' -f (Get-Random -Maximum 16) })
$psk | Out-File -FilePath "C:\Program Files\Zabbix Agent 2\zabbix_agent2.psk" -Encoding ASCII -NoNewline
```

Or using OpenSSL (if available):

```bash
openssl rand -hex 32 > zabbix_agent2.psk
```

Store the PSK identity and value in your secrets manager for each host. You will need to enter these in the Zabbix frontend when the host auto-registers (or configure them via API).

### 2d. MSI Installation via Command Line

For silent deployment via GPO, SCCM, or Intune:

```powershell
# Note: {{FILE_SHARE}} is defined in the environment variables catalog (00-environment-variables.md)
msiexec /i "\\{{FILE_SHARE}}\zabbix_agent2-7.0.0-windows-amd64-openssl.msi" /qn ^
  SERVER="{{PROXY_A1_IP}},{{PROXY_A2_IP}}" ^
  SERVERACTIVE="{{PROXY_A1_IP}};{{PROXY_A2_IP}}" ^
  HOSTNAME="<HOSTNAME>" ^
  TLSCONNECT="psk" ^
  TLSACCEPT="psk" ^
  TLSPSKIDENTITY="PSK_<HOSTNAME>" ^
  TLSPSKFILE="C:\Program Files\Zabbix Agent 2\zabbix_agent2.psk" ^
  ENABLEPATH="C:\Program Files\Zabbix Agent 2" ^
  INSTALLFOLDER="C:\Program Files\Zabbix Agent 2"
```

After installation, deploy the PSK file and custom configuration, then restart the service:

```powershell
Restart-Service "Zabbix Agent 2"
```

### 2e. Windows Event Log Monitoring

The Windows by Zabbix agent active template includes basic Event Log monitoring. Add these additional items for more granular coverage:

**System Event Log — Warnings and Errors:**

| Field | Value |
|-------|-------|
| Name | `Windows Event Log: System (Warning+)` |
| Key | `eventlog[System,,"Warning\|Error\|Critical",,,,skip]` |
| Type | Zabbix agent (active) |
| Type of information | Log |
| Update interval | `1s` |

**Security Event Log — Audit Failures:**

| Field | Value |
|-------|-------|
| Name | `Windows Event Log: Security (Audit Failure)` |
| Key | `eventlog[Security,,"Audit Failure",,,,skip]` |
| Type | Zabbix agent (active) |
| Type of information | Log |
| Update interval | `1s` |

> **Note:** The `skip` parameter at the end tells the agent to skip old (already existing) log entries when monitoring starts, preventing a flood of historical events.

These items can be added directly to hosts or included in a custom overlay template (see Step 4).

### 2f. Windows Firewall Rule

If Windows Firewall is enabled, allow inbound connections on port 10050 for passive checks (if used):

```powershell
New-NetFirewallRule -DisplayName "Zabbix Agent 2" `
  -Direction Inbound -Protocol TCP -LocalPort 10050 `
  -Action Allow -Profile Domain
```

> **Note:** For active-only agent configurations (recommended), no inbound firewall rule is needed. The agent initiates all connections outbound to the proxy on port 10051.

---

## Step 3 — Linux Agent Deployment

### 3a. Install Zabbix Repository

On each Linux host (Rocky/RHEL 9 example):

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rocky/9/x86_64/zabbix-release-latest-7.0.el9.noarch.rpm
dnf clean all
dnf install -y zabbix-agent2 zabbix-agent2-plugin-*
```

For Debian/Ubuntu:

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu22.04_all.deb
dpkg -i zabbix-release_latest_7.0+ubuntu22.04_all.deb
apt update
apt install -y zabbix-agent2 zabbix-agent2-plugin-*
```

### 3b. Agent Configuration File

Create or edit `/etc/zabbix/zabbix_agent2.conf`:

```ini
# /etc/zabbix/zabbix_agent2.conf

# --- Server Connection ---
ServerActive={{PROXY_A1_IP}};{{PROXY_A2_IP}}
Server={{PROXY_A1_IP}},{{PROXY_A2_IP}}

# --- Host Identity ---
Hostname=<HOSTNAME>
HostnameItem=system.hostname
HostMetadata=Linux <ROLES> <ENVIRONMENT>

# --- TLS / PSK Encryption ---
TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=PSK_<HOSTNAME>
TLSPSKFile=/etc/zabbix/zabbix_agent2.psk

# --- Logging ---
LogFile=/var/log/zabbix/zabbix_agent2.log
LogFileSize=10
DebugLevel=3

# --- Security ---
DenyKey=system.run[*]

# --- Performance ---
BufferSend=5
BufferSize=100
```

**HostMetadata construction for Linux:**

| Server Role | HostMetadata Value |
|-------------|-------------------|
| Basic Linux server | `Linux production` |
| Nginx reverse proxy | `Linux Nginx production` |
| PostgreSQL database | `Linux PostgreSQL production` |
| Docker host | `Linux Docker production` |
| Nginx + Docker (multi-role) | `Linux Nginx Docker production` |

### 3c. PSK Key Generation

```bash
openssl rand -hex 32 > /etc/zabbix/zabbix_agent2.psk
chmod 640 /etc/zabbix/zabbix_agent2.psk
chown root:zabbix /etc/zabbix/zabbix_agent2.psk
```

### 3d. Ansible Deployment Role

For automated deployment across multiple hosts, use an Ansible role. Example playbook structure:

```yaml
# deploy-zabbix-agent.yml
---
- name: Deploy Zabbix Agent 2
  hosts: all
  become: true
  vars:
    zabbix_proxy_active: "{{PROXY_A1_IP}};{{PROXY_A2_IP}}"
    zabbix_proxy_passive: "{{PROXY_A1_IP}},{{PROXY_A2_IP}}"
    zabbix_environment: "production"
  tasks:
    - name: Install Zabbix repository
      ansible.builtin.dnf:
        name: "https://repo.zabbix.com/zabbix/7.0/rocky/9/x86_64/zabbix-release-latest-7.0.el9.noarch.rpm"
        state: present
        disable_gpg_check: true

    - name: Install Zabbix Agent 2
      ansible.builtin.dnf:
        name:
          - zabbix-agent2
          - zabbix-agent2-plugin-mongodb
          - zabbix-agent2-plugin-postgresql
        state: present

    - name: Generate PSK file
      ansible.builtin.shell:
        cmd: openssl rand -hex 32
      register: psk_value
      changed_when: false

    - name: Deploy PSK file
      ansible.builtin.copy:
        content: "{{ psk_value.stdout }}"
        dest: /etc/zabbix/zabbix_agent2.psk
        owner: root
        group: zabbix
        mode: "0640"

    - name: Deploy agent configuration
      ansible.builtin.template:
        src: zabbix_agent2.conf.j2
        dest: /etc/zabbix/zabbix_agent2.conf
        owner: root
        group: zabbix
        mode: "0644"
      notify: Restart Zabbix Agent 2

    - name: Enable and start Zabbix Agent 2
      ansible.builtin.systemd:
        name: zabbix-agent2
        enabled: true
        state: started

  handlers:
    - name: Restart Zabbix Agent 2
      ansible.builtin.systemd:
        name: zabbix-agent2
        state: restarted
```

> **Note:** Store PSK values generated during deployment in your secrets manager. You will need them if you ever need to re-register the host or verify encryption.

### 3e. Linux Log File Monitoring

Add these log monitoring items for Linux hosts. These can be placed in a custom overlay template or added directly.

**Syslog — Errors and Critical:**

| Field | Value |
|-------|-------|
| Name | `Linux Syslog: Errors and Critical` |
| Key | `log[/var/log/syslog,"error\|critical\|alert",,100,,skip]` |
| Type | Zabbix agent (active) |
| Type of information | Log |
| Update interval | `1s` |

For RHEL/Rocky systems using `/var/log/messages`:

| Field | Value |
|-------|-------|
| Key | `log[/var/log/messages,"error\|critical\|alert",,100,,skip]` |

> **Note:** The agent user (`zabbix`) must have read permission on the log files. For journal-based systems, use `systemd.unit.get` items instead.

### 3f. Firewall Configuration

If `firewalld` is active, allow inbound for passive checks (if used):

```bash
firewall-cmd --permanent --add-port=10050/tcp
firewall-cmd --reload
```

For active-only configurations, no inbound firewall rule is needed.

---

## Step 4 — Template Linking Strategy

Zabbix supports a layered approach to template management. This strategy provides consistent baseline monitoring with per-role and per-host customization.

### Layer 1 — Base OS Templates (auto-registration)

Applied automatically by auto-registration actions:

```
Linux by Zabbix agent active    → All Linux hosts
Windows by Zabbix agent active  → All Windows hosts
```

These provide: CPU, memory, disk, network, process, and service monitoring.

### Layer 2 — Role Templates (auto-registration)

Applied automatically based on `HostMetadata` keywords:

```
IIS by Zabbix agent             → Windows hosts with "IIS" in metadata
MSSQL by ODBC                   → Windows hosts with "SQLServer" in metadata
Active Directory DS              → Windows hosts with "DC" in metadata
Nginx by Zabbix agent 2         → Linux hosts with "Nginx" in metadata
PostgreSQL by Zabbix agent 2    → Linux hosts with "PostgreSQL" in metadata
Docker by Zabbix agent 2        → Linux hosts with "Docker" in metadata
```

### Layer 3 — Custom Overlay Templates (manual or API)

Create custom templates for organization-specific monitoring. Link these manually or via the Zabbix API after auto-registration:

| Template | Contains |
|----------|----------|
| `Custom - Windows Event Logs` | Event log items for System, Security, Application |
| `Custom - Linux Log Monitoring` | Syslog/journald monitoring items |
| `Custom - Certificate Expiry` | TLS certificate expiration checks |
| `Custom - Backup Verification` | Backup job status items and triggers |

### Layer 4 — Environment Threshold Overrides (host-level macros)

Use host-level macro overrides to adjust thresholds per host without modifying templates:

| Host | Macro | Value | Purpose |
|------|-------|-------|---------|
| `sql-prod-01` | `{$CPU.UTIL.CRIT}` | `95` | DB servers tolerate higher CPU |
| `sql-prod-01` | `{$VFS.FS.PUSED.MAX.CRIT}` | `95` | Larger disk threshold on DB |
| `web-prod-01` | `{$NET.IF.ERRORS.WARN}` | `10` | Stricter network error threshold |
| `dev-app-01` | `{$MEMORY.UTIL.MAX}` | `95` | Dev servers: relaxed memory threshold |

To set host-level macros:
1. Navigate to **Data collection > Hosts > [hostname] > Macros** tab
2. Add the macro and value
3. Host-level macros override template-level macros of the same name

---

## Step 5 — Agent Verification Per Host

After deploying the agent to a host, verify the end-to-end flow.

### 5a. Verify Agent Service

**Windows:**

```powershell
Get-Service "Zabbix Agent 2" | Select-Object Status, StartType
# Expected: Status=Running, StartType=Automatic
```

**Linux:**

```bash
systemctl status zabbix-agent2
# Expected: Active: active (running)
```

### 5b. Verify Agent Connectivity

Check the agent log for successful connection to the proxy:

**Windows:**

```powershell
Get-Content "C:\Program Files\Zabbix Agent 2\zabbix_agent2.log" -Tail 20
```

**Linux:**

```bash
tail -20 /var/log/zabbix/zabbix_agent2.log
```

Look for lines indicating:
- `active check configuration update from [{{PROXY_A1_IP}}:10051]` — successful active check registration
- No `TLS handshake` errors — PSK is correctly configured

### 5c. Verify Auto-Registration in Frontend

1. Navigate to **Data collection > Hosts**
2. Search for the newly deployed hostname
3. Verify:
   - Host appears in the correct host group(s)
   - Expected templates are linked
   - Host status is "Enabled"
   - Availability icon shows green (ZBX) for active agent

### 5d. Verify Data Collection

1. Navigate to **Monitoring > Latest data**
2. Filter by the host
3. Confirm items are collecting data:
   - `system.cpu.util` — CPU utilization
   - `vm.memory.utilization` — Memory usage
   - `vfs.fs.size[/,pused]` — Filesystem usage
   - `net.if.in` / `net.if.out` — Network traffic

### 5e. Verify PSK Encryption

1. Navigate to **Data collection > Hosts > [hostname] > Encryption** tab
2. Confirm:
   - Connections to host: PSK
   - Connections from host: PSK
   - PSK identity matches the agent configuration

---

## Verification Checkpoint

Complete all checks before marking this phase done.

| # | Check | How to Verify | Expected Result | Pass |
|---|-------|---------------|-----------------|------|
| 1 | Auto-registration actions created | Alerts > Actions > Autoregistration actions | 8 actions configured and enabled | [ ] |
| 2 | Host groups created | Data collection > Host groups | 6+ groups (Windows, Linux, Web, DB, DC, Container) | [ ] |
| 3 | Sample Windows host registered | Data collection > Hosts > search hostname | Host appears with Windows template linked | [ ] |
| 4 | Sample Windows host collecting data | Monitoring > Latest data > [host] | CPU, memory, disk items have values | [ ] |
| 5 | Windows Event Log items working | Monitoring > Latest data > [host] | eventlog items show recent entries | [ ] |
| 6 | Sample Linux host registered | Data collection > Hosts > search hostname | Host appears with Linux template linked | [ ] |
| 7 | Sample Linux host collecting data | Monitoring > Latest data > [host] | CPU, memory, disk items have values | [ ] |
| 8 | Multi-role host linked correctly | Deploy host with `Linux Nginx Docker` metadata | Both Nginx and Docker templates linked | [ ] |
| 9 | PSK encryption active | Data collection > Hosts > [host] > Encryption | PSK configured, no handshake errors in log | [ ] |
| 10 | Agent connects via proxy | Data collection > Hosts > [host] | "Monitored by proxy" shows correct proxy group | [ ] |

---

## Troubleshooting

### Agent Not Connecting to Proxy

**Symptom:** Agent log shows `active check configuration update from [proxy-ip:10051] is not available` or connection timeout errors.

**Resolution:**
1. Verify the `ServerActive` address in the agent configuration matches a proxy IP that is online:
   ```bash
   # From the agent host
   nc -zv {{PROXY_A1_IP}} 10051
   ```
2. Verify the proxy is running and accepting connections:
   ```bash
   # On the proxy
   systemctl status zabbix-proxy
   ss -tlnp | grep 10051
   ```
3. Check for firewall rules blocking outbound port 10051 from the agent host.
4. If using DNS names in `ServerActive`, verify they resolve correctly from the agent host.

### Auto-Registration Not Triggering

**Symptom:** Agent connects (log shows successful configuration update) but the host does not appear in Zabbix or appears without the expected templates.

**Resolution:**
1. Verify the `HostMetadata` value in the agent configuration contains the exact keyword the auto-registration action expects (case-sensitive).
2. Check **Alerts > Actions > Autoregistration actions** — ensure the action is **Enabled**.
3. Verify the action condition matches:
   - Condition type: Host metadata
   - Operator: contains
   - Value: exact keyword (e.g., `Windows`, not `windows`)
4. Check **Alerts > Actions > Action log** for the autoregistration action to see if it executed and any errors.
5. If the host already exists in Zabbix (from a previous registration attempt), auto-registration will not re-trigger. Delete the host and restart the agent to force re-registration.

### PSK Handshake Failure

**Symptom:** Agent log shows `TLS handshake returned error` or `PSK identity not found`.

**Resolution:**
1. Verify the PSK identity in the agent configuration (`TLSPSKIdentity`) matches exactly what is configured in the Zabbix frontend for that host (Data collection > Hosts > [host] > Encryption tab).
2. Verify the PSK file contains exactly 64 hexadecimal characters with no trailing newline or whitespace:
   ```bash
   wc -c /etc/zabbix/zabbix_agent2.psk
   # Expected: 64 characters
   hexdump -C /etc/zabbix/zabbix_agent2.psk | tail -2
   # Verify no 0a (newline) at the end
   ```
3. Verify file permissions — the zabbix user must be able to read the PSK file:
   ```bash
   ls -la /etc/zabbix/zabbix_agent2.psk
   # Expected: -rw-r----- root zabbix
   ```
4. If the host was auto-registered, PSK configuration may not have been set automatically. Navigate to the host's Encryption tab and configure PSK identity and value manually.

### Template Not Linked After Auto-Registration

**Symptom:** Host appears in Zabbix but is missing expected templates (e.g., IIS template not linked on a Windows IIS server).

**Resolution:**
1. Check the `HostMetadata` string on the agent — it must contain the keyword for each role:
   ```
   HostMetadata=Windows IIS production
   ```
   If `IIS` is missing, the auto-registration action for IIS will not fire.
2. Auto-registration actions process **only once** per host. If the host registered before the action was created (or with wrong metadata), you must:
   - Delete the host from Zabbix
   - Fix the `HostMetadata` in the agent configuration
   - Restart the agent to trigger re-registration
3. Alternatively, manually link the missing template at **Data collection > Hosts > [host] > Templates** tab.
