# Appendix C — Pre-Go-Live Security Checklist

> **Purpose:** Comprehensive security review checklist to complete before promoting the Zabbix monitoring platform to production. Every item must be verified and signed off by the responsible reviewer. Items marked **(Critical)** are blocking — the platform must not go live until these are resolved.

---

## How to Use

1. Assign each section to a reviewer (security engineer, infrastructure lead, or Zabbix administrator)
2. Work through each checklist item, verifying the current state of the deployment
3. Mark items as checked when verified
4. Document any exceptions or compensating controls in the Notes column
5. Complete the summary sign-off at the end before go-live approval

---

## Encryption

Verify that all data in transit between Zabbix components is encrypted.

| # | Check | Notes | Status |
|---|-------|-------|--------|
| E-1 | All agent-to-proxy and agent-to-server communication uses PSK or TLS certificates **(Critical)** | Configured in agent config: `TLSConnect` and `TLSAccept` parameters | [ ] Verified |
| E-2 | Each monitored host has a unique PSK identity and value (not shared across hosts) **(Critical)** | Shared PSKs allow impersonation of any host using the same key | [ ] Verified |
| E-3 | PSK values stored in Ansible Vault, HashiCorp Vault, or equivalent secret manager — never in plaintext files in source control | Reference: `{{PSK_PROXY_A1}}`, `{{PSK_PROXY_A2}}`, etc. from Phase 0 | [ ] Verified |
| E-4 | etcd peer communication uses mutual TLS (peer certificates) | Verify `ETCD_PEER_CERT_FILE` and `ETCD_PEER_KEY_FILE` in etcd config | [ ] Verified |
| E-5 | etcd client communication uses TLS (client certificates) | Verify `ETCD_CERT_FILE` and `ETCD_KEY_FILE`; Patroni connects via `https://` | [ ] Verified |
| E-6 | PostgreSQL replication traffic uses SSL (`sslmode=verify-ca` or `verify-full`) | Verify `ssl = on` in `postgresql.conf` and `sslmode` in Patroni replication config | [ ] Verified |
| E-7 | Zabbix frontend served over HTTPS via KEMP SSL offloading | Verify KEMP Virtual Service on port 443 with SSL Acceleration enabled | [ ] Verified |
| E-8 | HTTP-to-HTTPS redirect active on KEMP (port 80 → 443 with 301) | Verify: `curl -I http://{{VIP_FRONTEND_A}}` returns 301 to HTTPS | [ ] Verified |
| E-9 | SSL certificate chain is complete (server cert + intermediates + root) | Verify: `openssl s_client -connect {{VIP_FRONTEND_A}}:443` shows full chain | [ ] Verified |
| E-10 | TLS 1.2 is the minimum supported protocol version on KEMP | Disable SSLv3, TLS 1.0, TLS 1.1 in KEMP SSL Properties | [ ] Verified |
| E-11 | Auto-registered hosts have PSK configured post-registration (manual or API-automated) **(Critical)** | Auto-registration does NOT set PSK on the server side; verify all hosts have PSK identity and value configured | [ ] Verified |

---

## Credential Management

Verify that all passwords, secrets, and API keys are properly secured.

| # | Check | Notes | Status |
|---|-------|-------|--------|
| C-1 | Database passwords not stored in plaintext configuration files **(Critical)** | Use Ansible Vault references, environment variables, or Zabbix `Include` files with restricted permissions | [ ] Verified |
| C-2 | `{{DB_ZABBIX_PASSWORD}}` and `{{DB_POSTGRES_PASSWORD}}` are strong (20+ characters, mixed case, numbers, symbols) | Generate with: `openssl rand -base64 32` | [ ] Verified |
| C-3 | `{{DB_REPLICATOR_PASSWORD}}` is unique (not the same as database user passwords) | Used by Patroni for streaming replication | [ ] Verified |
| C-4 | Microsoft 365 client secret (`{{M365_CLIENT_SECRET}}`) stored as a Zabbix **secret macro** (type: Secret text, not visible in frontend) | Configure as `{$M365_CLIENT_SECRET}` with type "Secret text" in host/template macros | [ ] Verified |
| C-5 | KEMP admin password changed from default (`1fourall`) on all four appliances **(Critical)** | Default password is publicly documented; verify on `{{KEMP_A1_IP}}`, `{{KEMP_A2_IP}}`, `{{KEMP_B1_IP}}`, `{{KEMP_B2_IP}}` | [ ] Verified |
| C-6 | etcd authentication enabled with strong passwords for `root` and `patroni` users | Verify: `etcdctl auth status` returns `Authentication Status: true` | [ ] Verified |
| C-7 | SNMPv3 credentials use SHA-256 authentication and AES-256 privacy **(Critical)** | SNMPv1/v2c community strings transmit in cleartext; SNMPv3 with SHA-256+AES-256 is the minimum acceptable standard | [ ] Verified |
| C-8 | No SNMPv1 or SNMPv2c community strings in use (if possible) | If legacy devices require v2c, document the exception and restrict to isolated VLAN | [ ] Verified |
| C-9 | Zabbix frontend admin account password changed from default | Default: Admin / zabbix | [ ] Verified |
| C-10 | No credentials committed to the Git repository (`.gitignore` covers `*.vault`, `.env`, `*password*`) | Scan with: `git log --all -p -- '*.conf' '*.yml' | grep -i password` | [ ] Verified |

