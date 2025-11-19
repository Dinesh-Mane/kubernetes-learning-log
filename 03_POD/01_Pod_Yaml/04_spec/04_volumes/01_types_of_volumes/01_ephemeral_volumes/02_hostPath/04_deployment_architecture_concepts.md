# Part 4: Deployment Architecture Concepts

## 1. hostPath + Static Pod Behavior

### What Are Static Pods?

**Definition**: Pods managed **directly by kubelet**, not by the API server. Kubelet watches a directory for Pod manifests and ensures they're running.

**Key differences from normal Pods**:
```
Normal Pods:
API Server → etcd → Scheduler → Kubelet → Container Runtime

Static Pods:
/etc/kubernetes/manifests/*.yaml → Kubelet → Container Runtime
```

### Static Pod Configuration

```bash
# Kubelet config
cat /var/lib/kubelet/config.yaml
# ...
staticPodPath: /etc/kubernetes/manifests
# ...

# Static Pod manifests location
ls -la /etc/kubernetes/manifests/
# -rw------- 1 root root 3821 Nov 19 10:00 etcd.yaml
# -rw------- 1 root root 3156 Nov 19 10:00 kube-apiserver.yaml
# -rw------- 1 root root 2719 Nov 19 10:00 kube-controller-manager.yaml
# -rw------- 1 root root 1435 Nov 19 10:00 kube-scheduler.yaml
```

### Static Pod with hostPath: The Bootstrap Pattern

**Example: etcd Static Pod** (the most critical component)

```yaml
# /etc/kubernetes/manifests/etcd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: etcd
  namespace: kube-system
  labels:
    component: etcd
    tier: control-plane
spec:
  hostNetwork: true  # Use host networking
  priorityClassName: system-node-critical
  
  containers:
  - name: etcd
    image: registry.k8s.io/etcd:3.5.9-0
    command:
    - etcd
    - --advertise-client-urls=https://192.168.1.10:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd  # ← hostPath mount point
    - --initial-advertise-peer-urls=https://192.168.1.10:2380
    - --initial-cluster=master-1=https://192.168.1.10:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://192.168.1.10:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://192.168.1.10:2380
    - --name=master-1
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    
    livenessProbe:
      httpGet:
        path: /health
        port: 2381
        scheme: HTTP
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    
    resources:
      requests:
        cpu: 100m
        memory: 100Mi
    
    volumeMounts:
    # Critical: etcd data MUST persist on host
    - name: etcd-data
      mountPath: /var/lib/etcd
    
    # TLS certificates from host
    - name: etcd-certs
      mountPath: /etc/kubernetes/pki/etcd
      readOnly: true
  
  volumes:
  # hostPath for persistent etcd data
  - name: etcd-data
    hostPath:
      path: /var/lib/etcd
      type: DirectoryOrCreate
  
  # hostPath for certificates
  - name: etcd-certs
    hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
```

### Why Static Pods NEED hostPath

**The Bootstrap Paradox**:
```
Problem: How do you run Kubernetes control plane?
  ↓
Control plane needs storage (etcd data, certificates)
  ↓
But PersistentVolumes require API server
  ↓
But API server needs etcd
  ↓
But etcd needs storage
  ↓
CIRCULAR DEPENDENCY!
```

**Solution: hostPath breaks the cycle**:
```
1. kubelet starts (no API server needed)
2. kubelet reads /etc/kubernetes/manifests/etcd.yaml
3. etcd Pod starts with hostPath to /var/lib/etcd
4. etcd runs, stores data on host
5. API server can now start (connects to etcd)
6. Now PersistentVolumes can be created
```

### Static Pod Lifecycle with hostPath

**What happens when Static Pod is modified**:

```bash
# Original manifest
cat /etc/kubernetes/manifests/etcd.yaml | grep data-dir
# --data-dir=/var/lib/etcd

# Edit manifest to change data directory
sudo sed -i 's|/var/lib/etcd|/var/lib/etcd-new|' /etc/kubernetes/manifests/etcd.yaml
sudo sed -i 's|path: /var/lib/etcd|path: /var/lib/etcd-new|' /etc/kubernetes/manifests/etcd.yaml

# Kubelet detects change within ~20 seconds
# Stops old container
# Starts new container with new hostPath

# Check kubelet logs
journalctl -u kubelet -f
# Nov 19 10:05:32 master-1 kubelet[1234]: Static pod manifest changed, restarting pod etcd-master-1
# Nov 19 10:05:35 master-1 kubelet[1234]: Killing container etcd (old version)
# Nov 19 10:05:40 master-1 kubelet[1234]: Starting container etcd (new version)
```

