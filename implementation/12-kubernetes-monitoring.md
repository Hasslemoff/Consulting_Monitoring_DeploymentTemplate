# Phase 12 — Kubernetes Monitoring

## Objective

Deploy Zabbix monitoring components into Kubernetes clusters using the official Helm chart, configure cluster state and node monitoring templates, and establish visibility into pods, deployments, persistent volumes, and control plane components.

## Prerequisites

- [ ] Phase 6 complete — Zabbix Server running and accessible
- [ ] `kubectl` configured with cluster-admin access to the target Kubernetes cluster
- [ ] `helm` v3 installed on the workstation or jump host
- [ ] Outbound TCP/10051 from the Kubernetes cluster to the Zabbix Server (both HA nodes)
- [ ] Kubernetes API server URL and port known (`{{K8S_API_URL}}`)
- [ ] Namespace for Zabbix monitoring components decided (`{{K8S_NAMESPACE}}`)

> **Note:** This phase can run in parallel with Phases 09 and 10. It has no dependencies on template imports or alerting configuration (though templates must be present before creating hosts in Step 5).

---

## Step 1 — Add the Zabbix Helm Repository

Add the official Zabbix Helm chart repository and update the local chart index:

```bash
helm repo add zabbix-chart-7.2 https://cdn.zabbix.com/zabbix/integrations/kubernetes-helm/7.2
helm repo update
```

Verify the chart is available:

```bash
helm search repo zabbix-chart-7.2
```

Expected output should list `zabbix-chart-7.2/zabbix-helm-chrt` with a version matching your Zabbix deployment.

---

## Step 2 — Export and Customize Helm Values

### 2a. Export Default Values

Export the full default values file for reference and customization:

```bash
helm show values zabbix-chart-7.2/zabbix-helm-chrt > k8s-zabbix-values.yaml
```

### 2b. Customize Key Values

Edit `k8s-zabbix-values.yaml` and set the following values. All other values can remain at their defaults unless your environment requires changes.

**Zabbix Proxy Configuration:**

```yaml
zabbixProxy:
  enabled: true
  resources:
    requests:
      memory: 256Mi
      cpu: 250m
    limits:
      memory: 512Mi
      cpu: 500m
  env:
    # Both HA nodes separated by semicolons for active proxy failover
    - name: ZBX_SERVER_HOST
      value: "{{ZBX_SERVER_A_IP}};{{ZBX_SERVER_B_IP}}"
    # Active proxy mode (proxy pushes data to server)
    - name: ZBX_PROXYMODE
      value: "0"
    # Cache size for configuration data
    - name: ZBX_CACHESIZE
      value: "256M"
    # Proxy hostname as it will appear in Zabbix frontend
    - name: ZBX_HOSTNAME
      value: "k8s-proxy-{{K8S_CLUSTER_NAME}}"
    # Configuration sync frequency
    - name: ZBX_CONFIGFREQUENCY
      value: "60"
```

**Zabbix Agent Configuration:**

```yaml
zabbixAgent:
  enabled: true
  resources:
    requests:
      memory: 128Mi
      cpu: 100m
    limits:
      memory: 256Mi
      cpu: 200m
  # Deploy agent to ALL nodes including control-plane
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
    - key: node-role.kubernetes.io/master
      operator: Exists
      effect: NoSchedule
```

**kube-state-metrics Configuration:**

```yaml
kubeStateMetrics:
  # Set to false if kube-state-metrics is already deployed in your cluster
  # (e.g., by Prometheus or another monitoring stack)
  enabled: true
```

> **Important:** Running two instances of kube-state-metrics in the same cluster can cause duplicate metrics and increased API server load. Check for existing deployments first:
> ```bash
> kubectl get deployment -A | grep kube-state-metrics
> ```
> If output shows an existing deployment, set `enabled: false`.

**Service Account Configuration:**

```yaml
rbac:
  create: true
  # The Helm chart creates a service account with the RBAC roles needed
  # to query the Kubernetes API for cluster state data
serviceAccount:
  create: true
  name: "zabbix-service-account"
```

