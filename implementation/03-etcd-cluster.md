# Phase 3 — etcd Cluster

## Objective

Deploy a TLS-secured, 3-node etcd cluster that provides distributed consensus for Patroni. The etcd cluster is the foundation for automatic PostgreSQL failover — if etcd loses quorum, Patroni cannot promote a new primary.

## Prerequisites

- [ ] Phase 2 complete — all packages installed, database schema imported
- [ ] 3 etcd VMs provisioned and accessible: `etcd-1` (`{{ETCD_1_IP}}`), `etcd-2` (`{{ETCD_2_IP}}`), `etcd-3` (`{{ETCD_3_IP}}`)
- [ ] Firewall ports open: 2379/tcp (client), 2380/tcp (peer) on all 3 etcd VMs
- [ ] Cross-site connectivity verified between all etcd nodes
- [ ] `{{ETCD_ROOT_PASSWORD}}` and `{{ETCD_PATRONI_PASSWORD}}` generated and stored in vault

---

## Step 1 — Install etcd

Run on **all 3 etcd nodes** (`etcd-1`, `etcd-2`, `etcd-3`).

### Option A: PGDG RPM (if PGDG repo is already configured)

```bash
dnf install -y 'dnf-command(config-manager)'
dnf config-manager --enable pgdg-rhel9-extras
dnf install -y etcd

# Verify
etcd --version
etcdctl version
```

### Option B: Binary Install (recommended for specific version control)

```bash
ETCD_VER=v3.5.21

curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
  -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

tar xzf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/

sudo install -m 0755 /tmp/etcd-${ETCD_VER}-linux-amd64/etcd /usr/local/bin/etcd
sudo install -m 0755 /tmp/etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/etcdctl

# Verify
etcd --version
etcdctl version

# Clean up
rm -rf /tmp/etcd-${ETCD_VER}-linux-amd64*
```

---

## Step 2 — Create etcd User and Directories

Run on **all 3 etcd nodes**.

```bash
# Create dedicated etcd system user
useradd -r -s /sbin/nologin etcd

# Create data and config directories
mkdir -p /var/lib/etcd
mkdir -p /etc/etcd/pki

# Set ownership and permissions
chown etcd:etcd /var/lib/etcd
chmod 700 /var/lib/etcd

chown etcd:etcd /etc/etcd
chown etcd:etcd /etc/etcd/pki
chmod 750 /etc/etcd
chmod 750 /etc/etcd/pki
```

---

## Step 3 — Generate TLS Certificates

Run certificate generation on a single workstation (or on `etcd-1`), then distribute to all nodes.

### 3a. Create a Working Directory

```bash
mkdir -p ~/etcd-pki && cd ~/etcd-pki
```

### 3b. Generate the Certificate Authority (CA)

```bash
# Generate CA private key (4096-bit RSA)
openssl genrsa -out ca.key 4096

# Generate self-signed CA certificate (10-year validity)
openssl req -x509 -new -nodes \
  -key ca.key \
  -sha256 \
  -days 3650 \
  -subj "/CN=etcd-ca/O={{TLS_ORG_NAME}}" \
  -out ca.crt
```

### 3c. Generate Server and Peer Certificates for etcd-1

```bash
for ROLE in server peer; do
  # Generate private key
  openssl genrsa -out etcd-1-${ROLE}.key 2048

  # Generate certificate signing request with SAN
  openssl req -new \
    -key etcd-1-${ROLE}.key \
    -subj "/CN=etcd-1" \
    -addext "subjectAltName=IP:{{ETCD_1_IP}},IP:127.0.0.1,DNS:etcd-1,DNS:etcd-1.{{DOMAIN}}" \
    -out etcd-1-${ROLE}.csr

  # Sign with CA
  openssl x509 -req \
    -in etcd-1-${ROLE}.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out etcd-1-${ROLE}.crt \
    -days 1825 \
    -sha256 \
    -extfile <(printf "subjectAltName=IP:{{ETCD_1_IP}},IP:127.0.0.1,DNS:etcd-1,DNS:etcd-1.{{DOMAIN}}")
done
```