**CRITICAL**: Data at `/var/lib/etcd` is NOT automatically migrated to `/var/lib/etcd-new`. You must manually copy data first or cluster breaks!

### Static Pod Failure Scenarios

#### Scenario 1: hostPath Directory Deleted

```bash
# Someone accidentally deletes etcd data
sudo rm -rf /var/lib/etcd

# Kubelet attempts to restart Pod
# etcd starts with EMPTY data directory
# Cluster thinks it's a new member
# ERROR: existing cluster members don't recognize it
# Control plane goes down!
```

**Recovery**:
```bash
# Stop kubelet to prevent Pod restart
sudo systemctl stop kubelet

# Restore from backup
sudo etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd \
  --name=master-1 \
  --initial-cluster=master-1=https://192.168.1.10:2380 \
  --initial-advertise-peer-urls=https://192.168.1.10:2380

# Fix permissions
sudo chown -R etcd:etcd /var/lib/etcd

# Restart kubelet
sudo systemctl start kubelet

# Verify etcd Pod is running
sudo crictl ps | grep etcd
```

#### Scenario 2: Permission Issues

```bash
# hostPath directory created with wrong ownership
sudo chown root:root /var/lib/etcd
sudo chmod 700 /var/lib/etcd

# etcd container runs as UID 999 (typical)
# Cannot write to directory
# Pod crashes immediately

# Check logs
sudo crictl logs $(sudo crictl ps -a | grep etcd | awk '{print $1}')
# 2024-11-19 10:00:00.123456 E | etcdserver: cannot access data directory: permission denied

# Fix
sudo chown -R 999:999 /var/lib/etcd  # etcd UID
sudo chmod 700 /var/lib/etcd
```

### Static Pod + hostPath Best Practices---

### Static Pods + hostPath: Production Operations Guide

## Critical Understanding

Static Pods are **non-negotiable** for Kubernetes control plane. They MUST use hostPath because:
- No API server available during bootstrap
- No PersistentVolume system available
- Must survive API server failures
- Data must persist across Pod restarts

---

## Standard Static Pod Architecture

### Control Plane Components (kubeadm-based)

```
Master Node Filesystem:
├── /etc/kubernetes/
│   ├── manifests/               ← Static Pod definitions
│   │   ├── etcd.yaml            (hostPath: /var/lib/etcd)
│   │   ├── kube-apiserver.yaml  (hostPath: /etc/kubernetes/)
│   │   ├── kube-controller-manager.yaml
│   │   └── kube-scheduler.yaml
│   ├── pki/                     ← Certificates (mounted via hostPath)
│   │   ├── ca.crt
│   │   ├── apiserver.crt
│   │   ├── etcd/
│   │   │   ├── ca.crt
│   │   │   ├── server.crt
│   │   │   └── server.key
│   └── admin.conf               ← Admin kubeconfig
├── /var/lib/etcd/               ← etcd persistent data
│   ├── member/
│   │   ├── snap/
│   │   └── wal/
└── /var/lib/kubelet/
    └── config.yaml              (staticPodPath: /etc/kubernetes/manifests)
```

---

## hostPath Patterns in Control Plane

### Pattern 1: etcd (Data Persistence)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: etcd
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: etcd
    image: registry.k8s.io/etcd:3.5.9-0
    volumeMounts:
    # Critical persistent data
    - name: etcd-data
      mountPath: /var/lib/etcd
    
    # TLS certificates (read-only)
    - name: etcd-certs
      mountPath: /etc/kubernetes/pki/etcd
      readOnly: true
  
  volumes:
  - name: etcd-data
    hostPath:
      path: /var/lib/etcd
      type: DirectoryOrCreate
  
  - name: etcd-certs
    hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