### 2c. Review the Complete Values File

Before installing, review the values file for any environment-specific adjustments:

```bash
# Quick check — all customized values
grep -n "{{" k8s-zabbix-values.yaml
```

Ensure all `{{VARIABLE}}` placeholders have been replaced with actual values.

---

## Step 3 — Install the Helm Chart

### 3a. Create the Namespace

```bash
kubectl create namespace {{K8S_NAMESPACE}}
```

### 3b. Install the Chart

```bash
helm install zabbix zabbix-chart-7.2/zabbix-helm-chrt \
  --dependency-update \
  -f k8s-zabbix-values.yaml \
  -n {{K8S_NAMESPACE}}
```

### 3c. Verify Deployment

Wait for all pods to reach Running state:

```bash
kubectl get pods -n {{K8S_NAMESPACE}} -w
```

Expected pods (names will include random suffixes):

| Pod | Role | Expected State |
|-----|------|----------------|
| `zabbix-zabbix-helm-chrt-proxy-*` | Zabbix Proxy | Running |
| `zabbix-zabbix-helm-chrt-agent-*` | Zabbix Agent (DaemonSet) | Running (one per node) |
| `zabbix-zabbix-helm-chrt-kube-state-metrics-*` | kube-state-metrics | Running (if enabled) |

Verify logs for the proxy pod:

```bash
kubectl logs -n {{K8S_NAMESPACE}} -l app=zabbix-zabbix-helm-chrt-proxy --tail=50
```

Look for:
- `proxy #0 started` — proxy process is running
- `sending configuration data to server` — proxy connected to Zabbix Server

---

## Step 4 — Retrieve the Service Account Token

The Kubernetes cluster state templates require an API token to authenticate against the Kubernetes API server.

### 4a. Get the Token

```bash
kubectl get secret zabbix-service-account \
  -n {{K8S_NAMESPACE}} \
  -o jsonpath='{.data.token}' | base64 -d
```

If the secret does not exist (Kubernetes 1.24+ may not auto-create secrets for service accounts), create a token manually:

```bash
kubectl create token zabbix-service-account \
  -n {{K8S_NAMESPACE}} \
  --duration=8760h
```

> **Note:** The `--duration=8760h` flag creates a token valid for 1 year. Set a calendar reminder to rotate this token before expiration. For production environments, consider using shorter-lived tokens with automated rotation.

### 4b. Store the Token

Copy the token output and store it securely. You will enter this as a Zabbix host macro (`{$KUBE.API.TOKEN}`) in the next step.

> **Warning:** This token has cluster-wide read access as defined by the Helm chart's RBAC roles. Treat it as a sensitive credential.

---

## Step 5 — Create Hosts in Zabbix Frontend

### 5a. Kubernetes Cluster State Host

This host monitors overall cluster health: deployments, pods, persistent volumes, jobs, and namespaces.

1. Navigate to **Data collection > Hosts > Create host**
2. Configure:

| Field | Value |
|-------|-------|
| Host name | `K8s Cluster State - {{K8S_CLUSTER_NAME}}` |
| Visible name | `Kubernetes Cluster — {{K8S_CLUSTER_NAME}}` |
| Host groups | `Kubernetes/{{K8S_CLUSTER_NAME}}` (create if needed) |
| Interfaces | None required (HTTP agent items query the API directly) |
| Monitored by proxy group | `SiteA-Proxies` (or the proxy group closest to the cluster) |

3. Link template: **Kubernetes cluster state by HTTP**

4. Set macros:

| Macro | Value | Type |
|-------|-------|------|
| `{$KUBE.API.URL}` | `{{K8S_API_URL}}` | Text |
| `{$KUBE.API.TOKEN}` | (token from Step 4) | Secret |

### 5b. Kubernetes Nodes Host

This host discovers and monitors individual cluster nodes (CPU, memory, conditions, kubelet health).

1. Navigate to **Data collection > Hosts > Create host**
2. Configure:

| Field | Value |
|-------|-------|
| Host name | `K8s Nodes - {{K8S_CLUSTER_NAME}}` |
| Visible name | `Kubernetes Nodes — {{K8S_CLUSTER_NAME}}` |
| Host groups | `Kubernetes/{{K8S_CLUSTER_NAME}}` |
| Interfaces | None required |
| Monitored by proxy group | `SiteA-Proxies` |

3. Link template: **Kubernetes nodes by HTTP**

4. Set macros (same as cluster state host):

| Macro | Value | Type |
|-------|-------|------|
| `{$KUBE.API.URL}` | `{{K8S_API_URL}}` | Text |
| `{$KUBE.API.TOKEN}` | (token from Step 4) | Secret |

### 5c. Auto-Discovered Control Plane Hosts

The Kubernetes cluster state template automatically discovers control plane components and creates monitoring items for:

| Component | Template | What It Monitors |
|-----------|----------|-----------------|
| kubelet | Kubernetes kubelet by HTTP | Container runtime, pod lifecycle, resource usage |
| API server | Kubernetes API server by HTTP | Request latency, error rate, etcd health |
| Controller Manager | Kubernetes Controller Manager by HTTP | Work queue depth, reconciliation latency |
| Scheduler | Kubernetes Scheduler by HTTP | Scheduling latency, pending pods |

These hosts are created automatically through low-level discovery. No manual configuration is needed for the control plane components.

---

## Step 6 — Configure Namespace Filtering

For large clusters with many namespaces, filtering discovery to relevant namespaces reduces Zabbix Server load and keeps the host inventory manageable.

### 6a. Set Discovery Filter Macros

On the **K8s Cluster State** host, navigate to **Macros** and add:

| Macro | Value | Purpose |
|-------|-------|---------|
| `{$KUBE.LLD.FILTER.POD.NAMESPACE.MATCHES}` | `production\|staging` | Only discover pods in these namespaces |
| `{$KUBE.LLD.FILTER.POD.NAMESPACE.NOT_MATCHES}` | `kube-system\|monitoring\|kube-public\|kube-node-lease` | Exclude system namespaces from discovery |

> **Note:** Use regex pipe syntax (`namespace1\|namespace2`) for multiple namespace matches. The `MATCHES` macro takes precedence — if set, only matching namespaces are included.

### 6b. Additional Filter Macros

| Macro | Default | Description |
|-------|---------|-------------|
| `{$KUBE.LLD.FILTER.DEPLOYMENT.NAMESPACE.MATCHES}` | `.*` | Filter deployments by namespace |
| `{$KUBE.LLD.FILTER.PV.NAME.MATCHES}` | `.*` | Filter persistent volumes by name |
| `{$KUBE.LLD.FILTER.NODE.NAME.MATCHES}` | `.*` | Filter nodes by name |

Adjust these macros based on your cluster size and monitoring requirements. The defaults (`.*`) match everything.

---

## Step 7 — Add Custom Triggers for Known Gaps

The default Kubernetes templates cover many scenarios but have known gaps for certain failure conditions. Add these custom triggers to improve detection.

### 7a. OOMKilled Detection

Create a trigger on the **K8s Cluster State** host:

| Field | Value |
|-------|-------|
| Name | `K8s: OOMKilled container detected in {#NAMESPACE}/{#POD}` |
| Severity | High |
| Expression | `find(/K8s Cluster State - {{K8S_CLUSTER_NAME}}/kube.pod.containers.restarts[{#NAMESPACE}/{#POD}],#2,"regexp","OOMKilled")=1` |
| Description | A container was terminated due to exceeding its memory limit. Review the pod's memory requests/limits and application memory usage. |

> **Note:** The exact item key depends on your template version. Verify the available items via **Monitoring > Latest data** after initial data collection.

### 7b. ImagePullBackOff Detection

