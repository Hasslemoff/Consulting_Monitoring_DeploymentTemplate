# Appendix B — VM Inventory Worksheet

> **Purpose:** Bill of materials and provisioning worksheet for the VMware/infrastructure team. Fill in IP addresses, VLAN IDs, and check off each VM as provisioned. Default specifications are for the **Medium** sizing tier (500-2,000 monitored hosts). Adjust per the sizing tier reference table if your deployment requires a different tier.

---

## How to Use

1. Review the sizing tier reference table (below) and confirm `{{SIZING_TIER}}` with the project lead
2. Adjust vCPU, RAM, and storage values if using a tier other than Medium
3. Fill in all `{{VARIABLE}}` placeholders with values from [Phase 0 — Environment Variables](../00-environment-variables.md)
4. Provision each VM and check the **Status** box when complete
5. Return this completed worksheet to the Zabbix deployment team before Phase 2 begins

---

## Site A VM Inventory

**Site:** Site A (Primary)
**vCenter:** `{{VCENTER_URL_A}}`
**VLAN:** `{{SITE_A_VLAN_ID}}`
**Subnet:** `{{SITE_A_SUBNET}}`
**Gateway:** `{{SITE_A_GATEWAY}}`

| VM Name | Role | vCPU | RAM | Storage | Storage Type | IP Address | VLAN | OS | Status |
|---------|------|------|-----|---------|--------------|------------|------|----|--------|
| zabbix-node-siteA | Zabbix Server (HA Node 1) | 8 | 32 GB | 100 GB | SSD | `{{ZBX_SERVER_A_IP}}` | `{{SITE_A_VLAN_ID}}` | RHEL 9 / Rocky 9 | [ ] Provisioned |
| pg-siteA | PostgreSQL + TimescaleDB + Patroni | 8 | 32 GB | 500 GB | NVMe / SSD | `{{PG_A_IP}}` | `{{SITE_A_VLAN_ID}}` | RHEL 9 / Rocky 9 | [ ] Provisioned |
| etcd-1 | etcd (consensus node 1) | 2 | 4 GB | 20 GB | SSD | `{{ETCD_1_IP}}` | `{{SITE_A_VLAN_ID}}` | RHEL 9 / Rocky 9 | [ ] Provisioned |
| frontend-siteA | Zabbix Frontend (Apache/Nginx + PHP) | 2 | 4 GB | 20 GB | Any | `{{FRONTEND_A_IP}}` | `{{SITE_A_VLAN_ID}}` | RHEL 9 / Rocky 9 | [ ] Provisioned |
| kemp-a1 | KEMP LoadMaster VLM (Active) | 2 | 4 GB | 32 GB | Thick Eager Zeroed | `{{KEMP_A1_IP}}` | `{{SITE_A_VLAN_ID}}` | KEMP Appliance | [ ] Provisioned |
| kemp-a2 | KEMP LoadMaster VLM (Passive) | 2 | 4 GB | 32 GB | Thick Eager Zeroed | `{{KEMP_A2_IP}}` | `{{SITE_A_VLAN_ID}}` | KEMP Appliance | [ ] Provisioned |
| proxy-siteA-01 | Zabbix Proxy (proxy group member) | 4 | 8 GB | 50 GB | Any | `{{PROXY_A1_IP}}` | `{{SITE_A_VLAN_ID}}` | RHEL 9 / Rocky 9 | [ ] Provisioned |
| proxy-siteA-02 | Zabbix Proxy (proxy group member) | 4 | 8 GB | 50 GB | Any | `{{PROXY_A2_IP}}` | `{{SITE_A_VLAN_ID}}` | RHEL 9 / Rocky 9 | [ ] Provisioned |
| proxy-siteA-snmptrap | SNMP Trap Proxy (standalone) | 2 | 4 GB | 20 GB | Any | `{{PROXY_A_SNMPTRAP_IP}}` | `{{SITE_A_VLAN_ID}}` | RHEL 9 / Rocky 9 | [ ] Provisioned |

### Site A Totals

| Resource | Value |
|----------|-------|
| VMs | 9 |
| vCPU | 34 |
| RAM | 100 GB |
| Storage | ~824 GB |

---

## Site B VM Inventory

**Site:** Site B (Secondary / DR)
**vCenter:** `{{VCENTER_URL_B}}`
**VLAN:** `{{SITE_B_VLAN_ID}}`
**Subnet:** `{{SITE_B_SUBNET}}`
**Gateway:** `{{SITE_B_GATEWAY}}`

