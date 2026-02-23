# Phase 0 — Environment Variables Catalog

> **Fill this file first.** Every `{{VARIABLE}}` used in phases 01-15 is defined here. Complete all values before starting any implementation phase. Variables marked `(vault)` must be stored in a secret manager (Ansible Vault, HashiCorp Vault, etc.) — never in plaintext.

---

## How to Use

1. Copy this file to `00-environment-variables.completed.md`
2. Replace every `<FILL>` with the actual value for your environment
3. Reference this completed file when executing each phase
4. Variables appear in `{{DOUBLE_BRACES}}` throughout all phase documents

---

## Network — Site A

| Variable | Description | Example | Used In |
|----------|-------------|---------|---------|
| `{{SITE_A_SUBNET}}` | Site A management subnet (CIDR) | `10.1.1.0/24` | 01, 02, 04, 17 |
| `{{SITE_A_GATEWAY}}` | Site A default gateway | `10.1.1.1` | 01, 02 |
| `{{SITE_A_VLAN_ID}}` | Site A VLAN for Zabbix infra | `100` | 01, 05 |
| `{{ZBX_SERVER_A_IP}}` | Zabbix Server Node 1 IP | `10.1.1.10` | 01, 06, 07, 08 |
| `{{FRONTEND_A_IP}}` | Zabbix Frontend Site A IP | `10.1.1.15` | 01, 05, 07 |
| `{{PG_A_IP}}` | PostgreSQL node Site A IP | `10.1.1.20` | 01, 03, 04, 05, 06 |
| `{{ETCD_1_IP}}` | etcd node 1 IP (Site A) | `10.1.1.30` | 01, 03, 04 |
| `{{ETCD_3_IP}}` | etcd node 3 / arbitrator IP | `10.1.1.31` | 01, 03, 04 |
| `{{PROXY_A1_IP}}` | Proxy Site A #1 IP | `10.1.1.40` | 01, 08 |
| `{{PROXY_A2_IP}}` | Proxy Site A #2 IP | `10.1.1.41` | 01, 08 |
| `{{PROXY_A_SNMPTRAP_IP}}` | SNMP trap proxy Site A IP | `10.1.1.42` | 01, 08 |
| `{{KEMP_A1_IP}}` | KEMP VLM-A1 management IP | `10.1.1.50` | 01, 05 |
| `{{KEMP_A2_IP}}` | KEMP VLM-A2 management IP | `10.1.1.51` | 01, 05 |
| `{{VIP_FRONTEND_A}}` | KEMP shared VIP — Frontend (Site A) | `10.1.1.100` | 01, 05, 07, 14, 15 |
| `{{VIP_DB_A}}` | KEMP shared VIP — Database (Site A) | `10.1.1.200` | 01, 05, 06, 07, 14, 15 |

## Network — Site B

| Variable | Description | Example | Used In |
|----------|-------------|---------|---------|
| `{{SITE_B_SUBNET}}` | Site B management subnet (CIDR) | `10.2.1.0/24` | 01, 02, 04, 17 |
| `{{SITE_B_GATEWAY}}` | Site B default gateway | `10.2.1.1` | 01, 02 |
| `{{SITE_B_VLAN_ID}}` | Site B VLAN for Zabbix infra | `200` | 01, 05 |
| `{{ZBX_SERVER_B_IP}}` | Zabbix Server Node 2 IP | `10.2.1.10` | 01, 06, 07, 08 |
| `{{FRONTEND_B_IP}}` | Zabbix Frontend Site B IP | `10.2.1.15` | 01, 05, 07 |
| `{{PG_B_IP}}` | PostgreSQL node Site B IP | `10.2.1.20` | 01, 03, 04, 05, 06 |
| `{{ETCD_2_IP}}` | etcd node 2 IP (Site B) | `10.2.1.30` | 01, 03, 04 |
| `{{PROXY_B1_IP}}` | Proxy Site B #1 IP | `10.2.1.40` | 01, 08 |
| `{{PROXY_B2_IP}}` | Proxy Site B #2 IP | `10.2.1.41` | 01, 08 |
| `{{PROXY_B_SNMPTRAP_IP}}` | SNMP trap proxy Site B IP | `10.2.1.42` | 01, 08 |
| `{{KEMP_B1_IP}}` | KEMP VLM-B1 management IP | `10.2.1.50` | 01, 05 |
| `{{KEMP_B2_IP}}` | KEMP VLM-B2 management IP | `10.2.1.51` | 01, 05 |
| `{{VIP_FRONTEND_B}}` | KEMP shared VIP — Frontend (Site B) | `10.2.1.100` | 01, 05, 07, 14, 15 |
| `{{VIP_DB_B}}` | KEMP shared VIP — Database (Site B) | `10.2.1.200` | 01, 05, 06, 07, 14, 15 |