### 3d. Generate Server and Peer Certificates for etcd-2

```bash
for ROLE in server peer; do
  openssl genrsa -out etcd-2-${ROLE}.key 2048

  openssl req -new \
    -key etcd-2-${ROLE}.key \
    -subj "/CN=etcd-2" \
    -addext "subjectAltName=IP:{{ETCD_2_IP}},IP:127.0.0.1,DNS:etcd-2,DNS:etcd-2.{{DOMAIN}}" \
    -out etcd-2-${ROLE}.csr

  openssl x509 -req \
    -in etcd-2-${ROLE}.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out etcd-2-${ROLE}.crt \
    -days 1825 \
    -sha256 \
    -extfile <(printf "subjectAltName=IP:{{ETCD_2_IP}},IP:127.0.0.1,DNS:etcd-2,DNS:etcd-2.{{DOMAIN}}")
done
```

### 3e. Generate Server and Peer Certificates for etcd-3

```bash
for ROLE in server peer; do
  openssl genrsa -out etcd-3-${ROLE}.key 2048

  openssl req -new \
    -key etcd-3-${ROLE}.key \
    -subj "/CN=etcd-3" \
    -addext "subjectAltName=IP:{{ETCD_3_IP}},IP:127.0.0.1,DNS:etcd-3,DNS:etcd-3.{{DOMAIN}}" \
    -out etcd-3-${ROLE}.csr

  openssl x509 -req \
    -in etcd-3-${ROLE}.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out etcd-3-${ROLE}.crt \
    -days 1825 \
    -sha256 \
    -extfile <(printf "subjectAltName=IP:{{ETCD_3_IP}},IP:127.0.0.1,DNS:etcd-3,DNS:etcd-3.{{DOMAIN}}")
done
```

### 3f. Deploy Certificates to Each Node

Copy the CA certificate and the node-specific server/peer certificates to each etcd node.

**On etcd-1:**

```bash
# From the workstation where certs were generated:
scp ca.crt etcd-1-server.crt etcd-1-server.key etcd-1-peer.crt etcd-1-peer.key root@{{ETCD_1_IP}}:/etc/etcd/pki/

# On etcd-1, rename to standard names:
ssh root@{{ETCD_1_IP}} << 'REMOTE'
cd /etc/etcd/pki
mv etcd-1-server.crt server.crt
mv etcd-1-server.key server.key
mv etcd-1-peer.crt peer.crt
mv etcd-1-peer.key peer.key
chown etcd:etcd *.crt *.key
chmod 644 ca.crt server.crt peer.crt
chmod 600 server.key peer.key
REMOTE
```

**On etcd-2:**

```bash
scp ca.crt etcd-2-server.crt etcd-2-server.key etcd-2-peer.crt etcd-2-peer.key root@{{ETCD_2_IP}}:/etc/etcd/pki/

ssh root@{{ETCD_2_IP}} << 'REMOTE'
cd /etc/etcd/pki
mv etcd-2-server.crt server.crt
mv etcd-2-server.key server.key
mv etcd-2-peer.crt peer.crt
mv etcd-2-peer.key peer.key
chown etcd:etcd *.crt *.key
chmod 644 ca.crt server.crt peer.crt
chmod 600 server.key peer.key
REMOTE
```

**On etcd-3:**

```bash
scp ca.crt etcd-3-server.crt etcd-3-server.key etcd-3-peer.crt etcd-3-peer.key root@{{ETCD_3_IP}}:/etc/etcd/pki/

ssh root@{{ETCD_3_IP}} << 'REMOTE'
cd /etc/etcd/pki
mv etcd-3-server.crt server.crt
mv etcd-3-server.key server.key
mv etcd-3-peer.crt peer.crt
mv etcd-3-peer.key peer.key
chown etcd:etcd *.crt *.key
chmod 644 ca.crt server.crt peer.crt
chmod 600 server.key peer.key
REMOTE
```

