# Appendix A — Firewall Port Matrix

> **Purpose:** Standalone reference for the firewall and network team. Open all ports listed below before beginning any implementation phase. All source/destination addresses use `{{VARIABLE}}` placeholders defined in [Phase 0 — Environment Variables](../00-environment-variables.md).

---

## How to Use

1. Work through each section with the firewall team
2. Replace `{{VARIABLE}}` placeholders with actual IPs/subnets from Phase 0
3. Apply rules in order — core ports first, then cross-site, then outbound
4. After applying, validate with the connection tests listed in each section
5. Check off each rule as confirmed open

---

## Core Zabbix Ports

These ports support agent data collection, proxy communication, and web frontend access.

| Port | Protocol | Source | Destination | Direction | Purpose |
|------|----------|--------|-------------|-----------|---------|
| 10050 | TCP | Proxy VMs (`{{PROXY_A1_IP}}`, `{{PROXY_A2_IP}}`, `{{PROXY_B1_IP}}`, `{{PROXY_B2_IP}}`) | Monitored agents (all hosts) | Outbound from proxy | Zabbix agent passive checks (proxy polls agent) |
| 10051 | TCP | Zabbix agents (all hosts) | Proxy VMs (`{{PROXY_A1_IP}}`, `{{PROXY_A2_IP}}`, `{{PROXY_B1_IP}}`, `{{PROXY_B2_IP}}`) | Inbound to proxy | Zabbix agent active checks (agent pushes to proxy) |
| 10051 | TCP | Proxy VMs (`{{PROXY_A1_IP}}`, `{{PROXY_A2_IP}}`, `{{PROXY_B1_IP}}`, `{{PROXY_B2_IP}}`) | Server VMs (`{{ZBX_SERVER_A_IP}}`, `{{ZBX_SERVER_B_IP}}`) | Inbound to server | Proxy data sync (proxy pushes collected data to server) |
| 443 | TCP | End users / management network | KEMP Frontend VIPs (`{{VIP_FRONTEND_A}}`, `{{VIP_FRONTEND_B}}`) | Inbound to KEMP VIP | Zabbix web frontend — HTTPS (SSL terminated at KEMP) |
| 80 | TCP | KEMP VLM internal (`{{KEMP_A1_IP}}`, `{{KEMP_A2_IP}}`, `{{KEMP_B1_IP}}`, `{{KEMP_B2_IP}}`) | Frontend VMs (`{{FRONTEND_A_IP}}`, `{{FRONTEND_B_IP}}`) | KEMP → Frontend | Backend HTTP traffic after SSL offloading |

**Validation test — agent connectivity:**

```bash
# From a proxy, test passive check connectivity to an agent
zabbix_get -s <AGENT_IP> -k agent.ping

# From an agent, test active check connectivity to a proxy
nc -zv {{PROXY_A1_IP}} 10051
```

---

## Database Cluster Ports

These ports support PostgreSQL client access, connection pooling, Patroni HA orchestration, and etcd consensus.

| Port | Protocol | Source | Destination | Direction | Purpose |
|------|----------|--------|-------------|-----------|---------|
| 5432 | TCP | Zabbix Server VMs (`{{ZBX_SERVER_A_IP}}`, `{{ZBX_SERVER_B_IP}}`), Frontend VMs (`{{FRONTEND_A_IP}}`, `{{FRONTEND_B_IP}}`) | Database VIPs (`{{VIP_DB_A}}`, `{{VIP_DB_B}}`) | Inbound to DB VIP | PostgreSQL client connections (via KEMP to Patroni primary) |
| 5432 | TCP | PostgreSQL VM (`{{PG_A_IP}}`) | PostgreSQL VM (`{{PG_B_IP}}`) | Bidirectional | PostgreSQL streaming replication (Patroni-managed) |
| 6432 | TCP | Zabbix Server VMs (`{{ZBX_SERVER_A_IP}}`, `{{ZBX_SERVER_B_IP}}`) | Database VMs (`{{PG_A_IP}}`, `{{PG_B_IP}}`) | Inbound to DB | PgBouncer connection pooling (if deployed) |
| 8008 | TCP | KEMP VLMs (`{{KEMP_A1_IP}}`, `{{KEMP_A2_IP}}`, `{{KEMP_B1_IP}}`, `{{KEMP_B2_IP}}`) | Database VMs (`{{PG_A_IP}}`, `{{PG_B_IP}}`) | Inbound to DB | Patroni REST API health checks (KEMP checks `/primary`) |
| 2379 | TCP | Patroni nodes (`{{PG_A_IP}}`, `{{PG_B_IP}}`), etcd peers | etcd nodes (`{{ETCD_1_IP}}`, `{{ETCD_2_IP}}`, `{{ETCD_3_IP}}`) | Inbound to etcd | etcd client API (Patroni reads/writes cluster state) |
| 2380 | TCP | etcd nodes (`{{ETCD_1_IP}}`, `{{ETCD_2_IP}}`, `{{ETCD_3_IP}}`) | etcd nodes (`{{ETCD_1_IP}}`, `{{ETCD_2_IP}}`, `{{ETCD_3_IP}}`) | Bidirectional (etcd mesh) | etcd peer communication (Raft consensus, leader election) |