## DNS

| Variable | Description | Example | Used In |
|----------|-------------|---------|---------|
| `{{DOMAIN}}` | Primary DNS domain | `example.com` | 01, 07, 10, 11, 15 |
| `{{FQDN_FRONTEND}}` | Primary frontend URL | `zabbix.example.com` | 01, 05, 07, 15 |
| `{{FQDN_FRONTEND_B}}` | Secondary/DR frontend URL | `zabbix-b.example.com` | 01 |
| `{{FQDN_DB_VIP_A}}` | Database VIP DNS (Site A) | `db-vip-a.example.com` | 01, 04 |
| `{{FQDN_DB_VIP_B}}` | Database VIP DNS (Site B) | `db-vip-b.example.com` | 01, 04 |
| `{{NTP_SERVER_1}}` | Primary NTP server | `ntp1.example.com` | 01, 02 |
| `{{NTP_SERVER_2}}` | Secondary NTP server | `ntp2.example.com` | 01, 02 |
| `{{DNS_SERVER_1}}` | Primary DNS server | `10.1.1.2` | 01, 02 |
| `{{DNS_SERVER_2}}` | Secondary DNS server | `10.1.1.3` | 01, 02 |
| `{{DNS_TTL}}` | DNS record TTL (seconds) | `60` | 01 |

## Credentials (vault)

| Variable | Description | Example | Used In |
|----------|-------------|---------|---------|
| `{{DB_ZABBIX_PASSWORD}}` | PostgreSQL `zabbix` user password `(vault)` | `<generated>` | 02, 04, 06, 07 |
| `{{DB_POSTGRES_PASSWORD}}` | PostgreSQL `postgres` superuser password `(vault)` | `<generated>` | 04 |
| `{{DB_REPLICATOR_PASSWORD}}` | PostgreSQL replication user password `(vault)` | `<generated>` | 04 |
| `{{ETCD_ROOT_PASSWORD}}` | etcd `root` user password `(vault)` | `<generated>` | 03 |
| `{{ETCD_PATRONI_PASSWORD}}` | etcd `patroni` user password `(vault)` | `<generated>` | 03, 04 |
| `{{KEMP_ADMIN_PASSWORD}}` | KEMP LoadMaster admin password `(vault)` | `<generated>` | 05, 14 |
| `{{PGBOUNCER_MD5_HASH}}` | PgBouncer auth hash for zabbix user `(vault)` | `md5...` | 04 |
| `{{PSK_PROXY_A1}}` | PSK value for Proxy-A1 `(vault)` | `<64-char hex>` | 08, 16 |
| `{{PSK_PROXY_A2}}` | PSK value for Proxy-A2 `(vault)` | `<64-char hex>` | 08, 16 |
| `{{PSK_PROXY_B1}}` | PSK value for Proxy-B1 `(vault)` | `<64-char hex>` | 08, 16 |
| `{{PSK_PROXY_B2}}` | PSK value for Proxy-B2 `(vault)` | `<64-char hex>` | 08, 16 |
| `{{PSK_PROXY_A_SNMPTRAP}}` | PSK value for SNMP trap proxy A `(vault)` | `<64-char hex>` | 08, 16 |
| `{{PSK_PROXY_B_SNMPTRAP}}` | PSK value for SNMP trap proxy B `(vault)` | `<64-char hex>` | 08, 16 |
| `{{DB_MONITOR_PASSWORD}}` | PostgreSQL `zbx_monitor` read-only user password `(vault)` | `<generated>` | 02, 09 |
| `{{PATRONI_API_PASSWORD}}` | Patroni REST API authentication password `(vault)` | `<generated>` | 04, 05 |
| `{{BACKUP_ENCRYPTION_KEY}}` | pgBackRest backup encryption passphrase `(vault)` | `<generated>` | 13 |

