# Zabbix 7.0 LTS — HA Deployment Template

Two-site, active/standby high-availability deployment of Zabbix 7.0 LTS across
on-premises VMware infrastructure. Monitors Windows, Linux, Kubernetes, and
Microsoft 365 workloads using PostgreSQL 16 + TimescaleDB, Patroni + etcd
clustering, and KEMP LoadMaster VLM load balancers.

## Architecture Summary

```
  Site A (Primary)                          Site B (DR / Standby)
 ┌─────────────────────────┐              ┌─────────────────────────┐
 │  KEMP VLM HA Pair       │              │  KEMP VLM HA Pair       │
 │  ┌───────┐ ┌──────────┐ │              │  ┌───────┐ ┌──────────┐ │
 │  │Front- │ │Zabbix Srv│ │              │  │Front- │ │Zabbix Srv│ │
 │  │end  A │ │Node-01   │ │              │  │end  B │ │Node-02   │ │
 │  │       │ │(Active)  │ │              │  │       │ │(Standby) │ │
 │  └───────┘ └────┬─────┘ │              │  └───────┘ └────┬─────┘ │
 │           ┌─────┴──────┐│              │           ┌─────┴──────┐│
 │           │ PostgreSQL ││   Streaming  │           │ PostgreSQL ││
 │           │ Primary    ││◄────────────►│           │ Standby    ││
 │           │ + Patroni  ││  Replication │           │ + Patroni  ││
 │           └────────────┘│              │           └────────────┘│
 │  etcd-1    etcd-2       │              │  etcd-3 (arbitrator)    │
 │  Proxy-A1  Proxy-A2     │              │  Proxy-B1  Proxy-B2     │
 └─────────────────────────┘              └─────────────────────────┘
```

| Component | Count | Role |
|---|---|---|
| KEMP LoadMaster VLM | 4 | HA pair per site (CARP) |
| Zabbix Server | 2 | Active/standby HA cluster |
| PostgreSQL 16 + TimescaleDB | 2 | Primary/standby via Patroni |
| etcd | 3 | Consensus for Patroni + Zabbix HA |
| Zabbix Frontend | 2 | Nginx + PHP-FPM behind KEMP VIP |
| Zabbix Proxies | 4+ | Proxy groups, 2+ per site |
| Zabbix Agents | All hosts | DaemonSet (K8s) / package (VMs) |

## Repository Structure

```
├── README.md                          ← You are here
├── Zabbix_Deployment_Guide.md         ← Master design reference (full architecture doc)
└── implementation/
    ├── 00-environment-variables.md
    ├── 01-prerequisites-and-dns.md
    ├── 02-base-os-and-packages.md
    ├── 03-etcd-cluster.md
    ├── 04-database-layer.md
    ├── 05-kemp-loadmaster.md
    ├── 06-zabbix-server-ha.md
    ├── 07-zabbix-frontend.md
    ├── 08-proxy-architecture.md
    ├── 09-templates-and-m365.md
    ├── 10-alerting-and-notifications.md
    ├── 11-agent-rollout.md
    ├── 12-kubernetes-monitoring.md
    ├── 13-backup-and-dr.md
    ├── 14-validation-and-ha-testing.md
    ├── 15-day2-operations.md
    └── appendix/
        ├── A-firewall-port-matrix.md
        ├── B-vm-inventory-worksheet.md
        └── C-security-checklist.md
```

## Getting Started

Before deploying, complete these prerequisites in order:

1. **Fill out environment variables** — Open [`implementation/00-environment-variables.md`](implementation/00-environment-variables.md) and populate every `{{VARIABLE_NAME}}` placeholder with your site-specific values (IPs, hostnames, credentials, paths).
2. **Review the design reference** — Read [`Zabbix_Deployment_Guide.md`](Zabbix_Deployment_Guide.md) to understand architectural decisions and sizing guidance.
3. **Provision VMs** — Deploy all virtual machines per the inventory in [`Appendix B — VM Inventory Worksheet`](implementation/appendix/B-vm-inventory-worksheet.md).
4. **Submit firewall rules** — Request all required network flows listed in [`Appendix A — Firewall Port Matrix`](implementation/appendix/A-firewall-port-matrix.md).

## Step-by-Step Deployment

Execute phases sequentially unless noted otherwise. Each phase document contains
copy-paste-ready commands with `{{PLACEHOLDER}}` variables that resolve from Phase 0.