---

## Access Control

Verify that access to all components follows the principle of least privilege.

| # | Check | Notes | Status |
|---|-------|-------|--------|
| A-1 | Zabbix frontend session timeout configured (recommended: 30 minutes or less) | Administration > General > GUI: Auto-logout | [ ] Verified |
| A-2 | Zabbix frontend authentication uses LDAP, SAML, or Entra ID (not local accounts for day-to-day access) | Administration > Authentication; local `Admin` account retained for break-glass only | [ ] Verified |
| A-3 | Zabbix API access restricted to authorized service accounts and user groups | Verify API tokens are scoped; no wildcard API access for regular users | [ ] Verified |
| A-4 | Zabbix user groups follow least-privilege model (read-only groups for operators, read-write for administrators) | Administration > User groups; verify host group permissions | [ ] Verified |
| A-5 | KEMP WUI access (port 8443) restricted to management network only **(Critical)** | Firewall rules must block 8443 from end-user networks; see Appendix A | [ ] Verified |
| A-6 | KEMP REST API access restricted by source IP | System Configuration > Miscellaneous Options > Remote Access; whitelist `{{ZBX_SERVER_A_IP}}`, `{{ZBX_SERVER_B_IP}}` | [ ] Verified |
| A-7 | etcd client access restricted to Patroni nodes only (firewall) | Port 2379 inbound to etcd nodes should only accept from `{{PG_A_IP}}`, `{{PG_B_IP}}`, and peer etcd nodes | [ ] Verified |
| A-8 | PostgreSQL `pg_hba.conf` restricts connections to known Zabbix IPs only **(Critical)** | Verify no `0.0.0.0/0` or overly broad CIDR entries; only `{{ZBX_SERVER_A_IP}}`, `{{ZBX_SERVER_B_IP}}`, `{{FRONTEND_A_IP}}`, `{{FRONTEND_B_IP}}`, and replication IPs | [ ] Verified |
| A-9 | Zabbix Agent 2 `system.run[*]` is disabled (default behavior; verify DenyKey/AllowKey) **(Critical)** | `system.run[*]` allows arbitrary command execution on the monitored host; must be denied | [ ] Verified |
| A-10 | Zabbix Agent 2 `AllowKey` / `DenyKey` configured to restrict item keys to only those required | Review `/etc/zabbix/zabbix_agent2.conf` for permissive `AllowKey=*` entries | [ ] Verified |
| A-11 | SSH access to all Zabbix VMs restricted to authorized personnel (key-based authentication preferred) | Password-based SSH should be disabled; `PermitRootLogin no` in sshd_config | [ ] Verified |
| A-12 | Kubernetes Zabbix service account uses least-privilege RBAC (NOT `cluster-admin`) **(Critical)** | Verify ClusterRole bound to the service account grants only required API groups and resources | [ ] Verified |
| A-13 | Kubernetes namespace has NetworkPolicy restricting ingress/egress | Verify egress limited to Zabbix Server IPs on port 10051 and Kubernetes API | [ ] Verified |
| A-14 | Kubernetes Pod Security Standard applied (minimum `baseline`) | Verify: `kubectl get ns {{K8S_NAMESPACE}} -o jsonpath='{.metadata.labels}'` | [ ] Verified |
| A-15 | Patroni REST API authentication enabled (`restapi.authentication` in patroni.yml) **(Critical)** | Unauthenticated REST API allows unauthorized failovers and configuration changes | [ ] Verified |

---

## Network Security

Verify that network-level controls are properly configured.

| # | Check | Notes | Status |
|---|-------|-------|--------|
| N-1 | Firewall rules follow least-privilege — only required ports per role are open (see Appendix A) **(Critical)** | Cross-reference every open port against Appendix A per-role summary | [ ] Verified |
| N-2 | No unnecessary ports open on any Zabbix VM | Run `ss -tlnp` and `ss -ulnp` on each VM; compare against expected ports | [ ] Verified |
| N-3 | Cross-site traffic between Site A and Site B uses encrypted channels or a dedicated WAN link | If traffic crosses the public internet, a VPN or encrypted tunnel is mandatory | [ ] Verified |
| N-4 | SNMP trap proxy is standalone (not a member of any proxy group) | Standalone ensures a stable IP for trap reception; proxy group load-balancing would break SNMP trap routing | [ ] Verified |
| N-5 | KEMP management port (8443) is not exposed to end users or the internet | Verify with external port scan if applicable | [ ] Verified |
| N-6 | Database ports (5432, 6432) are not exposed beyond KEMP VIPs and Zabbix server IPs | Database should never be directly accessible from end-user networks | [ ] Verified |
| N-7 | etcd ports (2379, 2380) are not accessible from outside the Zabbix infrastructure VLAN | etcd compromise would allow complete control of the Patroni cluster | [ ] Verified |
| N-8 | Host-based firewall (firewalld) enabled on all RHEL/Rocky VMs with only required ports open | Verify: `firewall-cmd --list-all` on each VM | [ ] Verified |