| VM Name | Role | vCPU | RAM | Storage | Storage Type | IP Address | VLAN | OS | Status |
|---------|------|------|-----|---------|--------------|------------|------|----|--------|
| zabbix-node-siteB | Zabbix Server (HA Node 2) | 8 | 32 GB | 100 GB | SSD | `{{ZBX_SERVER_B_IP}}` | `{{SITE_B_VLAN_ID}}` | RHEL 9 / Rocky 9 | [ ] Provisioned |
| pg-siteB | PostgreSQL + TimescaleDB + Patroni | 8 | 32 GB | 500 GB | NVMe / SSD | `{{PG_B_IP}}` | `{{SITE_B_VLAN_ID}}` | RHEL 9 / Rocky 9 | [ ] Provisioned |
| etcd-2 | etcd (consensus node 2) | 2 | 4 GB | 20 GB | SSD | `{{ETCD_2_IP}}` | `{{SITE_B_VLAN_ID}}` | RHEL 9 / Rocky 9 | [ ] Provisioned |
| frontend-siteB | Zabbix Frontend (Apache/Nginx + PHP) | 2 | 4 GB | 20 GB | Any | `{{FRONTEND_B_IP}}` | `{{SITE_B_VLAN_ID}}` | RHEL 9 / Rocky 9 | [ ] Provisioned |
| kemp-b1 | KEMP LoadMaster VLM (Active) | 2 | 4 GB | 32 GB | Thick Eager Zeroed | `{{KEMP_B1_IP}}` | `{{SITE_B_VLAN_ID}}` | KEMP Appliance | [ ] Provisioned |
| kemp-b2 | KEMP LoadMaster VLM (Passive) | 2 | 4 GB | 32 GB | Thick Eager Zeroed | `{{KEMP_B2_IP}}` | `{{SITE_B_VLAN_ID}}` | KEMP Appliance | [ ] Provisioned |
| proxy-siteB-01 | Zabbix Proxy (proxy group member) | 4 | 8 GB | 50 GB | Any | `{{PROXY_B1_IP}}` | `{{SITE_B_VLAN_ID}}` | RHEL 9 / Rocky 9 | [ ] Provisioned |
| proxy-siteB-02 | Zabbix Proxy (proxy group member) | 4 | 8 GB | 50 GB | Any | `{{PROXY_B2_IP}}` | `{{SITE_B_VLAN_ID}}` | RHEL 9 / Rocky 9 | [ ] Provisioned |
| proxy-siteB-snmptrap | SNMP Trap Proxy (standalone) | 2 | 4 GB | 20 GB | Any | `{{PROXY_B_SNMPTRAP_IP}}` | `{{SITE_B_VLAN_ID}}` | RHEL 9 / Rocky 9 | [ ] Provisioned |

### Site B Totals

| Resource | Value |
|----------|-------|
| VMs | 9 |
| vCPU | 34 |
| RAM | 100 GB |
| Storage | ~824 GB |

---

## Arbitrator Node

The etcd arbitrator provides the tiebreaker vote for the 3-node etcd consensus cluster. It may be placed at Site A, Site B, or a third location. It does not store PostgreSQL data.

| VM Name | Role | vCPU | RAM | Storage | Storage Type | IP Address | VLAN | OS | Status |
|---------|------|------|-----|---------|--------------|------------|------|----|--------|
| etcd-3 | etcd Arbitrator (consensus node 3) | 2 | 4 GB | 20 GB | SSD | `{{ETCD_3_IP}}` | _______ | RHEL 9 / Rocky 9 | [ ] Provisioned |

> **Placement note:** If etcd-3 is placed at a third location, ensure WAN connectivity to both `{{ETCD_1_IP}}` and `{{ETCD_2_IP}}` on ports 2379 and 2380. If placed at Site A alongside etcd-1, the cluster can tolerate the loss of Site B but not Site A (two of three nodes would be at Site A).

---

## Combined Totals

| Resource | Site A | Site B | Arbitrator | Total |
|----------|--------|--------|------------|-------|
| VMs | 9 | 9 | 1 | **19** |
| vCPU | 34 | 34 | 2 | **70** |
| RAM | 100 GB | 100 GB | 4 GB | **204 GB** |
| Storage | ~824 GB | ~824 GB | 20 GB | **~1,668 GB (~1.7 TB)** |

> **Note:** These totals reflect the Medium tier baseline with 2 proxies per site. Additional proxy VMs increase totals by 4 vCPU, 8 GB RAM, and 50 GB storage per proxy.

---

## Sizing Tier Reference

Adjust the Zabbix Server and Database VM specifications based on your expected host count. KEMP, etcd, frontend, and proxy specifications remain constant across all tiers.

### Zabbix Server VM Sizing

| Tier | Host Count | vCPU | RAM | Storage | Storage Type |
|------|------------|------|-----|---------|--------------|
| Small | < 500 | 4 | 16 GB | 50 GB | SSD |
| **Medium** | **500 - 2,000** | **8** | **32 GB** | **100 GB** | **SSD** |
| Large | 2,000 - 10,000 | 16 | 64 GB | 200 GB | SSD |
| Very Large | 10,000+ | 32 | 96 GB | 200 GB | SSD |