**Validation test — database connectivity:**

```bash
# From Zabbix server, test PostgreSQL via VIP
psql -h {{VIP_DB_A}} -p 5432 -U zabbix -d zabbix -c "SELECT 1;"

# From KEMP network, test Patroni health endpoint
curl -s -o /dev/null -w "%{http_code}" http://{{PG_A_IP}}:8008/primary

# From Patroni node, test etcd client API
etcdctl --endpoints=https://{{ETCD_1_IP}}:2379 endpoint health
```

---

## KEMP Load Balancer Ports

These ports support KEMP management, HA synchronization, and VIP failover.

| Port | Protocol | Source | Destination | Direction | Purpose |
|------|----------|--------|-------------|-----------|---------|
| 8443 | TCP | Management network | KEMP VLMs (`{{KEMP_A1_IP}}`, `{{KEMP_A2_IP}}`, `{{KEMP_B1_IP}}`, `{{KEMP_B2_IP}}`) | Inbound to KEMP | KEMP Web User Interface (WUI) and REST API management |
| 6973 | TCP | KEMP VLM (`{{KEMP_A1_IP}}`) | KEMP VLM (`{{KEMP_A2_IP}}`) | Bidirectional within HA pair | KEMP HA configuration synchronization (Site A pair) |
| 6973 | TCP | KEMP VLM (`{{KEMP_B1_IP}}`) | KEMP VLM (`{{KEMP_B2_IP}}`) | Bidirectional within HA pair | KEMP HA configuration synchronization (Site B pair) |
| 224.0.0.18 | IP Protocol 112 (VRRP/CARP) | KEMP VLM (`{{KEMP_A1_IP}}`) | KEMP VLM (`{{KEMP_A2_IP}}`) | Multicast within HA pair | CARP heartbeat for VIP failover (Site A pair) |
| 224.0.0.18 | IP Protocol 112 (VRRP/CARP) | KEMP VLM (`{{KEMP_B1_IP}}`) | KEMP VLM (`{{KEMP_B2_IP}}`) | Multicast within HA pair | CARP heartbeat for VIP failover (Site B pair) |

> **Important:** CARP uses IP protocol 112 (not a TCP/UDP port). Ensure the firewall permits IP protocol 112 between KEMP HA peers, and that the VMware port group allows multicast to `224.0.0.18`. The port group security policy must have **MAC Address Changes = Accept** and **Forged Transmits = Accept** for CARP failover to function.

**Validation test — KEMP HA:**

```bash
# Test KEMP WUI access
curl -k -s -o /dev/null -w "%{http_code}" https://{{KEMP_A1_IP}}:8443

# Check HA sync status via REST API
curl -k -s "https://bal:{{KEMP_ADMIN_PASSWORD}}@{{KEMP_A1_IP}}:8443/access/hastatus" | xmllint --format -
```

---

## Cross-Site Requirements (Minimum)

All traffic flows required between Site A and Site B across the WAN link. These rules are mandatory for HA failover to function correctly.