---

## Operational Security

Verify that logging, backup, and maintenance processes are properly secured.

| # | Check | Notes | Status |
|---|-------|-------|--------|
| O-1 | Zabbix log files have appropriate permissions (`0640 zabbix:zabbix`) | Verify: `ls -la /var/log/zabbix/` on all server, proxy, and agent VMs | [ ] Verified |
| O-2 | Log rotation configured for all Zabbix components (server, proxy, agent, frontend) | Verify: `ls -la /etc/logrotate.d/zabbix*` or `LogFileSize` in Zabbix configs | [ ] Verified |
| O-3 | PostgreSQL log files have restricted permissions and rotation configured | Verify Patroni log configuration and `log_rotation_age` / `log_rotation_size` | [ ] Verified |
| O-4 | pgBackRest backup encryption enabled **(Critical)** | Verify `repo1-cipher-type=aes-256-cbc` in `/etc/pgbackrest/pgbackrest.conf` | [ ] Verified |
| O-5 | Backup integrity verified — at least one successful full backup and restore test completed | Document the restore test date and results | [ ] Verified |
| O-6 | SSL certificate expiry monitoring configured in Zabbix | Use the built-in "Website certificate by Zabbix agent 2" template or custom check against `{{FQDN_FRONTEND}}` | [ ] Verified |
| O-7 | etcd TLS certificate expiry monitoring configured | Create a Zabbix item to check certificate expiry on `{{ETCD_1_IP}}:2379` | [ ] Verified |
| O-8 | Microsoft 365 app registration client secret has a documented rotation schedule (12-24 months) | Record the secret expiry date and set a Zabbix reminder trigger 30 days before expiry | [ ] Verified |
| O-9 | SELinux set to **Enforcing** on all RHEL/Rocky VMs | Verify: `getenforce` returns `Enforcing` on each VM | [ ] Verified |
| O-10 | Zabbix SELinux policy module installed (`zabbix-selinux-policy` package) | Without the policy module, SELinux will block Zabbix server/agent operations | [ ] Verified |
| O-11 | Automated OS patching schedule documented (but does not auto-reboot Zabbix server or database VMs without coordination) | Uncoordinated reboots can trigger unintended HA failovers | [ ] Verified |
| O-12 | Backup encryption key stored separately from backup repository (not co-located) | If the encryption key is lost, all encrypted backups become unrecoverable | [ ] Verified |

---

## Audit Trail

Verify that an audit trail exists for administrative actions and authentication events.

| # | Check | Notes | Status |
|---|-------|-------|--------|
| AU-1 | Zabbix frontend audit log enabled | Administration > General > Audit log: **Enabled** | [ ] Verified |
| AU-2 | Audit log retention period set (recommended: 365 days minimum) | Administration > Housekeeping > Audit log: verify retention | [ ] Verified |
| AU-3 | Failed login attempts are logged and visible in audit log | Test with an intentional bad password; verify the event appears | [ ] Verified |
| AU-4 | LDAP / Entra ID integration configured for centralized authentication (if applicable) | Enables SSO and centralized account lifecycle management | [ ] Verified |
| AU-5 | KEMP system logs are forwarded to a central syslog server (if available) | System Configuration > Logging Options > Remote Syslog | [ ] Verified |
| AU-6 | PostgreSQL connection logging enabled (`log_connections = on`, `log_disconnections = on`) | Provides audit trail of database access | [ ] Verified |
| AU-7 | SSH access to all VMs logged to a central syslog server or SIEM | Enables detection of unauthorized access attempts | [ ] Verified |

---

## Summary Sign-Off

All sections must be marked **Complete** before the platform is approved for production go-live.

| Section | Reviewer | Date | Status |
|---------|----------|------|--------|
| Encryption | _______ | _______ | [ ] Complete |
| Credential Management | _______ | _______ | [ ] Complete |
| Access Control | _______ | _______ | [ ] Complete |
| Network Security | _______ | _______ | [ ] Complete |
| Operational Security | _______ | _______ | [ ] Complete |
| Audit Trail | _______ | _______ | [ ] Complete |

### Final Approval

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Security Reviewer | _______ | _______ | _______ |
| Infrastructure Lead | _______ | _______ | _______ |
| Zabbix Administrator | _______ | _______ | _______ |
| Project Manager | _______ | _______ | _______ |

> **Go-Live Gate:** This checklist must be completed and signed off before the Zabbix platform accepts production monitoring traffic. Any items marked as exceptions must have documented compensating controls and a remediation timeline.