| # | Phase | What It Accomplishes | Depends On |
|---|---|---|---|
| 00 | [Environment Variables](implementation/00-environment-variables.md) | Defines all site-specific values used by every subsequent phase | — |
| 01 | [Prerequisites and DNS](implementation/01-prerequisites-and-dns.md) | DNS records, NTP, firewall rules, repo configuration | 00 |
| 02 | [Base OS and Packages](implementation/02-base-os-and-packages.md) | RHEL hardening, Zabbix/PostgreSQL repos, base packages | 01 |
| 03 | [etcd Cluster](implementation/03-etcd-cluster.md) | Three-node etcd cluster for Patroni and Zabbix HA consensus | 02 |
| 04 | [Database Layer](implementation/04-database-layer.md) | PostgreSQL 16 + TimescaleDB with Patroni HA and PgBouncer | 03 |
| 05 | [KEMP LoadMaster](implementation/05-kemp-loadmaster.md) | Load-balancer HA pairs with VIPs for frontend and database | 02 |
| 06 | [Zabbix Server HA](implementation/06-zabbix-server-ha.md) | Two-node Zabbix server cluster (active/standby) | 04, 05 |
| 07 | [Zabbix Frontend](implementation/07-zabbix-frontend.md) | Nginx + PHP-FPM frontend behind KEMP VIP | 06 |
| 08 | [Proxy Architecture](implementation/08-proxy-architecture.md) | Proxy groups at each site for distributed data collection | 06 |
| 09 | [Templates and M365](implementation/09-templates-and-m365.md) | Monitoring templates, host groups, Microsoft 365 integration | 07 |
| 10 | [Alerting and Notifications](implementation/10-alerting-and-notifications.md) | Media types, actions, escalations, maintenance windows | 07 |
| 11 | [Agent Rollout](implementation/11-agent-rollout.md) | Zabbix Agent 2 deployment on Windows and Linux hosts | 08 |
| 12 | [Kubernetes Monitoring](implementation/12-kubernetes-monitoring.md) | Helm-based in-cluster proxy and K8s template deployment | 08 |
| 13 | [Backup and DR](implementation/13-backup-and-dr.md) | Backup jobs, retention policies, DR failover procedures | 06 |
| 14 | [Validation and HA Testing](implementation/14-validation-and-ha-testing.md) | End-to-end validation, HA failover tests, smoke tests | 11, 12, 13 |
| 15 | [Day-2 Operations](implementation/15-day2-operations.md) | Upgrades, housekeeping, capacity reviews, runbook procedures | 14 |

### Phase Dependency Diagram

```
00 ──► 01 ──► 02 ──┬──► 03 ──► 04 ──┐
                   │                 ├──► 06 ──┬──► 07 ──┬──► 09 *
                   └──► 05 ──────────┘         │         └──► 10 *
                                               ├──► 08 ──┬──► 11
                                               │         └──► 12 *
                                               └──► 13
                                                          ┌──── all
                                                          ▼
                                                    14 ──► 15

  * Phases 09, 10, and 12 can run in parallel with each other.
```

## Appendices

| Document | Description |
|---|---|
| [Appendix A — Firewall Port Matrix](implementation/appendix/A-firewall-port-matrix.md) | Every required network flow with source, destination, port, and protocol |
| [Appendix B — VM Inventory Worksheet](implementation/appendix/B-vm-inventory-worksheet.md) | VM specifications, hostnames, IPs, and resource allocations |
| [Appendix C — Security Checklist](implementation/appendix/C-security-checklist.md) | Pre-go-live security verification checklist |

## Post-Deployment

After completing all phases:

1. **Validate** — Run the full test suite in [Phase 14](implementation/14-validation-and-ha-testing.md): HA failover, data-integrity checks, and end-to-end smoke tests.
2. **Security review** — Walk through [Appendix C — Security Checklist](implementation/appendix/C-security-checklist.md) before handing over to production.
3. **Operational handoff** — Follow [Phase 15](implementation/15-day2-operations.md) for upgrade procedures, housekeeping schedules, and capacity review cadence.

## Placeholder Convention

All implementation guides use `{{VARIABLE_NAME}}` placeholders for site-specific
values (IPs, hostnames, credentials, paths). Before executing any phase, replace
these placeholders with the values defined in
[`Phase 0 — Environment Variables`](implementation/00-environment-variables.md).

Example:
```bash
# In the guide:
psql -h {{PG_PRIMARY_IP}} -U {{ZABBIX_DB_USER}} -d {{ZABBIX_DB_NAME}}

# After substitution:
psql -h 10.1.1.50 -U zabbix -d zabbix
```