### 3g. Verify Certificate SANs

On each node, verify the server certificate contains the correct SAN entries:

```bash
openssl x509 -in /etc/etcd/pki/server.crt -text -noout | grep -A1 "Subject Alternative Name"
```

> **IMPORTANT -- Secure the CA private key:** After all node certificates have been generated and deployed, move `ca.key` to an offline secure location (e.g., hardware security module, encrypted USB in a safe, or secrets vault). The CA key can sign new certificates trusted by the entire etcd cluster. Do **not** leave it on any networked system.

---

## Step 4 — Configure etcd-1

Create `/etc/etcd/etcd.conf.yml` on `etcd-1`:

```yaml
name: 'etcd-1'
data-dir: /var/lib/etcd

listen-peer-urls: 'https://{{ETCD_1_IP}}:2380'
initial-advertise-peer-urls: 'https://{{ETCD_1_IP}}:2380'
listen-client-urls: 'https://{{ETCD_1_IP}}:2379,https://127.0.0.1:2379'
advertise-client-urls: 'https://{{ETCD_1_IP}}:2379'

initial-cluster-token: 'patroni-etcd-cluster'
initial-cluster-state: 'new'
initial-cluster: 'etcd-1=https://{{ETCD_1_IP}}:2380,etcd-2=https://{{ETCD_2_IP}}:2380,etcd-3=https://{{ETCD_3_IP}}:2380'

# Tuning
heartbeat-interval: 250
election-timeout: 2500
snapshot-count: 10000
quota-backend-bytes: 2147483648

auto-compaction-mode: periodic
auto-compaction-retention: '1h'

# TLS: Client
client-transport-security:
  cert-file: /etc/etcd/pki/server.crt
  key-file: /etc/etcd/pki/server.key
  trusted-ca-file: /etc/etcd/pki/ca.crt
  client-cert-auth: true

# TLS: Peer
peer-transport-security:
  cert-file: /etc/etcd/pki/peer.crt
  key-file: /etc/etcd/pki/peer.key
  trusted-ca-file: /etc/etcd/pki/ca.crt
  client-cert-auth: true
```

> **Security:** `client-cert-auth: true` requires all clients (including Patroni) to present a valid TLS client certificate. This provides defense-in-depth beyond password authentication. Patroni's etcd3 connection must include `cert` and `key` parameters pointing to a client certificate signed by the same CA.

Set ownership:

```bash
chown etcd:etcd /etc/etcd/etcd.conf.yml
chmod 640 /etc/etcd/etcd.conf.yml
```

---

## Step 5 — Configure etcd-2

Create `/etc/etcd/etcd.conf.yml` on `etcd-2`:

```yaml
name: 'etcd-2'
data-dir: /var/lib/etcd

listen-peer-urls: 'https://{{ETCD_2_IP}}:2380'
initial-advertise-peer-urls: 'https://{{ETCD_2_IP}}:2380'
listen-client-urls: 'https://{{ETCD_2_IP}}:2379,https://127.0.0.1:2379'
advertise-client-urls: 'https://{{ETCD_2_IP}}:2379'

initial-cluster-token: 'patroni-etcd-cluster'
initial-cluster-state: 'new'
initial-cluster: 'etcd-1=https://{{ETCD_1_IP}}:2380,etcd-2=https://{{ETCD_2_IP}}:2380,etcd-3=https://{{ETCD_3_IP}}:2380'

# Tuning
heartbeat-interval: 250
election-timeout: 2500
snapshot-count: 10000
quota-backend-bytes: 2147483648

auto-compaction-mode: periodic
auto-compaction-retention: '1h'

# TLS: Client
client-transport-security:
  cert-file: /etc/etcd/pki/server.crt
  key-file: /etc/etcd/pki/server.key
  trusted-ca-file: /etc/etcd/pki/ca.crt
  client-cert-auth: true

# TLS: Peer
peer-transport-security:
  cert-file: /etc/etcd/pki/peer.crt
  key-file: /etc/etcd/pki/peer.key
  trusted-ca-file: /etc/etcd/pki/ca.crt
  client-cert-auth: true
```