| Field | Value |
|-------|-------|
| Name | `K8s: ImagePullBackOff on {#NAMESPACE}/{#POD}` |
| Severity | High |
| Expression | `find(/K8s Cluster State - {{K8S_CLUSTER_NAME}}/kube.pod.status.phase[{#NAMESPACE}/{#POD}],#1,"regexp","ImagePullBackOff")=1` |
| Description | Pod cannot pull container image. Check image name, tag, and registry authentication. |

### 7c. CrashLoopBackOff Detection

This is a known gap in some Zabbix Kubernetes template versions (reference: ZBX-21571).

| Field | Value |
|-------|-------|
| Name | `K8s: CrashLoopBackOff on {#NAMESPACE}/{#POD}` |
| Severity | High |
| Expression | `last(/K8s Cluster State - {{K8S_CLUSTER_NAME}}/kube.pod.containers.restarts[{#NAMESPACE}/{#POD}])>5 and change(/K8s Cluster State - {{K8S_CLUSTER_NAME}}/kube.pod.containers.restarts[{#NAMESPACE}/{#POD}])>0` |
| Description | Pod is repeatedly crashing and restarting. Check container logs for the root cause. |

### 7d. Pod Eviction Monitoring

| Field | Value |
|-------|-------|
| Name | `K8s: Pod eviction detected in namespace {#NAMESPACE}` |
| Severity | Warning |
| Expression | `find(/K8s Cluster State - {{K8S_CLUSTER_NAME}}/kube.pod.status.phase[{#NAMESPACE}/{#POD}],#1,"regexp","Evicted")=1` |
| Description | Pod was evicted, likely due to node resource pressure. Check node conditions for memory or disk pressure. |

### 7e. Persistent Volume Utilization

| Field | Value |
|-------|-------|
| Name | `K8s: PV {#PV_NAME} usage > 85%` |
| Severity | Warning |
| Expression | `last(/K8s Cluster State - {{K8S_CLUSTER_NAME}}/kube.pv.capacity.used.percent[{#PV_NAME}])>85` |
| Description | Persistent volume is running low on space. Plan for expansion or cleanup. |

| Field | Value |
|-------|-------|
| Name | `K8s: PV {#PV_NAME} usage > 95%` |
| Severity | High |
| Expression | `last(/K8s Cluster State - {{K8S_CLUSTER_NAME}}/kube.pv.capacity.used.percent[{#PV_NAME}])>95` |
| Description | Persistent volume is critically low. Immediate action required to prevent data loss or application failure. |

---

## Step 8 — What Gets Monitored (Summary)

After completing this phase, the following Kubernetes components are under monitoring:

### Cluster Level

| Category | Metrics | Source |
|----------|---------|--------|
| Deployments | Replicas desired/available/unavailable, rollout status | Kubernetes cluster state by HTTP |
| Pods | Phase (Running/Pending/Failed), restart count, container status | Kubernetes cluster state by HTTP |
| Persistent Volumes | Capacity, used, available, status (Bound/Released) | Kubernetes cluster state by HTTP |
| Jobs | Active, succeeded, failed, completion time | Kubernetes cluster state by HTTP |
| Namespaces | Pod counts, resource quotas | Kubernetes cluster state by HTTP |
| DaemonSets | Desired/ready/available, misscheduled | Kubernetes cluster state by HTTP |
| StatefulSets | Replicas desired/ready | Kubernetes cluster state by HTTP |

### Node Level

| Category | Metrics | Source |
|----------|---------|--------|
| CPU | Utilization, allocatable, capacity | Kubernetes nodes by HTTP + Agent |
| Memory | Utilization, allocatable, capacity, pressure | Kubernetes nodes by HTTP + Agent |
| Disk | Utilization, pressure | Kubernetes nodes by HTTP + Agent |
| Conditions | Ready, MemoryPressure, DiskPressure, PIDPressure | Kubernetes nodes by HTTP |
| Network | Pod CIDR, InternalIP | Kubernetes nodes by HTTP |

### Control Plane