## Sizing

| Variable | Description | Example | Used In |
|----------|-------------|---------|---------|
| `{{SIZING_TIER}}` | Deployment tier: Small / Medium / Large / Very Large | `Medium` | 02, 06 |
| `{{EXPECTED_HOSTS}}` | Total monitored hosts | `1500` | 02, 06 |
| `{{EXPECTED_NVPS}}` | Expected New Values Per Second | `12000` | 02, 04, 06 |
| `{{INTER_SITE_RTT_MS}}` | Inter-site round-trip time (ms) | `3` | 04, 06 |
| `{{REPLICATION_MODE}}` | `synchronous` or `asynchronous` | `synchronous` | 04 |

## Database Connectivity

| Variable | Description | Example | Used In |
|----------|-------------|---------|---------|
| `{{DB_PORT}}` | Database client port (`5432` direct, `6432` via PgBouncer) | `5432` | 05, 06, 07 |

## VMware

| Variable | Description | Example | Used In |
|----------|-------------|---------|---------|
| `{{VCENTER_URL_A}}` | vCenter URL for Site A | `https://vcenter-a.example.com/sdk` | 05, 09 |
| `{{VCENTER_URL_B}}` | vCenter URL for Site B | `https://vcenter-b.example.com/sdk` | 05, 09 |
| `{{VCENTER_USER}}` | vCenter monitoring user | `zabbix-monitor@vsphere.local` | 09 |
| `{{VCENTER_PASSWORD}}` | vCenter monitoring user password `(vault)` | `<generated>` | 09 |
| `{{PORTGROUP_KEMP_A}}` | vSwitch port group for KEMP HA at Site A | `VLAN100-Zabbix` | 05 |
| `{{PORTGROUP_KEMP_B}}` | vSwitch port group for KEMP HA at Site B | `VLAN200-Zabbix` | 05 |

## TLS / SSL

| Variable | Description | Example | Used In |
|----------|-------------|---------|---------|
| `{{TLS_ORG_NAME}}` | Organization name for certificate subjects | `MyOrg` | 03 |
| `{{ETCD_CA_CERT_PATH}}` | Path to etcd CA certificate | `/etc/etcd/pki/ca.crt` | 03, 04, 09, 13, 14 |
| `{{ETCD_CA_KEY_PATH}}` | Path to etcd CA key | `/etc/etcd/pki/ca.key` | 03 |
| `{{SSL_CERT_PATH}}` | Frontend SSL certificate path (KEMP import) | `/path/to/zabbix.crt` | 05 |
| `{{SSL_KEY_PATH}}` | Frontend SSL private key path (KEMP import) | `/path/to/zabbix.key` | 05 |
| `{{SSL_CA_PATH}}` | CA certificate chain path | `/path/to/ca-chain.crt` | 05 |

## Backup

| Variable | Description | Example | Used In |
|----------|-------------|---------|---------|
| `{{BACKUP_TYPE}}` | `nfs` or `s3` | `nfs` | 13 |
| `{{BACKUP_NFS_SERVER}}` | NFS server hostname | `nfs-server.example.com` | 13 |
| `{{BACKUP_NFS_PATH}}` | NFS export path | `/pgbackrest` | 13 |
| `{{BACKUP_NFS_MOUNT}}` | Local mount point | `/mnt/pgbackrest-nfs` | 13 |
| `{{BACKUP_S3_ENDPOINT}}` | S3-compatible endpoint | `s3.your-provider.com` | 13 |
| `{{BACKUP_S3_BUCKET}}` | S3 bucket name | `pgbackrest-zabbix` | 13 |
| `{{BACKUP_S3_KEY}}` | S3 access key `(vault)` | `<access-key>` | 13 |
| `{{BACKUP_S3_SECRET}}` | S3 secret key `(vault)` | `<secret-key>` | 13 |
| `{{BACKUP_S3_REGION}}` | S3 region | `us-east-1` | 13 |
| `{{BACKUP_RETENTION_FULL}}` | Full backup retention count | `2` | 13 |
| `{{BACKUP_RETENTION_DIFF}}` | Differential backup retention count | `7` | 13 |