Set ownership:

```bash
chown etcd:etcd /etc/etcd/etcd.conf.yml
chmod 640 /etc/etcd/etcd.conf.yml
```

---

## Step 6 — Configure etcd-3

Create `/etc/etcd/etcd.conf.yml` on `etcd-3` (arbitrator):

```yaml
name: 'etcd-3'
data-dir: /var/lib/etcd

listen-peer-urls: 'https://{{ETCD_3_IP}}:2380'
initial-advertise-peer-urls: 'https://{{ETCD_3_IP}}:2380'
listen-client-urls: 'https://{{ETCD_3_IP}}:2379,https://127.0.0.1:2379'
advertise-client-urls: 'https://{{ETCD_3_IP}}:2379'

initial-cluster-token: 'patroni-etcd-cluster'
initial-cluster-state: 'new'
initial-cluster: 'etcd-1=https://{{ETCD_1_IP}}:2380,etcd-2=https://{{ETCD_2_IP}}:2380,etcd-3=https://{{ETCD_3_IP}}:2380'

# Tuning
heartbeat-interval: 250
election-timeout: 2500
snapshot-count: 10000
quota-backend-bytes: 2147483648

auto-compaction-mode: periodic
auto-compaction-retention: '1h'

# TLS: Client
client-transport-security:
  cert-file: /etc/etcd/pki/server.crt
  key-file: /etc/etcd/pki/server.key
  trusted-ca-file: /etc/etcd/pki/ca.crt
  client-cert-auth: true

# TLS: Peer
peer-transport-security:
  cert-file: /etc/etcd/pki/peer.crt
  key-file: /etc/etcd/pki/peer.key
  trusted-ca-file: /etc/etcd/pki/ca.crt
  client-cert-auth: true
```

Set ownership:

```bash
chown etcd:etcd /etc/etcd/etcd.conf.yml
chmod 640 /etc/etcd/etcd.conf.yml
```

---

## Step 7 — Tuning Notes

The following tuning parameters are set in the configuration files above. Adjust as needed based on your inter-site latency.

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `heartbeat-interval` | 250 ms | Heartbeat frequency between leader and followers. Should be ~1.5x the inter-site RTT. |
| `election-timeout` | 2500 ms | Time a follower waits before calling a new election. Must be 5-10x the heartbeat interval. |
| `snapshot-count` | 10000 | Number of committed transactions before triggering a snapshot. Balances memory vs. I/O. |
| `quota-backend-bytes` | 2147483648 (2 GB) | Maximum database size. Patroni uses minimal etcd storage; 2 GB is generous. |
| `auto-compaction-mode` | periodic | Automatically compact old revisions to free space. |
| `auto-compaction-retention` | 1h | Keep 1 hour of revision history before compacting. |

**Adjusting for High-Latency Links:**

If your inter-site RTT is > 5 ms, consider increasing the tuning values:

```
heartbeat-interval = RTT * 1.5   (e.g., 10ms RTT -> 15ms, round up to 20ms minimum)
election-timeout = heartbeat-interval * 10
```

> **Warning:** Setting `election-timeout` too low relative to RTT causes unnecessary leader elections. Setting it too high delays failover detection. The defaults (250ms / 2500ms) work well for RTT up to ~10ms.

---

## Step 8 — Create systemd Unit File

Create `/etc/systemd/system/etcd.service` on **all 3 nodes**:

```ini
[Unit]
Description=etcd key-value store
Documentation=https://etcd.io/docs/
After=network-online.target
Wants=network-online.target

[Service]
User=etcd
Group=etcd
Type=notify
Environment="ETCD_CONFIG_FILE=/etc/etcd/etcd.conf.yml"
ExecStart=/usr/local/bin/etcd
Restart=always
RestartSec=10s
LimitNOFILE=40000
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

> **Note:** If you installed etcd via RPM (Option A in Step 1), change the `ExecStart` path to `/usr/bin/etcd`.

Set permissions:

```bash
chmod 644 /etc/systemd/system/etcd.service
systemctl daemon-reload
```

---

## Step 9 — Bootstrap the Cluster

> **CRITICAL:** All 3 etcd nodes must start within **30 seconds** of each other for initial cluster formation. If the bootstrap window is missed, the nodes will fail to connect and you must clean up and retry (see Troubleshooting).

### 9a. Prepare All Nodes

On **all 3 nodes**, reload systemd and enable the service:

```bash
systemctl daemon-reload
systemctl enable etcd
```

### 9b. Start All 3 Nodes

Start the nodes as quickly as possible. Use multiple SSH sessions or a parallel execution tool.

```bash
# Terminal 1 (etcd-1):
systemctl start etcd

# Terminal 2 (etcd-2):
systemctl start etcd

# Terminal 3 (etcd-3):
systemctl start etcd
```

### 9c. Verify Bootstrap

Wait 10-15 seconds, then check the cluster status from any node:

```bash
# Set environment for etcdctl
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS="https://{{ETCD_1_IP}}:2379,https://{{ETCD_2_IP}}:2379,https://{{ETCD_3_IP}}:2379"
export ETCDCTL_CACERT="/etc/etcd/pki/ca.crt"

# Check member list
etcdctl member list -w table

# Check endpoint health
etcdctl endpoint health -w table

# Check cluster status (shows leader, DB size, Raft term)
etcdctl endpoint status -w table
```

**Expected output from `member list`:** 3 members, all with `started` status.

**Expected output from `endpoint health`:** All 3 endpoints show `healthy: true`.

**Expected output from `endpoint status`:** One node shows `IS LEADER: true`, other two show `false`.

---

## Step 10 — Enable RBAC for Patroni

Once the cluster is healthy, enable authentication so that only authorized clients (Patroni) can read and write the service discovery prefix.

### 10a. Add the root User

```bash
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS="https://{{ETCD_1_IP}}:2379"
export ETCDCTL_CACERT="/etc/etcd/pki/ca.crt"

# Add root user (enter {{ETCD_ROOT_PASSWORD}} when prompted)
etcdctl user add root
```

### 10b. Add the patroni User

```bash
# Add patroni user (enter {{ETCD_PATRONI_PASSWORD}} when prompted)
etcdctl user add patroni
```

### 10c. Create the patroni-role with Scoped Permissions

```bash
# Create the role
etcdctl role add patroni-role

# Grant read-write access to the /service/ prefix (Patroni's namespace)
etcdctl role grant-permission patroni-role readwrite --prefix=true /service/

# Assign the role to the patroni user
etcdctl user grant-role patroni patroni-role

# Assign the root role to the root user
etcdctl user grant-role root root
```

### 10d. Enable Authentication

```bash
etcdctl auth enable
```

> **WARNING:** After auth is enabled, all subsequent etcdctl commands require `--user root:<password>` or `--user patroni:<password>`. Save the root password securely.

### 10e. Verify Authentication

```bash
# This should fail (no credentials):
etcdctl --endpoints=https://{{ETCD_1_IP}}:2379 --cacert=/etc/etcd/pki/ca.crt \
  get / --prefix --keys-only

# This should succeed:
etcdctl --endpoints=https://{{ETCD_1_IP}}:2379 --cacert=/etc/etcd/pki/ca.crt \
  --user patroni:{{ETCD_PATRONI_PASSWORD}} \
  put /service/test "hello"

etcdctl --endpoints=https://{{ETCD_1_IP}}:2379 --cacert=/etc/etcd/pki/ca.crt \
  --user patroni:{{ETCD_PATRONI_PASSWORD}} \
  get /service/test