```

**Why each mount**:
- `/var/lib/etcd`: ALL cluster state (Pods, Services, Secrets, etc.)
- `/etc/kubernetes/pki/etcd`: mTLS for etcd cluster communication

**Failure impact**:
- Lose `/var/lib/etcd` → Lose entire cluster state → DISASTER
- Lose `/etc/kubernetes/pki/etcd` → etcd can't authenticate → Control plane down

---

### Pattern 2: kube-apiserver (Certificate Access)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-apiserver
    image: registry.k8s.io/kube-apiserver:v1.28.0
    command:
    - kube-apiserver
    - --etcd-servers=https://127.0.0.1:2379
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
    - --service-account-issuer=https://kubernetes.default.svc.cluster.local
    
    volumeMounts:
    # ALL PKI certificates
    - name: k8s-certs
      mountPath: /etc/kubernetes/pki
      readOnly: true
    
    # Service account token signing
    - name: ca-certs
      mountPath: /etc/ssl/certs
      readOnly: true
  
  volumes:
  - name: k8s-certs
    hostPath:
      path: /etc/kubernetes/pki
      type: DirectoryOrCreate
  
  - name: ca-certs
    hostPath:
      path: /etc/ssl/certs
      type: DirectoryOrCreate
```

**Why kube-apiserver needs so many certs**:
- Client authentication (kubectl, kubelets)
- etcd authentication
- Service account token verification
- Aggregation layer (metrics-server, custom APIs)
- Admission webhooks

**Security consideration**: `/etc/kubernetes/pki` contains **cluster-admin equivalent** credentials. Access = full cluster control.

---

## Critical Operations

### Operation 1: Backing Up Static Pod Data

```bash
#!/bin/bash
# Backup script for control plane

# 1. Backup etcd data
BACKUP_DIR="/backup/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Snapshot etcd
ETCDCTL_API=3 etcdctl snapshot save "$BACKUP_DIR/etcd-snapshot.db" \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 2. Backup certificates
tar czf "$BACKUP_DIR/pki.tar.gz" /etc/kubernetes/pki/

# 3. Backup manifests
tar czf "$BACKUP_DIR/manifests.tar.gz" /etc/kubernetes/manifests/

# 4. Backup kubeconfigs
tar czf "$BACKUP_DIR/kubeconfigs.tar.gz" \
  /etc/kubernetes/admin.conf \
  /etc/kubernetes/kubelet.conf \
  /etc/kubernetes/controller-manager.conf \
  /etc/kubernetes/scheduler.conf

echo "Backup complete: $BACKUP_DIR"
```

**Automation**:
```yaml
# CronJob for automated backups (runs on master node)
apiVersion: v1
kind: Pod
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  hostNetwork: true
  nodeName: master-1  # Pin to master node
  containers:
  - name: backup
    image: registry.k8s.io/etcd:3.5.9-0
    command:
    - /bin/sh
    - -c
    - |
      while true; do
        etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db \
          --endpoints=https://127.0.0.1:2379 \
          --cacert=/etc/kubernetes/pki/etcd/ca.crt \
          --cert=/etc/kubernetes/pki/etcd/server.crt \
          --key=/etc/kubernetes/pki/etcd/server.key
        
        # Keep last 7 days
        find /backup -name "etcd-*.db" -mtime +7 -delete
        
        sleep 3600  # Every hour
      done
    volumeMounts:
    - name: etcd-certs
      mountPath: /etc/kubernetes/pki/etcd
      readOnly: true
    - name: backup
      mountPath: /backup
  volumes:
  - name: etcd-certs
    hostPath:
      path: /etc/kubernetes/pki/etcd
  - name: backup
    hostPath:
      path: /var/backup/etcd
      type: DirectoryOrCreate
```

---

### Operation 2: Disaster Recovery

**Scenario**: Control plane node dies, `/var/lib/etcd` corrupted

```bash
# Step 1: Stop kubelet on ALL control plane nodes
for node in master-1 master-2 master-3; do
  ssh $node "sudo systemctl stop kubelet"
done

# Step 2: Restore etcd snapshot on first master
ssh master-1
sudo etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore \
  --name=master-1 \
  --initial-cluster=master-1=https://192.168.1.10:2380,master-2=https://192.168.1.11:2380,master-3=https://192.168.1.12:2380 \
  --initial-advertise-peer-urls=https://192.168.1.10:2380

# Step 3: Move restored data into place
sudo rm -rf /var/lib/etcd
sudo mv /var/lib/etcd-restore /var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd

# Step 4: Update etcd manifest to force-new-cluster (FIRST NODE ONLY)
sudo vi /etc/kubernetes/manifests/etcd.yaml
# Add: --force-new-cluster=true

# Step 5: Start kubelet on first master
sudo systemctl start kubelet

# Step 6: Wait for etcd to start (check logs)
sudo journalctl -u kubelet -f
# Wait for: "etcd cluster is healthy"

# Step 7: Remove --force-new-cluster flag
sudo vi /etc/kubernetes/manifests/etcd.yaml
# Remove: --force-new-cluster=true

# Step 8: Join other masters (one at a time)
ssh master-2
sudo rm -rf /var/lib/etcd/*
sudo systemctl start kubelet
# etcd will sync from master-1

# Repeat for master-3

# Step 9: Verify cluster health
kubectl get nodes
kubectl get pods -A
etcdctl member list
```