| Component | Key Metrics | Source |
|-----------|-------------|--------|
| API Server | Request latency, error rate, current inflight requests | Kubernetes API server by HTTP |
| Controller Manager | Work queue depth, reconciliation errors | Kubernetes Controller Manager by HTTP |
| Scheduler | Scheduling latency, pending pods, preemption | Kubernetes Scheduler by HTTP |
| kubelet | Running pods/containers, runtime operations, PLEG duration | Kubernetes kubelet by HTTP |

### Pod Level

| Category | Metrics | Source |
|----------|---------|--------|
| Phase | Running, Pending, Succeeded, Failed, Unknown | Kubernetes cluster state by HTTP |
| Restarts | Container restart count | Kubernetes cluster state by HTTP |
| Containers | Ready count, waiting reason, terminated reason | Kubernetes cluster state by HTTP |
| Resource usage | CPU/memory requests and limits vs actual | kube-state-metrics + kubelet |

---

## Verification Checkpoint

Complete all checks before marking this phase done.

| # | Check | How to Verify | Expected Result | Pass |
|---|-------|---------------|-----------------|------|
| 1 | Helm chart deployed | `kubectl get pods -n {{K8S_NAMESPACE}}` | All pods in Running state | [ ] |
| 2 | In-cluster proxy connecting | Proxy pod logs: `kubectl logs -n {{K8S_NAMESPACE}} -l app=*proxy*` | `sending configuration data to server` | [ ] |
| 3 | Proxy registered in Zabbix | Administration > Proxies | Proxy `k8s-proxy-{{K8S_CLUSTER_NAME}}` online | [ ] |
| 4 | Agent DaemonSet running on all nodes | `kubectl get ds -n {{K8S_NAMESPACE}}` | DESIRED = CURRENT = READY | [ ] |
| 5 | kube-state-metrics running | `kubectl get deployment -n {{K8S_NAMESPACE}}` | kube-state-metrics deployment ready (if enabled) | [ ] |
| 6 | Cluster state items collecting | Monitoring > Latest data > K8s Cluster State host | Deployment, pod, PV items have values | [ ] |
| 7 | Node discovery running | Monitoring > Latest data > K8s Nodes host | Node items show CPU, memory, conditions | [ ] |
| 8 | Pod monitoring active | Monitoring > Latest data > filter by pod items | Pod phase, restart count items populated | [ ] |
| 9 | Namespace filtering applied | Monitoring > Latest data > verify namespace scope | Only configured namespaces appear | [ ] |
| 10 | Custom triggers created | Data collection > Hosts > K8s Cluster State > Triggers | OOMKilled, CrashLoopBackOff, PV triggers exist | [ ] |
| 11 | No unsupported items | Monitoring > Hosts > filter "Unsupported items" | Zero unsupported items on K8s hosts | [ ] |

---

## Troubleshooting

### Helm Install Fails with RBAC Errors

**Symptom:** `helm install` returns `Error: INSTALLATION FAILED: ... forbidden: User "system:serviceaccount:..." cannot create resource ...`

**Resolution:**
1. Verify your `kubectl` context has cluster-admin privileges:
   ```bash
   kubectl auth can-i create clusterrole --all-namespaces
   # Expected: yes
   ```
2. If using a restricted context, create the RBAC resources manually first:
   ```bash
   kubectl create clusterrolebinding zabbix-admin \
     --clusterrole=cluster-admin \
     --serviceaccount={{K8S_NAMESPACE}}:zabbix-service-account
   ```
3. If your cluster uses Pod Security Admission, ensure the namespace allows the required security contexts:
   ```bash
   kubectl label namespace {{K8S_NAMESPACE}} \
     pod-security.kubernetes.io/enforce=privileged \
     --overwrite
   ```

### Proxy Cannot Reach Zabbix Server

**Symptom:** Proxy pod logs show `cannot send proxy data to server` or connection timeout errors.

**Resolution:**
1. Verify outbound TCP/10051 is allowed from the cluster:
   ```bash
   # Run a test pod
   kubectl run nettest --rm -it --image=busybox -n {{K8S_NAMESPACE}} -- \
     nc -zv {{ZBX_SERVER_A_IP}} 10051
   ```