| Source | Destination | Port | Protocol | Purpose |
|--------|-------------|------|----------|---------|
| `{{ZBX_SERVER_A_IP}}` | `{{VIP_DB_B}}` | 5432 | TCP | Server Node A → DB VIP at Site B (fallback path) |
| `{{ZBX_SERVER_B_IP}}` | `{{VIP_DB_A}}` | 5432 | TCP | Server Node B → DB VIP at Site A (fallback path) |
| `{{PG_A_IP}}` | `{{PG_B_IP}}` | 5432 | TCP | PostgreSQL streaming replication (primary → standby) |
| `{{PG_B_IP}}` | `{{PG_A_IP}}` | 5432 | TCP | PostgreSQL streaming replication (reverse after failover) |
| `{{ETCD_1_IP}}` | `{{ETCD_2_IP}}` | 2379, 2380 | TCP | etcd consensus — Site A node ↔ Site B node |
| `{{ETCD_2_IP}}` | `{{ETCD_1_IP}}` | 2379, 2380 | TCP | etcd consensus — Site B node ↔ Site A node |
| `{{ETCD_1_IP}}` | `{{ETCD_3_IP}}` | 2379, 2380 | TCP | etcd consensus — Site A node ↔ Arbitrator |
| `{{ETCD_3_IP}}` | `{{ETCD_1_IP}}` | 2379, 2380 | TCP | etcd consensus — Arbitrator ↔ Site A node |
| `{{ETCD_2_IP}}` | `{{ETCD_3_IP}}` | 2379, 2380 | TCP | etcd consensus — Site B node ↔ Arbitrator |
| `{{ETCD_3_IP}}` | `{{ETCD_2_IP}}` | 2379, 2380 | TCP | etcd consensus — Arbitrator ↔ Site B node |
| `{{PROXY_B1_IP}}`, `{{PROXY_B2_IP}}` | `{{ZBX_SERVER_A_IP}}` | 10051 | TCP | Site B proxies → Server at Site A (when A is active) |
| `{{PROXY_A1_IP}}`, `{{PROXY_A2_IP}}` | `{{ZBX_SERVER_B_IP}}` | 10051 | TCP | Site A proxies → Server at Site B (when B is active after failover) |
| KEMP at Site A (`{{KEMP_A1_IP}}`, `{{KEMP_A2_IP}}`) | `{{FRONTEND_B_IP}}` | 80 | TCP | Cross-site Real Server — KEMP A sends to Frontend B (backup) |
| KEMP at Site B (`{{KEMP_B1_IP}}`, `{{KEMP_B2_IP}}`) | `{{FRONTEND_A_IP}}` | 80 | TCP | Cross-site Real Server — KEMP B sends to Frontend A (backup) |
| KEMP at Site A (`{{KEMP_A1_IP}}`, `{{KEMP_A2_IP}}`) | `{{PG_B_IP}}` | 8008 | TCP | Patroni health check — KEMP A checks Site B database node |
| KEMP at Site B (`{{KEMP_B1_IP}}`, `{{KEMP_B2_IP}}`) | `{{PG_A_IP}}` | 8008 | TCP | Patroni health check — KEMP B checks Site A database node |

> **Note:** If the etcd arbitrator (`{{ETCD_3_IP}}`) is deployed at a third location, additional WAN rules are required between that location and both sites on ports 2379 and 2380.

---

## External Access (Outbound)

Outbound rules required for Microsoft 365 monitoring, Kubernetes integration, and optional services.

| Source | Destination | Port | Protocol | Purpose |
|--------|-------------|------|----------|---------|
| Zabbix Server / Proxy VMs | `graph.microsoft.com` | 443 | TCP (HTTPS) | Microsoft 365 monitoring — Graph API data collection |
| Zabbix Server / Proxy VMs | `login.microsoftonline.com` | 443 | TCP (HTTPS) | Microsoft 365 monitoring — OAuth2 token acquisition |
| Kubernetes in-cluster proxy | `{{ZBX_SERVER_A_IP}}`, `{{ZBX_SERVER_B_IP}}` | 10051 | TCP | Kubernetes monitoring data from in-cluster Zabbix proxy |
| All Zabbix VMs (optional) | NTP servers (`{{NTP_SERVER_1}}`, `{{NTP_SERVER_2}}`) | 123 | UDP | Time synchronization (critical for HA consistency) |
| All Zabbix VMs (optional) | DNS servers | 53 | TCP/UDP | DNS resolution |
| Zabbix Server / Frontend (optional) | SMTP relay | 25 / 587 | TCP | Email alert delivery |

> **Note:** If an HTTP proxy is required for outbound internet access, configure the `http_proxy` / `https_proxy` environment variables on the Zabbix server and proxy VMs. The M365 monitoring external scripts and webhook media types must also respect these proxy settings.

---

## SNMP Ports

Ports required for SNMP polling and trap reception on dedicated SNMP proxy VMs.

| Port | Protocol | Source | Destination | Direction | Purpose |
|------|----------|--------|-------------|-----------|---------|
| 161 | UDP | Proxy VMs (`{{PROXY_A1_IP}}`, `{{PROXY_A2_IP}}`, `{{PROXY_B1_IP}}`, `{{PROXY_B2_IP}}`) | Network devices (switches, routers, firewalls) | Outbound from proxy | SNMP polling (GET/WALK requests to monitored devices) |
| 162 | UDP | Network devices (switches, routers, firewalls) | SNMP Trap Proxy VMs (`{{PROXY_A_SNMPTRAP_IP}}`, `{{PROXY_B_SNMPTRAP_IP}}`) | Inbound to trap proxy | SNMP trap reception (devices send unsolicited alerts) |

> **Important:** SNMP trap proxy VMs are standalone instances — they are NOT members of a proxy group. This is because SNMP traps are push-based (devices send to a fixed IP), so the receiving proxy must have a stable, well-known address. Configure network devices to send traps to `{{PROXY_A_SNMPTRAP_IP}}` and/or `{{PROXY_B_SNMPTRAP_IP}}`.

**Validation test — SNMP connectivity:**