---

### Operation 3: Migrating Static Pod to Different hostPath

**Use case**: Move etcd data to larger disk

```bash
# Step 1: Ensure you have recent backup!
etcdctl snapshot save /backup/pre-migration.db

# Step 2: Stop kubelet
sudo systemctl stop kubelet

# Step 3: Copy data to new location
sudo rsync -av --progress /var/lib/etcd/ /mnt/new-disk/etcd/

# Step 4: Verify data integrity
sudo diff -r /var/lib/etcd/ /mnt/new-disk/etcd/

# Step 5: Update manifest
sudo vi /etc/kubernetes/manifests/etcd.yaml
# Change:
#   path: /var/lib/etcd
# To:
#   path: /mnt/new-disk/etcd

# Step 6: Update etcd command
# Change:
#   --data-dir=/var/lib/etcd
# To:
#   --data-dir=/var/lib/etcd  # (mount point remains same)

# Step 7: Start kubelet
sudo systemctl start kubelet

# Step 8: Verify etcd started correctly
sudo crictl ps | grep etcd
sudo crictl logs $(sudo crictl ps | grep etcd | awk '{print $1}')

# Step 9: Verify cluster health
kubectl get cs
etcdctl endpoint health

# Step 10: After verification (24-48 hours), remove old data
# sudo rm -rf /var/lib/etcd
```

---

## Troubleshooting Static Pods with hostPath

### Issue 1: Static Pod Not Starting

```bash
# Check kubelet logs
sudo journalctl -u kubelet -n 100 --no-pager | grep -i error

# Common errors:
# "hostPath type check failed: path does not exist"
# "failed to create container: permission denied"
# "failed to mount volumes for pod: path /var/lib/etcd is not a directory"

# Debug steps:
# 1. Verify path exists
ls -la /var/lib/etcd

# 2. Check ownership
stat /var/lib/etcd
# Should be owned by etcd user (UID 999 typically)

# 3. Check SELinux labels (if applicable)
ls -Z /var/lib/etcd

# 4. Check manifest syntax
sudo kubeadm init phase control-plane all --config /tmp/kubeadm.yaml --dry-run

# 5. Manually validate YAML
sudo yamllint /etc/kubernetes/manifests/etcd.yaml
```

### Issue 2: Pod Running But Not Functional

```bash
# etcd Pod is running but API server can't connect

# Check etcd logs
sudo crictl logs --tail=100 $(sudo crictl ps | grep etcd | awk '{print $1}')

# Common issues:
# "mvcc: database space exceeded"  → etcd disk full
# "transport: authentication handshake failed" → Certificate mismatch
# "failed to check initial cluster version" → Data corruption

# Fix disk space:
sudo etcdctl defrag --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Compact old revisions:
sudo etcdctl compact $(sudo etcdctl endpoint status --write-out="json" | jq .[0].Status.header.revision) \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Issue 3: Manifest Changes Not Taking Effect

```bash
# Modified /etc/kubernetes/manifests/etcd.yaml but Pod not restarting

# Check if kubelet is watching the directory
sudo journalctl -u kubelet -f | grep -i "static pod"

# Force kubelet to re-scan
sudo systemctl restart kubelet

# If still not working, check file permissions
ls -la /etc/kubernetes/manifests/
# Should be readable by root (kubelet runs as root)

# Check for syntax errors
sudo kubectl --kubeconfig=/etc/kubernetes/admin.conf apply --dry-run=client -f /etc/kubernetes/manifests/etcd.yaml
```

---

## Security Best Practices

### 1. Minimize hostPath Exposure

```yaml
# ❌ Bad: Mount entire /etc
volumes:
- name: config
  hostPath:
    path: /etc

# ✅ Good: Mount specific directory
volumes:
- name: k8s-certs
  hostPath:
    path: /etc/kubernetes/pki/etcd
    type: Directory