## Alerting

| Variable | Description | Example | Used In |
|----------|-------------|---------|---------|
| `{{SMTP_SERVER}}` | SMTP relay server hostname | `smtp.example.com` | 10 |
| `{{SMTP_FROM_EMAIL}}` | Zabbix email sender | `zabbix@example.com` | 10 |
| `{{SMTP_FROM_NAME}}` | Zabbix email display name | `Zabbix NOC` | 10 |
| `{{SMTP_HELO}}` | SMTP HELO domain | `example.com` | 10 |
| `{{TEAMS_WEBHOOK_URL}}` | Microsoft Teams incoming webhook URL `(vault)` | `https://...` | 10 |
| `{{SLACK_BOT_TOKEN}}` | Slack bot OAuth token `(vault)` | `xoxb-...` | 10 |
| `{{SLACK_CHANNEL}}` | Default Slack channel | `#zabbix-alerts` | 10 |
| `{{PAGERDUTY_INTEGRATION_KEY}}` | PagerDuty integration key `(vault)` | `<32-char hex>` | 10 |
| `{{ZABBIX_FRONTEND_URL}}` | Zabbix URL for alert links | `https://zabbix.example.com` | 07, 10 |

## Agent Deployment

| Variable | Description | Example | Used In |
|----------|-------------|---------|---------|
| `{{FILE_SHARE}}` | Windows network share for MSI distribution | `\\\\fileserver\\zabbix` | 11 |

## Kubernetes

| Variable | Description | Example | Used In |
|----------|-------------|---------|---------|
| `{{K8S_CLUSTER_NAME}}` | Kubernetes cluster display name | `prod-cluster-01` | 12 |
| `{{K8S_NAMESPACE}}` | Namespace for Zabbix monitoring | `monitoring` | 12 |
| `{{K8S_API_URL}}` | Kubernetes API server URL | `https://k8s-api.example.com:6443` | 12 |

## SNMP

| Variable | Description | Example | Used In |
|----------|-------------|---------|---------|
| `{{SNMPV3_SECURITY_NAME}}` | SNMPv3 security name | `zabbix-monitor` | 08, 09 |
| `{{SNMPV3_AUTH_PROTOCOL}}` | SNMPv3 auth protocol | `SHA256` | 08, 09 |
| `{{SNMPV3_AUTH_PASSPHRASE}}` | SNMPv3 auth passphrase `(vault)` | `<generated>` | 08, 09 |
| `{{SNMPV3_PRIV_PROTOCOL}}` | SNMPv3 privacy protocol | `AES256` | 08, 09 |
| `{{SNMPV3_PRIV_PASSPHRASE}}` | SNMPv3 privacy passphrase `(vault)` | `<generated>` | 08, 09 |

## Microsoft 365

| Variable | Description | Example | Used In |
|----------|-------------|---------|---------|
| `{{M365_TENANT_ID}}` | Azure AD / Entra tenant ID | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` | 09 |
| `{{M365_APP_ID}}` | Azure app registration client ID | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` | 09 |
| `{{M365_CLIENT_SECRET}}` | Azure app client secret `(vault)` | `<generated>` | 09 |

---

## Quick Reference — Variable Count by Category

| Category | Count | Sensitive |
|----------|-------|-----------|
| Network — Site A | 15 | No |
| Network — Site B | 14 | No |
| DNS | 10 | No |
| Credentials | 16 | Yes |
| Sizing | 5 | No |
| Database Connectivity | 1 | No |
| VMware | 6 | Partial |
| TLS/SSL | 6 | Partial |
| Backup | 11 | Partial |
| Alerting | 9 | Partial |
| Agent Deployment | 1 | No |
| Kubernetes | 3 | No |
| SNMP | 5 | Partial |
| Microsoft 365 | 3 | Partial |
| **Total** | **~105** | |