# Clean up test key
etcdctl --endpoints=https://{{ETCD_1_IP}}:2379 --cacert=/etc/etcd/pki/ca.crt \
  --user patroni:{{ETCD_PATRONI_PASSWORD}} \
  del /service/test
```

> **Note:** Avoid passing passwords on the command line in production. Use `export ETCDCTL_USER=patroni:$(cat /path/to/patroni-password-file)` or `ETCDCTL_USER` environment variable instead. The inline form is shown here for clarity.

---

## Verification Checkpoint

Complete all checks before proceeding to Phase 4.

| # | Check | Command | Expected Result | Pass |
|---|-------|---------|-----------------|------|
| 1 | etcd installed on all 3 nodes | `etcd --version` (on each node) | Version v3.5.x returned | [ ] |
| 2 | TLS certs deployed | `ls -la /etc/etcd/pki/` (on each node) | `ca.crt`, `server.crt`, `server.key`, `peer.crt`, `peer.key` present | [ ] |
| 3 | Cert SAN includes correct IP | `openssl x509 -in /etc/etcd/pki/server.crt -text -noout \| grep -A1 "Subject Alternative"` | Node's IP address listed | [ ] |
| 4 | Member list shows 3 members | `etcdctl member list -w table` | 3 rows, all `started` | [ ] |
| 5 | All endpoints healthy | `etcdctl endpoint health -w table` | All 3 `healthy: true` | [ ] |
| 6 | Leader elected | `etcdctl endpoint status -w table` | Exactly 1 node shows `IS LEADER: true` | [ ] |
| 7 | Auth enabled | `etcdctl --user root:{{ETCD_ROOT_PASSWORD}} auth status` | `Authentication Status: true` | [ ] |
| 8 | Patroni user can write `/service/` | `etcdctl --user patroni:{{ETCD_PATRONI_PASSWORD}} put /service/test "ok"` | `OK` | [ ] |
| 9 | Patroni user can read `/service/` | `etcdctl --user patroni:{{ETCD_PATRONI_PASSWORD}} get /service/test` | `ok` returned | [ ] |
| 10 | Unauthenticated access denied | `etcdctl get / --prefix` (no `--user`) | `Error: etcdserver: user name is empty` | [ ] |

> **Note:** All etcdctl commands in checks 4-10 require the ETCDCTL_ENDPOINTS, ETCDCTL_CACERT, and (after auth is enabled) `--user` flag. After auth is enabled, add `--user root:{{ETCD_ROOT_PASSWORD}}` to commands 4-6.

---

## Troubleshooting

### Bootstrap Timeout (Nodes Not Starting Within 30 Seconds)

**Symptom:** `systemctl start etcd` hangs or `journalctl -u etcd` shows `waiting for peers` or `connection refused` errors.

**Resolution:**
1. Stop etcd on all 3 nodes:
   ```bash
   systemctl stop etcd
   ```
2. Remove the data directory on all 3 nodes:
   ```bash
   rm -rf /var/lib/etcd/*
   ```
3. Verify the `initial-cluster` string in each node's config matches all 3 nodes with correct IPs.
4. Verify the `initial-cluster-token` is identical on all 3 nodes.
5. Verify the `initial-cluster-state` is `new` on all 3 nodes.
6. Verify connectivity between all nodes on port 2380:
   ```bash
   nc -zv {{ETCD_2_IP}} 2380
   nc -zv {{ETCD_3_IP}} 2380
   ```
7. Restart all 3 nodes simultaneously within 30 seconds.

### TLS Handshake Failures

**Symptom:** `journalctl -u etcd` shows `tls: bad certificate`, `x509: certificate signed by unknown authority`, or `remote error: tls: bad certificate`.

**Resolution:**
1. Verify the CA certificate is the same on all nodes:
   ```bash
   md5sum /etc/etcd/pki/ca.crt
   # Should be identical on all 3 nodes
   ```
2. Verify the server certificate was signed by the CA:
   ```bash
   openssl verify -CAfile /etc/etcd/pki/ca.crt /etc/etcd/pki/server.crt
   # Expected: server.crt: OK
   ```
3. Verify the SAN includes the correct IP:
   ```bash
   openssl x509 -in /etc/etcd/pki/server.crt -text -noout | grep -A2 "Subject Alternative"
   ```
4. Verify file ownership:
   ```bash
   ls -la /etc/etcd/pki/
   # All files should be owned by etcd:etcd
   ```
5. Verify the key matches the certificate:
   ```bash
   openssl x509 -noout -modulus -in /etc/etcd/pki/server.crt | md5sum
   openssl rsa -noout -modulus -in /etc/etcd/pki/server.key | md5sum
   # Both md5sums should match
   ```

### Quorum Issues

**Symptom:** `etcdctl endpoint health` shows 1 or 2 nodes as unhealthy, or `member list` shows fewer than 3 members.

**Resolution:**
1. Check which nodes are running:
   ```bash
   systemctl status etcd   # On each node
   ```
2. Check etcd logs for errors:
   ```bash
   journalctl -u etcd --no-pager -n 50
   ```
3. If a single node is unhealthy but the other 2 are fine, restart the unhealthy node:
   ```bash
   systemctl restart etcd
   ```
4. If 2 nodes are down, etcd has lost quorum. Recovery requires starting at least 2 of the 3 nodes. etcd will not accept writes until quorum is restored.
5. If the data directory is corrupted on one node:
   ```bash
   systemctl stop etcd
   rm -rf /var/lib/etcd/*
   # Remove this member and re-add it
   etcdctl --user root:{{ETCD_ROOT_PASSWORD}} member remove <member-id>
   etcdctl --user root:{{ETCD_ROOT_PASSWORD}} member add etcd-X --peer-urls=https://<IP>:2380
   ```
   Then update the node's config to use `initial-cluster-state: 'existing'` and restart.

### Auth Lockout

**Symptom:** You cannot authenticate to etcd after enabling auth (forgot password, mistyped password).

**Resolution:**
1. If you still have access as root:
   ```bash
   etcdctl --user root:{{ETCD_ROOT_PASSWORD}} user passwd patroni
   ```
2. If root access is also lost, disable auth by modifying the etcd data directory directly (last resort):
   ```bash
   # Stop etcd on all nodes
   systemctl stop etcd

   # On the leader node, use etcd-dump-db to export and reimport
   # OR simply re-bootstrap the cluster:
   rm -rf /var/lib/etcd/*  # On ALL nodes
   # Change initial-cluster-state back to 'new' in all configs
   # Restart all nodes and re-create users
   ```

   > **Warning:** Re-bootstrapping etcd erases all Patroni state. If Patroni is already running, this will cause a PostgreSQL failover. Only use this as a last resort before Phase 4.

### etcd Performance Issues

**Symptom:** etcd logs show `apply request took too long` or `failed to send out heartbeat on time`.

**Resolution:**
1. Verify the data directory is on SSD storage:
   ```bash
   # Check I/O latency
   dd if=/dev/zero of=/var/lib/etcd/testfile bs=512 count=1000 oflag=dsync 2>&1 | tail -1
   # Should show < 10ms latency per operation
   rm /var/lib/etcd/testfile
   ```
2. Check disk space:
   ```bash
   df -h /var/lib/etcd
   ```
3. If the database is approaching the quota limit:
   ```bash
   etcdctl --user root:{{ETCD_ROOT_PASSWORD}} endpoint status -w table
   # Check DB SIZE column
   ```
4. Trigger manual compaction if needed:
   ```bash
   # Get current revision
   REV=$(etcdctl --user root:{{ETCD_ROOT_PASSWORD}} endpoint status -w json | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['Status']['header']['revision'])")

   # Compact
   etcdctl --user root:{{ETCD_ROOT_PASSWORD}} compact $REV

   # Defragment
   etcdctl --user root:{{ETCD_ROOT_PASSWORD}} defrag
   ```