```

### 2. Always Use Read-Only When Possible

```yaml
volumeMounts:
- name: etcd-certs
  mountPath: /etc/kubernetes/pki/etcd
  readOnly: true  # ✅ Certificates don't need write access
```

### 3. Set Proper Ownership Before Pod Start

```bash
# In node bootstrap script
sudo mkdir -p /var/lib/etcd
sudo chown -R 999:999 /var/lib/etcd  # etcd UID
sudo chmod 700 /var/lib/etcd
```

### 4. Monitor hostPath Directories

```yaml
# Prometheus alert for disk space
- alert: EtcdDiskSpaceWarning
  expr: |
    (node_filesystem_avail_bytes{mountpoint="/var/lib/etcd"} 
    / node_filesystem_size_bytes{mountpoint="/var/lib/etcd"}) < 0.15
  for: 5m
  annotations:
    summary: "etcd disk space below 15%"
```

---

## Summary: Static Pod + hostPath

| Aspect | Requirement | Reason |
|--------|-------------|--------|
| **Usage** | Control plane ONLY | Bootstrap dependency |
| **Persistence** | MUST survive Pod restarts | Cluster state |
| **Backup** | Daily automated snapshots | Disaster recovery |
| **Security** | Restrict to kube-system | Prevent abuse |
| **Monitoring** | Disk space, permissions | Early warning |
| **Documentation** | Every hostPath justified | Audit compliance |

**The Golden Rule**: Static Pods + hostPath = Necessary evil. Minimize usage, maximize protection.

## 2. hostPath Usage Inside System Pods

### kube-proxy: Network Rules Management

**Why kube-proxy needs hostPath**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-proxy
  namespace: kube-system
spec:
  hostNetwork: true  # Required to manipulate host iptables
  containers:
  - name: kube-proxy
    image: registry.k8s.io/kube-proxy:v1.28.0
    command:
    - /usr/local/bin/kube-proxy
    - --config=/var/lib/kube-proxy/config.conf
    - --hostname-override=$(NODE_NAME)
    securityContext:
      privileged: true  # Required for iptables/ipvs
    volumeMounts:
    # iptables lock file
    - name: xtables-lock
      mountPath: /run/xtables.lock
    
    # Kernel modules
    - name: lib-modules
      mountPath: /lib/modules
      readOnly: true
    
    # Configuration
    - name: kube-proxy
      mountPath: /var/lib/kube-proxy
    
    # Kubeconfig
    - name: kubeconfig
      mountPath: /etc/kubernetes/kubeconfig.conf
      readOnly: true
  
  volumes:
  - name: xtables-lock
    hostPath:
      path: /run/xtables.lock
      type: FileOrCreate
  
  - name: lib-modules
    hostPath:
      path: /lib/modules
  
  - name: kube-proxy
    configMap:
      name: kube-proxy
  
  - name: kubeconfig
    hostPath:
      path: /var/lib/kube-proxy/kubeconfig.conf
```

**What kube-proxy does with hostPath**:
1. **`/run/xtables.lock`**: Ensures only one process modifies iptables at a time
2. **`/lib/modules`**: Loads kernel modules (iptables, ipvs, conntrack)
3. Sets up iptables/ipvs rules for Service load balancing

**Without hostPath**: Services wouldn't work. Traffic routing would fail.

---

### CoreDNS/kube-dns: DNS Resolution

**CoreDNS typically does NOT need hostPath**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: coredns
  namespace: kube-system
spec:
  containers:
  - name: coredns
    image: registry.k8s.io/coredns/coredns:v1.10.1
    args:
    - -conf
    - /etc/coredns/Corefile
    volumeMounts:
    - name: config-volume
      mountPath: /etc/coredns
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: coredns
      items:
      - key: Corefile
        path: Corefile
  # NO hostPath needed!
```

**Exception**: Custom DNS configurations might mount `/etc/resolv.conf`:

```yaml
# If you need to read host's DNS config
volumeMounts:
- name: host-resolv
  mountPath: /etc/host-resolv.conf
  readOnly: true

volumes:
- name: host-resolv
  hostPath:
    path: /etc/resolv.conf
    type: File