```bash
# From a proxy, test SNMP polling to a network device
snmpwalk -v3 -u {{SNMPV3_SECURITY_NAME}} -l authPriv \
  -a {{SNMPV3_AUTH_PROTOCOL}} -A '{{SNMPV3_AUTH_PASSPHRASE}}' \
  -x {{SNMPV3_PRIV_PROTOCOL}} -X '{{SNMPV3_PRIV_PASSPHRASE}}' \
  <DEVICE_IP> sysDescr.0

# Verify SNMP trap listener is running on trap proxy
ss -ulnp | grep 162
```

---

## Per-Role Summary

Quick reference showing which inbound ports must be open on each VM role. Use this as a checklist when configuring host-based firewalls (firewalld/iptables) on each VM.

### Database VMs (`{{PG_A_IP}}`, `{{PG_B_IP}}`)

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 5432 | TCP | KEMP VIPs, Zabbix Servers, peer PostgreSQL node | PostgreSQL client and replication connections |
| 6432 | TCP | Zabbix Servers (if PgBouncer deployed) | PgBouncer connection pooling |
| 8008 | TCP | KEMP VLMs | Patroni REST API health checks |
| 10050 | TCP | Zabbix Server / Proxy | Zabbix agent passive checks (self-monitoring) |

### etcd VMs (`{{ETCD_1_IP}}`, `{{ETCD_2_IP}}`, `{{ETCD_3_IP}}`)

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 2379 | TCP | Patroni nodes (`{{PG_A_IP}}`, `{{PG_B_IP}}`), peer etcd nodes | etcd client API |
| 2380 | TCP | Peer etcd nodes | etcd peer communication (Raft consensus) |
| 10050 | TCP | Zabbix Server / Proxy | Zabbix agent passive checks (self-monitoring) |

### Server VMs (`{{ZBX_SERVER_A_IP}}`, `{{ZBX_SERVER_B_IP}}`)

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 10051 | TCP | All proxies, Kubernetes in-cluster proxy, active-check agents | Zabbix trapper — proxy data sync and agent active checks |
| 10050 | TCP | Peer Zabbix Server / Proxy | Zabbix agent passive checks (self-monitoring) |

### Frontend VMs (`{{FRONTEND_A_IP}}`, `{{FRONTEND_B_IP}}`)

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 80 | TCP | KEMP VLMs (local and cross-site) | HTTP backend (KEMP forwards after SSL offload) |
| 443 | TCP | (Only if not using KEMP SSL offloading) | Direct HTTPS access — typically not used |
| 10050 | TCP | Zabbix Server / Proxy | Zabbix agent passive checks (self-monitoring) |

### Proxy VMs (`{{PROXY_A1_IP}}`, `{{PROXY_A2_IP}}`, `{{PROXY_B1_IP}}`, `{{PROXY_B2_IP}}`)

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 10051 | TCP | Active-check agents in the proxy's monitored network | Zabbix trapper — agent active check data |
| 10050 | TCP | Zabbix Server / peer proxy | Zabbix agent passive checks (self-monitoring) |
| 161 | UDP | (Outbound only — no inbound rule needed) | SNMP polling to network devices |

### SNMP Trap Proxy VMs (`{{PROXY_A_SNMPTRAP_IP}}`, `{{PROXY_B_SNMPTRAP_IP}}`)

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 162 | UDP | Network devices (switches, routers, firewalls) | SNMP trap reception |
| 10051 | TCP | Active-check agents (if any) | Zabbix trapper |
| 10050 | TCP | Zabbix Server / Proxy | Zabbix agent passive checks (self-monitoring) |

### KEMP VLMs (`{{KEMP_A1_IP}}`, `{{KEMP_A2_IP}}`, `{{KEMP_B1_IP}}`, `{{KEMP_B2_IP}}`)

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 8443 | TCP | Management network only | KEMP WUI and REST API |
| 6973 | TCP | HA partner KEMP VLM | HA configuration synchronization |
| 224.0.0.18 | IP Protocol 112 | HA partner KEMP VLM | CARP heartbeat for VIP failover |

---

## Firewall Rule Application Checklist

Use this checklist to track rule implementation progress.

| # | Rule Set | Ticket / CR | Applied By | Date | Verified |
|---|----------|-------------|------------|------|----------|
| 1 | Core Zabbix Ports | _______ | _______ | _______ | [ ] |
| 2 | Database Cluster Ports | _______ | _______ | _______ | [ ] |
| 3 | KEMP Load Balancer Ports | _______ | _______ | _______ | [ ] |
| 4 | Cross-Site Requirements | _______ | _______ | _______ | [ ] |
| 5 | External Access (Outbound) | _______ | _______ | _______ | [ ] |
| 6 | SNMP Ports | _______ | _______ | _______ | [ ] |
| 7 | Host-based firewalls (per-VM) | _______ | _______ | _______ | [ ] |