### Database VM Sizing (PostgreSQL + TimescaleDB)

| Tier | Host Count | vCPU | RAM | Storage | Storage Type |
|------|------------|------|-----|---------|--------------|
| Small | < 500 | 4 | 16 GB | 200 GB | SSD |
| **Medium** | **500 - 2,000** | **8** | **32 GB** | **500 GB** | **NVMe / SSD** |
| Large | 2,000 - 10,000 | 16 | 64 - 128 GB | 1 - 2 TB | NVMe |
| Very Large | 10,000+ | 32+ | 128 - 256 GB | Multi-TB | NVMe (RAID 10) |

### Components with Fixed Sizing (All Tiers)

| Component | vCPU | RAM | Storage | Storage Type | Notes |
|-----------|------|-----|---------|--------------|-------|
| KEMP LoadMaster VLM | 2 | 4 GB | 32 GB | Thick Eager Zeroed | Per appliance (4 total) |
| etcd | 2 | 4 GB | 20 GB | SSD | Per node (3 total) |
| Frontend | 2 | 4 GB | 20 GB | Any | Per site (2 total) |
| Proxy (standard) | 4 | 8 GB | 50 GB | Any | Per instance (2+ per site) |
| Proxy (SNMP trap) | 2 | 4 GB | 20 GB | Any | Per site (1 per site) |

> **Scaling note:** For the Large and Very Large tiers, consider adding more proxy VMs per site to distribute the polling workload. Each proxy group can have 2-5+ members depending on the number of monitored hosts assigned to that group.

---

## VMware Requirements Checklist

Complete these VMware-specific requirements before deploying VMs.

### Port Group Security (Required for KEMP HA)

- [ ] Site A port group `{{PORTGROUP_KEMP_A}}` — MAC Address Changes = **Accept**
- [ ] Site A port group `{{PORTGROUP_KEMP_A}}` — Forged Transmits = **Accept**
- [ ] Site B port group `{{PORTGROUP_KEMP_B}}` — MAC Address Changes = **Accept**
- [ ] Site B port group `{{PORTGROUP_KEMP_B}}` — Forged Transmits = **Accept**

> **Warning:** If these settings are left at the default (Reject), KEMP CARP failover will silently fail. The passive unit will never assume the VIP.

### KEMP Appliance Requirements

- [ ] All KEMP VMs use **VMXNET3** NIC adapters (not E1000 or E1000e)
- [ ] All KEMP VMs have **static MAC addresses** assigned in vCenter (prevents MAC conflicts during CARP failover)
- [ ] All KEMP VM disks are **Thick Provision Eager Zeroed** (required for appliance stability)
- [ ] KEMP VMs are deployed from the official KEMP LoadMaster VLM OVA/OVF image

### Storage Requirements

- [ ] Database VMs (`pg-siteA`, `pg-siteB`) placed on **NVMe or SSD-backed** datastores
- [ ] etcd VMs (`etcd-1`, `etcd-2`, `etcd-3`) placed on **SSD-backed** datastores
- [ ] Database VM datastores have sufficient IOPS for the expected workload (see sizing tier)
- [ ] Storage latency to database VMs confirmed < 2ms (critical for TimescaleDB performance)

### High Availability (vSphere HA / DRS)

- [ ] DRS anti-affinity rule created: `kemp-a1` and `kemp-a2` must run on **separate ESXi hosts** (prevents single-host failure from taking down both HA units)
- [ ] DRS anti-affinity rule created: `kemp-b1` and `kemp-b2` must run on **separate ESXi hosts**
- [ ] vSphere HA restart priority set to **High** for database and Zabbix server VMs
- [ ] vSphere HA restart priority set to **Medium** for KEMP and frontend VMs

### Network

- [ ] VLANs `{{SITE_A_VLAN_ID}}` and `{{SITE_B_VLAN_ID}}` configured and trunked to all relevant ESXi hosts
- [ ] Jumbo frames enabled on storage network (if using iSCSI/NFS for database datastores)
- [ ] NTP configured on all ESXi hosts (time consistency critical for HA)

---

## Sign-Off

| Milestone | Responsible | Date | Status |
|-----------|-------------|------|--------|
| Site A VMs provisioned | _______ | _______ | [ ] Complete |
| Site B VMs provisioned | _______ | _______ | [ ] Complete |
| Arbitrator VM provisioned | _______ | _______ | [ ] Complete |
| VMware requirements verified | _______ | _______ | [ ] Complete |
| IP addresses confirmed and documented | _______ | _______ | [ ] Complete |
| Worksheet returned to deployment team | _______ | _______ | [ ] Complete |