```

---

### CNI Plugins: The Necessary Evil

**Every CNI plugin needs extensive hostPath access**. Example from Calico:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: kube-system
spec:
  template:
    spec:
      hostNetwork: true
      tolerations:
      - operator: Exists
      
      initContainers:
      # Install CNI binaries
      - name: install-cni
        image: calico/cni:v3.26.0
        command: ["/opt/cni/bin/install"]
        volumeMounts:
        - name: cni-bin-dir
          mountPath: /host/opt/cni/bin
        - name: cni-net-dir
          mountPath: /host/etc/cni/net.d
      
      # Install FlexVolume driver
      - name: flexvol-driver
        image: calico/pod2daemon-flexvol:v3.26.0
        volumeMounts:
        - name: flexvol-driver-host
          mountPath: /host/driver
      
      containers:
      - name: calico-node
        image: calico/node:v3.26.0
        securityContext:
          privileged: true
        volumeMounts:
        # Network namespaces
        - name: lib-modules
          mountPath: /lib/modules
          readOnly: true
        - name: var-run-calico
          mountPath: /var/run/calico
        - name: var-lib-calico
          mountPath: /var/lib/calico
        - name: xtables-lock
          mountPath: /run/xtables.lock
        # CNI config
        - name: cni-bin-dir
          mountPath: /host/opt/cni/bin
        - name: cni-net-dir
          mountPath: /host/etc/cni/net.d
        - name: policysync
          mountPath: /var/run/nodeagent
        # Host /proc for network sysctls
        - name: proc
          mountPath: /host/proc
      
      volumes:
      - name: lib-modules
        hostPath:
          path: /lib/modules
      - name: var-run-calico
        hostPath:
          path: /var/run/calico
      - name: var-lib-calico
        hostPath:
          path: /var/lib/calico
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
      - name: cni-bin-dir
        hostPath:
          path: /opt/cni/bin
      - name: cni-net-dir
        hostPath:
          path: /etc/cni/net.d
      - name: flexvol-driver-host
        hostPath:
          path: /usr/libexec/kubernetes/kubelet-plugins/volume/exec/nodeagent~uds
          type: DirectoryOrCreate
      - name: policysync
        hostPath:
          path: /var/run/nodeagent
          type: DirectoryOrCreate
      - name: proc
        hostPath:
          path: /proc
```

**Why CNI needs ALL of this**:
- `/opt/cni/bin`: Kubelet looks for CNI plugins here (standard location)
- `/etc/cni/net.d`: CNI configuration files
- `/lib/modules`: Load kernel networking modules (vxlan, ipip, wireguard)
- `/var/run/calico`: Runtime state (network endpoints, policy cache)
- `/run/xtables.lock`: Coordinate iptables updates
- `/proc`: Modify network sysctls (e.g., enable IP forwarding)

---

## 3. hostPath + Kubelet Internal Directory Structures

### Kubelet's Filesystem Layout

```
/var/lib/kubelet/
├── config.yaml                  # Kubelet configuration
├── cpu_manager_state           # CPU manager state file
├── device-plugins/             # Device plugin sockets
│   ├── kubelet.sock            # Kubelet's gRPC socket
│   └── nvidia.sock             # NVIDIA device plugin socket
├── kubelet-plugins/            # Deprecated plugin directory
├── memory_manager_state        # Memory manager state
├── pki/                        # Kubelet certificates
│   ├── kubelet.crt
│   ├── kubelet.key
│   └── kubelet-client-current.pem
├── plugins/                    # Volume plugins
│   └── kubernetes.io/
│       ├── csi/                # CSI plugin sockets
│       │   └── ebs.csi.aws.com/
│       │       └── csi.sock
│       └── flexvolume/         # FlexVolume plugins (deprecated)
├── plugins_registry/           # Plugin registration directory
│   ├── csi-driver.sock
│   └── device-plugin.sock
├── pods/                       # Pod working directories
│   ├── <pod-uid>/
│   │   ├── containers/
│   │   │   └── <container-name>/
│   │   ├── etc-hosts           # /etc/hosts for Pod
│   │   ├── plugins/
│   │   └── volumes/            # Volume mounts
│   │       ├── kubernetes.io~configmap/
│   │       │   └── <configmap-name>/
│   │       ├── kubernetes.io~secret/
│   │       │   └── <secret-name>/
│   │       │       └── token
│   │       ├── kubernetes.io~projected/
│   │       └── kubernetes.io~empty-dir/
└── pod-resources/              # Resource allocation info
    └── kubelet_internal_checkpoint
```

### CSI Driver Registration (hostPath Required)

**How CSI drivers register with kubelet**:

```yaml
# CSI Node Plugin DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ebs-csi-node
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      # Driver registrar sidecar
      - name: node-driver-registrar
        image: registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.9.0
        args:
        - --csi-address=/csi/csi.sock
        - --kubelet-registration-path=/var/lib/kubelet/plugins/ebs.csi.aws.com/csi.sock
        volumeMounts:
        # CSI plugin socket
        - name: plugin-dir
          mountPath: /csi
        # Registration directory (kubelet watches this)
        - name: registration-dir
          mountPath: /registration
      
      # Main CSI driver
      - name: ebs-plugin
        image: public.ecr.aws/ebs-csi-driver/aws-ebs-csi-driver:v1.24.0
        args:
        - node
        - --endpoint=unix:///csi/csi.sock
        volumeMounts:
        - name: plugin-dir
          mountPath: /csi
        - name: kubelet-dir
          mountPath: /var/lib/kubelet
          mountPropagation: Bidirectional  # Critical!
        - name: device-dir
          mountPath: /dev
      
      volumes:
      # CSI plugin creates socket here
      - name: plugin-dir
        hostPath:
          path: /var/lib/kubelet/plugins/ebs.csi.aws.com/
          type: DirectoryOrCreate
      
      # Kubelet watches for new registrations
      - name: registration-dir
        hostPath:
          path: /var/lib/kubelet/plugins_registry/
          type: Directory
      
      # Access to all kubelet pod directories
      - name: kubelet-dir
        hostPath:
          path: /var/lib/kubelet
          type: Directory
      
      # Access to block devices
      - name: device-dir
        hostPath:
          path: /dev
          type: Directory
```

**The Registration Flow**:
```
1. CSI driver starts, creates socket: /var/lib/kubelet/plugins/ebs.csi.aws.com/csi.sock
2. node-driver-registrar creates registration socket: /var/lib/kubelet/plugins_registry/ebs.csi.aws.com-reg.sock
3. Kubelet (watching /var/lib/kubelet/plugins_registry/) detects new socket
4. Kubelet connects to registration socket
5. Registrar provides CSI driver info
6. Kubelet now knows to use /var/lib/kubelet/plugins/ebs.csi.aws.com/csi.sock for volumes
7. When Pod needs EBS volume:
   - Kubelet calls CSI driver via socket
   - Driver attaches EBS volume to node
   - Driver mounts volume to /var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~csi/
   - Kubelet bind-mounts into container
```

**Why `mountPropagation: Bidirectional` is critical**:
```yaml
volumeMounts:
- name: kubelet-dir
  mountPath: /var/lib/kubelet
  mountPropagation: Bidirectional  # Without this, volume mounts don't propagate!
```

**What happens**:
- CSI driver (inside container) mounts EBS volume to `/var/lib/kubelet/pods/.../volumes/...`
- With `Bidirectional`: Mount is visible BOTH inside container AND on host
- Without it: Mount only visible inside container, kubelet can't see it, Pod fails to start

---

## 4. hostPath in Operator-Based Deployments

**Operators** (like Rook, Prometheus Operator, etc.) sometimes deploy Pods with hostPath for specific functions.

### Example: Rook/Ceph Operator

**Operator itself**: NO hostPath needed
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rook-ceph-operator
  namespace: rook-ceph
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: rook-ceph-operator
        image: rook/ceph:v1.12.0
        # No host

Path volumes - it just manages other Pods
```

**OSD Pods deployed BY operator**: NEEDS hostPath
```yaml
# Created by Rook operator
apiVersion: v1
kind: Pod
metadata:
  name: rook-ceph-osd-0-abc123
  namespace: rook-ceph
  ownerReferences:
  - apiVersion: ceph.rook.io/v1
    kind: CephCluster
    name: rook-ceph
    uid: ...
spec:
  nodeName: worker-1  # Pinned to specific node
  containers:
  - name: osd
    image: rook/ceph:v1.12.0
    securityContext:
      privileged: true
    volumeMounts:
    - name: devices
      mountPath: /dev
    - name: udev
      mountPath: /run/udev
    - name: ceph-data
      mountPath: /var/lib/ceph/osd/ceph-0
  volumes:
  - name: devices
    hostPath:
      path: /dev
  - name: udev
    hostPath:
      path: /run/udev
  - name: ceph-data
    hostPath:
      path: /var/lib/rook/osd0
      type: DirectoryOrCreate
```

**Pattern**: Operator watches CRDs, creates Pods with hostPath as needed.

---