2. If the cluster uses a network policy controller (Calico, Cilium), create an egress policy:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-zabbix-proxy-egress
     namespace: {{K8S_NAMESPACE}}
   spec:
     podSelector:
       matchLabels:
         app: zabbix-zabbix-helm-chrt-proxy
     policyTypes:
       - Egress
     egress:
       - to:
           - ipBlock:
               cidr: {{ZBX_SERVER_A_IP}}/32
           - ipBlock:
               cidr: {{ZBX_SERVER_B_IP}}/32
         ports:
           - protocol: TCP
             port: 10051
   ```
3. If the cluster is behind a NAT or proxy, ensure the Zabbix Server's firewall allows connections from the cluster's egress IP.

### API Token Invalid or Expired

**Symptom:** Cluster state items show "HTTP error 401" or "Unauthorized" in the item status.

**Resolution:**
1. Verify the token is still valid:
   ```bash
   curl -k -H "Authorization: Bearer <TOKEN>" \
     {{K8S_API_URL}}/api/v1/namespaces
   ```
   If the response is `401 Unauthorized`, the token has expired or is invalid.

2. Create a new token:
   ```bash
   kubectl create token zabbix-service-account \
     -n {{K8S_NAMESPACE}} \
     --duration=8760h
   ```

3. Update the `{$KUBE.API.TOKEN}` macro on all Kubernetes hosts in Zabbix:
   - Data collection > Hosts > K8s Cluster State > Macros
   - Data collection > Hosts > K8s Nodes > Macros

4. If using Kubernetes 1.23 or earlier with auto-mounted secrets, verify the secret still exists:
   ```bash
   kubectl get secret -n {{K8S_NAMESPACE}} | grep zabbix-service-account
   ```

### No Discovery Data (kube-state-metrics Not Running)

**Symptom:** Cluster state items show "No data" or discovery rules return empty results.

**Resolution:**
1. Check if kube-state-metrics is running:
   ```bash
   kubectl get pods -n {{K8S_NAMESPACE}} -l app.kubernetes.io/name=kube-state-metrics
   ```
2. If the pod is not running and you set `kubeStateMetrics.enabled: false`, verify the existing kube-state-metrics deployment is healthy:
   ```bash
   kubectl get deployment -A | grep kube-state-metrics
   kubectl get pods -A -l app.kubernetes.io/name=kube-state-metrics
   ```
3. If kube-state-metrics exists in a different namespace, verify the Zabbix proxy can reach it:
   ```bash
   kubectl exec -n {{K8S_NAMESPACE}} -it <proxy-pod> -- \
     curl http://kube-state-metrics.<ksm-namespace>.svc:8080/metrics | head -20
   ```
4. If kube-state-metrics is not deployed anywhere, re-install the Helm chart with `kubeStateMetrics.enabled: true`:
   ```bash
   helm upgrade zabbix zabbix-chart-7.2/zabbix-helm-chrt \
     --set kubeStateMetrics.enabled=true \
     -f k8s-zabbix-values.yaml \
     -n {{K8S_NAMESPACE}}
   ```

### Agent DaemonSet Not Scheduling on All Nodes

**Symptom:** `kubectl get ds` shows DESIRED count does not match CURRENT or READY count.

**Resolution:**
1. Check for taints on nodes that prevent scheduling:
   ```bash
   kubectl describe nodes | grep -A 3 Taints
   ```
2. Verify the agent DaemonSet has the correct tolerations in the Helm values:
   ```yaml
   tolerations:
     - key: node-role.kubernetes.io/control-plane
       operator: Exists
       effect: NoSchedule
   ```
3. Check for pending pods and their events:
   ```bash
   kubectl get pods -n {{K8S_NAMESPACE}} -o wide | grep -v Running
   kubectl describe pod <pending-pod-name> -n {{K8S_NAMESPACE}}
   ```
4. If nodes have custom taints, add corresponding tolerations to the Helm values and upgrade the release.
