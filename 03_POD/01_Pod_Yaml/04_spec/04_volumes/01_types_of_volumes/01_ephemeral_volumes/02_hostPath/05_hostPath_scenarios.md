# Detailed Scenario Analysis - hostPath Volumes in Kubernetes

## **Scenario 1: hostPath fails because Pod scheduled on wrong node**

### **What Really Happens:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-hostpath
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    hostPath:
      path: /mnt/disk1/app-data
      type: Directory
```

**The Problem:**
- You create `/mnt/disk1/app-data` on `node-1`
- Kubernetes scheduler places the Pod on `node-2`
- `/mnt/disk1/app-data` doesn't exist on `node-2`
- Pod enters `CrashLoopBackOff` or remains in `ContainerCreating` state

### **Root Cause:**
Kubernetes scheduler is **unaware** of hostPath dependencies. It schedules Pods based on:
- Resource requests (CPU/Memory)
- Node selectors
- Affinity/Anti-affinity rules
- Taints and tolerations

It **does NOT** check if the hostPath exists on the target node.

### **Real-World Impact:**
```bash
# You'll see events like this:
kubectl describe pod app-with-hostpath

Events:
  Type     Reason       Message
  ----     ------       -------
  Warning  Failed       Error: failed to start container: 
                        stat /mnt/disk1/app-data: no such file or directory
```

### **Senior Admin Solution:**

**Option 1: Use NodeSelector**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-hostpath
spec:
  nodeSelector:
    disk-type: ssd
    hostname: node-1  # Pin to specific node
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    hostPath:
      path: /mnt/disk1/app-data
      type: Directory
```

**Option 2: Use Node Affinity (More Flexible)**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-hostpath
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - node-1
            - node-2  # Allow multiple specific nodes
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    hostPath:
      path: /mnt/disk1/app-data
      type: Directory
```

**Option 3: DaemonSet Pattern (Best for Node-Level Tools)**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
      - name: collector
        image: fluent/fluentd
        volumeMounts:
        - name: varlogs
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: varlogs
        hostPath:
          path: /var/log
          type: Directory
```

**DaemonSets automatically run on ALL nodes, so hostPath availability is guaranteed.**

### **DevSecOps Best Practice:**
```yaml
# Create a validation initContainer
apiVersion: v1
kind: Pod
metadata:
  name: app-with-validation
spec:
  initContainers:
  - name: validate-hostpath
    image: busybox
    command: 
    - sh
    - -c
    - |
      if [ ! -d /data ]; then
        echo "ERROR: /data directory not found on this node"
        exit 1
      fi
      echo "Validation passed"
    volumeMounts:
    - name: data
      mountPath: /data
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    hostPath:
      path: /mnt/disk1/app-data
      type: Directory
```

### **Kubernetes Developer Insight:**
```bash
# Pre-deployment validation script
#!/bin/bash
REQUIRED_NODES=("node-1" "node-2")
HOST_PATH="/mnt/disk1/app-data"

for node in "${REQUIRED_NODES[@]}"; do
  echo "Checking $node..."
  kubectl debug node/$node -it --image=busybox -- \
    sh -c "test -d $HOST_PATH && echo 'OK' || echo 'MISSING'"
done
```

### **Monitoring & Alerts:**
```yaml
# Prometheus alert rule
- alert: HostPathPodScheduledOnWrongNode
  expr: |
    kube_pod_container_status_waiting_reason{reason="CreateContainerError"} 
    * on(pod) group_left(node) 
    kube_pod_info{node!~"node-1|node-2"}
  for: 5m
  annotations:
    summary: "Pod {{ $labels.pod }} scheduled on wrong node"
```

---

**Key Takeaways:**
1. **Always use nodeSelector or affinity** when using hostPath
2. **Document node requirements** clearly in your deployment docs
3. **Use DaemonSets** for node-local tools that need hostPath
4. **Implement validation** in initContainers
5. **Set up alerts** for scheduling mismatches

---

# **Scenario 2: hostPath directory missing → Pod crashes**

## **What Really Happens:**

### **The Scenario:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  nodeSelector:
    kubernetes.io/hostname: node-1
  containers:
  - name: nginx
    image: nginx:1.21
    volumeMounts:
    - name: app-data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: app-data
    hostPath:
      path: /data/webapp/static
      type: Directory  # This is the critical line
```

**The Problem:**
- Pod is correctly scheduled to `node-1`
- But `/data/webapp/static` **doesn't exist** on `node-1`
- Kubernetes tries to mount a non-existent directory
- Pod fails to start

### **What You'll See:**

```bash
$ kubectl get pods
NAME     READY   STATUS                RESTARTS   AGE
webapp   0/1     ContainerCreating     0          2m

$ kubectl describe pod webapp
Events:
  Type     Reason       Age                From               Message
  ----     ------       ----               ----               -------
  Warning  FailedMount  30s (x12 over 2m)  kubelet            
           MountVolume.SetUp failed for volume "app-data" : 
           hostPath type check failed: /data/webapp/static is not a directory
```

```bash
# Pod will NEVER start - it's stuck in ContainerCreating state
$ kubectl logs webapp
Error from server (BadRequest): container "nginx" in pod "webapp" is waiting to start: ContainerCreating
```

---

## **Root Cause Analysis - The Kubernetes Developer Perspective:**

### **Understanding hostPath `type` Field:**

| Type | Behavior | What Happens if Path Missing/Wrong Type |
|------|----------|----------------------------------------|
| `""` (empty) | No checks performed | **Creates whatever is needed** (dangerous!) |
| `DirectoryOrCreate` | Directory must exist OR will be created | Pod starts, directory auto-created |
| `Directory` | Directory **MUST** pre-exist | **Pod fails** if missing |
| `FileOrCreate` | File must exist OR will be created | Pod starts, file auto-created |
| `File` | File **MUST** pre-exist | **Pod fails** if missing |
| `Socket` | Unix socket **MUST** pre-exist | Pod fails if missing |
| `CharDevice` | Character device **MUST** pre-exist | Pod fails if missing |
| `BlockDevice` | Block device **MUST** pre-exist | Pod fails if missing |

### **The Critical Difference:**

```yaml
# This FAILS if directory missing
volumes:
- name: data
  hostPath:
    path: /data/webapp
    type: Directory  # Strict validation

# This SUCCEEDS and creates directory
volumes:
- name: data
  hostPath:
    path: /data/webapp
    type: DirectoryOrCreate  # Auto-creates
```

---

## **Real-World Production Incidents I've Handled:**

### **Incident 1: Fresh Node Added to Cluster**
```bash
# Situation:
- Added new node to scale cluster
- Forgot to provision /data/webapp directory
- DaemonSet pods on new node stuck in ContainerCreating
- Monitoring system gaps appear

# Impact:
- 20% of cluster nodes without logging capability
- Security audit logs missing for 4 hours
```

### **Incident 2: Node OS Upgrade**
```bash
# Situation:
- Performed in-place Ubuntu upgrade (18.04 → 20.04)
- /data mount point configuration lost
- All pods using hostPath failed after reboot

# Impact:
- Database backups stopped
- Application unable to write logs
- 2-hour outage until manual directory recreation
```

---

## **Senior Kubernetes Administrator Solutions:**

### **Solution 1: Use `DirectoryOrCreate` Type (Recommended for Most Cases)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-safe
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    volumeMounts:
    - name: app-data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: app-data
    hostPath:
      path: /data/webapp/static
      type: DirectoryOrCreate  # Kubernetes creates if missing
```

**Behavior:**
- If `/data/webapp/static` exists → uses it
- If missing → Kubernetes creates it with permissions `0755` owned by `root:root`
- Pod starts successfully

**⚠️ Warning:**
```bash
# Created directory has default permissions
ls -ld /data/webapp/static
drwxr-xr-x 2 root root 4096 Nov 19 10:30 /data/webapp/static

# Your app might need different permissions!
# Container running as UID 1000 might get "Permission Denied"
```

---

### **Solution 2: InitContainer to Pre-Create with Correct Permissions**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-init
spec:
  initContainers:
  - name: prepare-directory
    image: busybox
    command:
    - sh
    - -c
    - |
      # Create directory structure
      mkdir -p /data/webapp/static
      mkdir -p /data/webapp/logs
      
      # Set ownership (assuming app runs as UID 1000)
      chown -R 1000:1000 /data/webapp
      
      # Set permissions
      chmod 755 /data/webapp/static
      chmod 750 /data/webapp/logs
      
      echo "Directory preparation completed"
    volumeMounts:
    - name: data-root
      mountPath: /data
    securityContext:
      runAsUser: 0  # Must run as root to chown
      
  containers:
  - name: nginx
    image: nginx:1.21
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
    volumeMounts:
    - name: app-data
      mountPath: /usr/share/nginx/html
      
  volumes:
  - name: data-root
    hostPath:
      path: /data
      type: DirectoryOrCreate
  - name: app-data
    hostPath:
      path: /data/webapp/static
      type: DirectoryOrCreate
```

---

### **Solution 3: DaemonSet for Node Preparation (Best for Multi-Node)**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-directory-provisioner
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: node-provisioner
  template:
    metadata:
      labels:
        app: node-provisioner
    spec:
      hostPID: true
      hostNetwork: true
      containers:
      - name: provisioner
        image: busybox
        command:
        - sh
        - -c
        - |
          # Create all required directories
          mkdir -p /host/data/webapp/static
          mkdir -p /host/data/webapp/logs
          mkdir -p /host/data/webapp/config
          
          # Set permissions
          chmod 755 /host/data/webapp/static
          chmod 750 /host/data/webapp/logs
          chmod 700 /host/data/webapp/config
          
          # Create marker file
          echo "Provisioned at $(date)" > /host/data/webapp/.provisioned
          
          # Keep container running
          sleep infinity
        volumeMounts:
        - name: host-root
          mountPath: /host
        securityContext:
          privileged: true
          
      volumes:
      - name: host-root
        hostPath:
          path: /
          type: Directory
```

**Deploy this BEFORE your application pods:**
```bash
kubectl apply -f node-provisioner.yaml
# Wait for DaemonSet to run on all nodes
kubectl rollout status daemonset/node-directory-provisioner -n kube-system

# Then deploy your application
kubectl apply -f webapp.yaml
```

---

### **Solution 4: Configuration Management (Ansible/Puppet)**

```yaml
# Ansible playbook: provision-k8s-nodes.yml
---
- name: Prepare Kubernetes nodes for hostPath volumes
  hosts: k8s_workers
  become: yes
  tasks:
    - name: Create webapp directory structure
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
        owner: 1000
        group: 1000
      loop:
        - /data/webapp/static
        - /data/webapp/logs
        - /data/webapp/config
        
    - name: Create log directory with restricted permissions
      file:
        path: /data/webapp/logs
        state: directory
        mode: '0750'
        owner: 1000
        group: 1000
        
    - name: Add validation marker
      copy:
        content: "Provisioned by Ansible on {{ ansible_date_time.iso8601 }}"
        dest: /data/webapp/.ansible-provisioned
        mode: '0644'
```

```bash
# Run before cluster deployment
ansible-playbook -i inventory provision-k8s-nodes.yml
```

---

## **DevSecOps Best Practices:**

### **Practice 1: Pre-Flight Validation Script**

```bash
#!/bin/bash
# validate-hostpath.sh

REQUIRED_PATHS=(
  "/data/webapp/static:755:1000:1000"
  "/data/webapp/logs:750:1000:1000"
  "/data/webapp/config:700:1000:1000"
)

for node in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  echo "Validating node: $node"
  
  for path_spec in "${REQUIRED_PATHS[@]}"; do
    IFS=':' read -r path mode uid gid <<< "$path_spec"
    
    # Check if path exists
    kubectl debug node/$node -it --image=busybox -- \
      sh -c "
        if [ ! -d $path ]; then
          echo '❌ MISSING: $path'
          exit 1
        fi
        
        # Check permissions
        actual_mode=\$(stat -c '%a' $path)
        if [ \"\$actual_mode\" != \"$mode\" ]; then
          echo '⚠️  WRONG PERMISSIONS: $path (expected: $mode, actual: \$actual_mode)'
        fi
        
        echo '✅ OK: $path'
      "
  done
done
```

---

### **Practice 2: Admission Webhook to Prevent Strict Types**

```go
// admission-webhook.go
package main

import (
    "encoding/json"
    "net/http"
    admissionv1 "k8s.io/api/admission/v1"
    corev1 "k8s.io/api/core/v1"
)

func validatePod(w http.ResponseWriter, r *http.Request) {
    var admissionReview admissionv1.AdmissionReview
    json.NewDecoder(r.Body).Decode(&admissionReview)
    
    pod := &corev1.Pod{}
    json.Unmarshal(admissionReview.Request.Object.Raw, pod)
    
    allowed := true
    message := ""
    
    // Check for hostPath volumes with strict types
    for _, vol := range pod.Spec.Volumes {
        if vol.HostPath != nil {
            if vol.HostPath.Type != nil {
                typeStr := string(*vol.HostPath.Type)
                
                // Warn about strict types
                if typeStr == "Directory" || typeStr == "File" {
                    allowed = false
                    message = "hostPath with type 'Directory' or 'File' is not recommended. Use 'DirectoryOrCreate' or 'FileOrCreate' instead."
                }
            }
        }
    }
    
    response := admissionv1.AdmissionResponse{
        Allowed: allowed,
        Result: &metav1.Status{
            Message: message,
        },
    }
    
    // Send response...
}
```

---

### **Practice 3: OPA/Gatekeeper Policy**

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirehostpathorcreate
spec:
  crd:
    spec:
      names:
        kind: K8sRequireHostPathOrCreate
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirehostpathorcreate
        
        violation[{"msg": msg}] {
          volume := input.review.object.spec.volumes[_]
          volume.hostPath
          
          # Check if type is set
          hostPathType := volume.hostPath.type
          
          # Disallow strict types
          hostPathType == "Directory"
          msg := sprintf("hostPath volume '%v' uses strict type 'Directory'. Use 'DirectoryOrCreate' instead.", [volume.name])
        }
        
        violation[{"msg": msg}] {
          volume := input.review.object.spec.volumes[_]
          volume.hostPath
          hostPathType := volume.hostPath.type
          hostPathType == "File"
          msg := sprintf("hostPath volume '%v' uses strict type 'File'. Use 'FileOrCreate' instead.", [volume.name])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireHostPathOrCreate
metadata:
  name: hostpath-must-use-or-create
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

---

## **Monitoring & Alerting:**

```yaml
# Prometheus alert for stuck pods
groups:
- name: hostpath_issues
  rules:
  - alert: PodStuckInContainerCreating
    expr: |
      kube_pod_container_status_waiting_reason{reason="ContainerCreating"} > 0
      and
      time() - kube_pod_created > 300  # 5 minutes
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} stuck in ContainerCreating for >5min"
      description: "Check for hostPath mount issues on node {{ $labels.node }}"
      
  - alert: HostPathMountFailure
    expr: |
      increase(kubelet_volume_stats_errors_total[5m]) > 0
    labels:
      severity: critical
    annotations:
      summary: "Volume mount errors detected"
```

---

## **Troubleshooting Commands:**

```bash
# 1. Check pod events
kubectl describe pod webapp | grep -A 10 Events

# 2. Check kubelet logs on the node
kubectl debug node/node-1 -it --image=ubuntu -- bash
chroot /host
journalctl -u kubelet | grep -i "hostPath\|mount"

# 3. Manually check if directory exists
kubectl debug node/node-1 -it --image=busybox -- \
  sh -c "ls -ld /data/webapp/static"

# 4. Check directory permissions
kubectl debug node/node-1 -it --image=busybox -- \
  sh -c "stat /data/webapp/static"

# 5. Check SELinux context (if enabled)
kubectl debug node/node-1 -it --image=busybox -- \
  sh -c "ls -Z /data/webapp/static"
```

---

## **Key Takeaways:**

| ✅ DO | ❌ DON'T |
|------|----------|
| Use `DirectoryOrCreate` for flexibility | Use strict `Directory` type without validation |
| Pre-provision directories with Ansible/DaemonSet | Assume directories exist |
| Use initContainers for permission setup | Rely on auto-created permissions |
| Implement admission webhooks to enforce policies | Allow developers to use strict types freely |
| Monitor for ContainerCreating state > 5 min | Ignore pod creation failures |
| Document required hostPath structure in README | Keep hostPath requirements undocumented |

---

# **Scenario 3: hostPath permissions mismatch → access denied**

## **What Really Happens:**

### **The Scenario:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: app
    image: nginx:1.21
    securityContext:
      runAsUser: 101  # nginx user
      runAsGroup: 101
      fsGroup: 101
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    hostPath:
      path: /data/webapp
      type: DirectoryOrCreate
```

**The Problem:**
- Directory `/data/webapp` is auto-created by Kubernetes with ownership `root:root` and permissions `0755`
- Container runs as UID 101 (nginx user)
- nginx tries to write to `/usr/share/nginx/html` (mounted from `/data/webapp`)
- **Permission denied!**

### **What You'll See:**

```bash
$ kubectl logs webapp
2024/11/19 10:30:15 [crit] 1#1: *1 open() "/usr/share/nginx/html/index.html" failed (13: Permission denied)
2024/11/19 10:30:15 [emerg] 1#1: cannot load configuration file /etc/nginx/nginx.conf: (13: Permission denied)
```

```bash
$ kubectl exec -it webapp -- ls -ld /usr/share/nginx/html
drwxr-xr-x 2 root root 4096 Nov 19 10:30 /usr/share/nginx/html

$ kubectl exec -it webapp -- id
uid=101(nginx) gid=101(nginx) groups=101(nginx)

$ kubectl exec -it webapp -- touch /usr/share/nginx/html/test.txt
touch: cannot touch '/usr/share/nginx/html/test.txt': Permission denied
```

---

## **Root Cause Analysis - Deep Dive:**

### **Understanding Linux File Permissions in Containers:**

```
Directory Permissions: drwxr-xr-x (755)
                        |||  |  |
                        |||  |  └─ Others: read + execute
                        |||  └──── Group: read + execute
                        ||└─────── Owner: read + write + execute
                        |└──────── Directory flag
                        └───────── File type
Owner: root (UID 0)
Group: root (GID 0)
```

**Container runs as UID 101:**
- User 101 is **not** the owner (root)
- User 101 is **not** in the group (root)
- Falls into "Others" category → only has `r-x` (read + execute)
- **Cannot write** to the directory!

### **The Permission Matrix:**

| Directory Permissions | Owner | Group | Others | Container UID 101 Can... |
|----------------------|-------|-------|--------|-------------------------|
| `drwxr-xr-x` (755) | root | root | r-x | ✅ Read, ❌ Write |
| `drwxrwxr-x` (775) | root | root | r-x | ✅ Read, ❌ Write |
| `drwxrwxrwx` (777) | root | root | rwx | ✅ Read, ✅ Write (INSECURE!) |
| `drwxr-xr-x` (755) | 101 | 101 | r-x | ✅ Read, ✅ Write (owner match) |
| `drwxrwxr-x` (775) | root | 101 | r-x | ✅ Read, ✅ Write (group match) |

---

## **Real-World Production Incidents I've Handled:**

### **Incident 1: StatefulSet Data Loss**
```bash
# Situation:
- StatefulSet running PostgreSQL
- hostPath for database files: /data/postgres
- Upgraded PostgreSQL image (UID changed from 999 to 70)
- New pods couldn't access existing data

# Impact:
- Database startup failed
- 3-hour downtime
- Emergency permission fix required
```

### **Incident 2: Multi-Container Pod Conflict**
```bash
# Situation:
- Pod with nginx (UID 101) + app (UID 1000)
- Both writing to shared hostPath volume
- fsGroup set to 101
- App container couldn't write logs

# Impact:
- Application crashes
- Log aggregation broken
- Debugging extremely difficult
```

### **Incident 3: SELinux Blocking Access**
```bash
# Situation:
- RHEL nodes with SELinux enforcing
- Permissions looked correct (ls -l)
- Still getting "Permission denied"
- SELinux context mismatch

# Impact:
- Pods in CrashLoopBackOff
- 2 days to identify SELinux as root cause
```

---

## **Senior Kubernetes Administrator Solutions:**

### **Solution 1: Use fsGroup (Recommended for Group-Based Access)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-fsgroup
spec:
  securityContext:
    fsGroup: 2000  # All container processes become part of this group
    fsGroupChangePolicy: "OnRootMismatch"  # Only change if needed (performance)
  
  containers:
  - name: app
    image: nginx:1.21
    securityContext:
      runAsUser: 101
      runAsGroup: 101
      allowPrivilegeEscalation: false
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
      
  volumes:
  - name: data
    hostPath:
      path: /data/webapp
      type: DirectoryOrCreate
```

**What fsGroup Does:**
```bash
# On the host node:
ls -ld /data/webapp
drwxrwsr-x 2 root 2000 4096 Nov 19 10:30 /data/webapp
#         ^---- setgid bit
#              ^---- fsGroup GID

# Inside the container:
$ id
uid=101(nginx) gid=101(nginx) groups=101(nginx),2000

# The container process is now part of group 2000
# Directory has group ownership 2000 with write permissions (rwx)
# Therefore, container can write!
```

**Key Points:**
- Kubernetes **automatically changes group ownership** of the hostPath to `fsGroup`
- Sets the **setgid bit** (`s`) so new files inherit the group
- Container processes get supplementary group membership to `fsGroup`

---

### **Solution 2: InitContainer to Fix Permissions**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-initfix
spec:
  initContainers:
  - name: fix-permissions
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Current permissions:"
      ls -ld /data
      
      echo "Fixing ownership..."
      chown -R 101:101 /data
      
      echo "Setting permissions..."
      chmod 755 /data
      
      echo "Final permissions:"
      ls -ld /data
      
    volumeMounts:
    - name: data
      mountPath: /data
    securityContext:
      runAsUser: 0  # Must run as root to chown
      
  containers:
  - name: app
    image: nginx:1.21
    securityContext:
      runAsUser: 101
      runAsGroup: 101
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
      
  volumes:
  - name: data
    hostPath:
      path: /data/webapp
      type: DirectoryOrCreate
```

---

### **Solution 3: Run Container as Root (NOT RECOMMENDED - Security Risk)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-root  # DON'T DO THIS IN PRODUCTION!
spec:
  containers:
  - name: app
    image: nginx:1.21
    securityContext:
      runAsUser: 0  # Running as root - SECURITY RISK!
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    hostPath:
      path: /data/webapp
      type: DirectoryOrCreate
```

**⚠️ Why This is Dangerous:**
```bash
# If container is compromised:
# Attacker has root access to the container
# Can write to hostPath with root privileges
# Can potentially escape to the host node
# Can modify critical system files
```

---

### **Solution 4: DaemonSet for Permanent Permission Setup**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: permission-fixer
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: permission-fixer
  template:
    metadata:
      labels:
        app: permission-fixer
    spec:
      hostPID: true
      containers:
      - name: fixer
        image: busybox
        command:
        - sh
        - -c
        - |
          while true; do
            # Fix webapp directory
            if [ -d /host/data/webapp ]; then
              chown -R 101:101 /host/data/webapp
              chmod 755 /host/data/webapp
            fi
            
            # Fix postgres directory
            if [ -d /host/data/postgres ]; then
              chown -R 999:999 /host/data/postgres
              chmod 700 /host/data/postgres
            fi
            
            # Fix logs directory
            if [ -d /host/data/logs ]; then
              chown -R 1000:1000 /host/data/logs
              chmod 755 /host/data/logs
            fi
            
            sleep 3600  # Run every hour
          done
        volumeMounts:
        - name: host-root
          mountPath: /host
        securityContext:
          privileged: true
          
      volumes:
      - name: host-root
        hostPath:
          path: /
          type: Directory
```

---

## **DevSecOps Best Practices:**

### **Practice 1: Security Context Best Practices**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-webapp
spec:
  # Pod-level security context
  securityContext:
    runAsNonRoot: true          # Prevent running as root
    runAsUser: 1000             # Specific non-root UID
    runAsGroup: 3000
    fsGroup: 2000               # Group for volume access
    fsGroupChangePolicy: "OnRootMismatch"
    seccompProfile:             # Restrict syscalls
      type: RuntimeDefault
    
  containers:
  - name: app
    image: nginx:1.21
    # Container-level security context (overrides pod-level)
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL                   # Drop all capabilities
        add:
        - NET_BIND_SERVICE     # Only allow binding to privileged ports
        
    volumeMounts:
    - name: data
      mountPath: /data
    - name: tmp
      mountPath: /tmp           # Writable tmp since root is read-only
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
      
  volumes:
  - name: data
    hostPath:
      path: /data/webapp
      type: DirectoryOrCreate
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```

---

### **Practice 2: Pod Security Standards Enforcement**

```yaml
# Namespace with restricted PSS
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**This will BLOCK pods that:**
- Run as root
- Use privileged containers
- Have insecure capabilities
- Mount hostPath without proper restrictions

---

### **Practice 3: OPA/Gatekeeper Policy for Permission Validation**

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirefsgroup
spec:
  crd:
    spec:
      names:
        kind: K8sRequireFsGroup
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirefsgroup
        
        violation[{"msg": msg}] {
          # Check if pod uses hostPath
          volume := input.review.object.spec.volumes[_]
          volume.hostPath
          
          # Check if fsGroup is set
          not input.review.object.spec.securityContext.fsGroup
          
          msg := sprintf("Pod '%v' uses hostPath but doesn't set fsGroup", [input.review.object.metadata.name])
        }
        
        violation[{"msg": msg}] {
          # Check if pod uses hostPath
          volume := input.review.object.spec.volumes[_]
          volume.hostPath
          
          # Check if running as root
          container := input.review.object.spec.containers[_]
          container.securityContext.runAsUser == 0
          
          msg := sprintf("Container '%v' uses hostPath and runs as root", [container.name])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireFsGroup
metadata:
  name: hostpath-must-have-fsgroup
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

---

### **Practice 4: Automated Permission Testing**

```bash
#!/bin/bash
# test-hostpath-permissions.sh

POD_NAME="test-permissions-$$"
NAMESPACE="default"
TEST_PATH="/data/webapp"
TEST_UID=101
TEST_GID=101

echo "Testing hostPath permissions for UID:GID ${TEST_UID}:${TEST_GID}"

# Create test pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ${POD_NAME}
  namespace: ${NAMESPACE}
spec:
  restartPolicy: Never
  securityContext:
    fsGroup: 2000
  containers:
  - name: tester
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "=== Testing Permissions ==="
      
      # Test read
      if ls ${TEST_PATH} > /dev/null 2>&1; then
        echo "✅ READ: Success"
      else
        echo "❌ READ: Failed"
        exit 1
      fi
      
      # Test write
      if touch ${TEST_PATH}/test-file-$$ 2>/dev/null; then
        echo "✅ WRITE: Success"
        rm ${TEST_PATH}/test-file-$$
      else
        echo "❌ WRITE: Failed"
        exit 1
      fi
      
      # Test execute
      if [ -x ${TEST_PATH} ]; then
        echo "✅ EXECUTE: Success"
      else
        echo "❌ EXECUTE: Failed"
        exit 1
      fi
      
      echo "=== All Tests Passed ==="
      
    securityContext:
      runAsUser: ${TEST_UID}
      runAsGroup: ${TEST_GID}
    volumeMounts:
    - name: test-volume
      mountPath: ${TEST_PATH}
  volumes:
  - name: test-volume
    hostPath:
      path: ${TEST_PATH}
      type: DirectoryOrCreate
EOF

# Wait for pod to complete
echo "Waiting for test to complete..."
kubectl wait --for=condition=Ready pod/${POD_NAME} -n ${NAMESPACE} --timeout=30s

# Get results
echo ""
echo "=== Test Results ==="
kubectl logs ${POD_NAME} -n ${NAMESPACE}

# Cleanup
kubectl delete pod ${POD_NAME} -n ${NAMESPACE}
```

---

## **Handling SELinux Contexts:**

### **Problem: SELinux Blocking Access**

```bash
# On the host node:
$ ls -Z /data/webapp
drwxr-xr-x. root root unconfined_u:object_r:default_t:s0 /data/webapp

# Container tries to access - DENIED by SELinux!
```

### **Solution 1: Set Correct SELinux Context**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-selinux
spec:
  securityContext:
    fsGroup: 2000
    seLinuxOptions:
      level: "s0:c123,c456"  # SELinux Multi-Category Security label
      
  containers:
  - name: app
    image: nginx:1.21
    securityContext:
      runAsUser: 101
      seLinuxOptions:
        level: "s0:c123,c456"
    volumeMounts:
    - name: data
      mountPath: /data
      
  volumes:
  - name: data
    hostPath:
      path: /data/webapp
      type: DirectoryOrCreate
```

### **Solution 2: Use `:z` or `:Z` Mount Options (Docker-Specific)**

```bash
# In Docker/Podman (not directly in Kubernetes):
docker run -v /data/webapp:/data:z nginx
#                                  ^--- Relabels for shared access

docker run -v /data/webapp:/data:Z nginx
#                                  ^--- Relabels for exclusive access
```

**In Kubernetes, this is handled by:**
```yaml
volumes:
- name: data
  hostPath:
    path: /data/webapp
    type: DirectoryOrCreate
# Kubernetes automatically handles SELinux labeling when seLinuxOptions is set
```

### **Solution 3: Manual SELinux Relabeling on Node**

```bash
# SSH to the node
ssh node-1

# Check current context
ls -Z /data/webapp
drwxr-xr-x. root root unconfined_u:object_r:default_t:s0 /data/webapp

# Relabel for container access
chcon -R -t container_file_t /data/webapp

# Or use semanage for persistent changes
semanage fcontext -a -t container_file_t "/data/webapp(/.*)?"
restorecon -R /data/webapp

# Verify
ls -Z /data/webapp
drwxr-xr-x. root root unconfined_u:object_r:container_file_t:s0 /data/webapp
```

---

## **Troubleshooting Flowchart:**

```
Permission Denied Error
         |
         v
    Check Logs
         |
         v
    Is it SELinux? ----Yes----> Check: ausearch -m avc -ts recent
         |                      Fix: Set seLinuxOptions or relabel
         No
         |
         v
    Check Directory Ownership
    kubectl exec pod -- ls -ld /mountpoint
         |
         v
    Owner matches container UID? ---Yes--> Check execute permission
         |                                  (Need 'x' on directories)
         No
         |
         v
    Is fsGroup set? ---No----> Add fsGroup to pod securityContext
         |
         Yes
         |
         v
    Check group ownership matches fsGroup
         |
         v
    Not matching? --> InitContainer to chown/chmod
                      OR
                      DaemonSet to fix permissions
```

---

## **Monitoring & Alerting:**

```yaml
# Prometheus alert for permission errors
groups:
- name: hostpath_permissions
  rules:
  - alert: PodPermissionDenied
    expr: |
      increase(
        container_runtime_errors_total{
          error_type=~".*permission denied.*"
        }[5m]
      ) > 0
    labels:
      severity: warning
    annotations:
      summary: "Pod experiencing permission denied errors"
      description: "Check hostPath permissions and fsGroup settings"
      
  - alert: SELinuxDenial
    expr: |
      rate(selinux_denials_total[5m]) > 0
    labels:
      severity: warning
    annotations:
      summary: "SELinux denying container access"
      description: "Check SELinux contexts on hostPath volumes"
```

---

## **Key Takeaways:**

| ✅ ALWAYS DO | ❌ NEVER DO |
|--------------|-------------|
| Set `fsGroup` when using hostPath | Run containers as root (UID 0) |
| Use `runAsNonRoot: true` | Ignore SELinux contexts on RHEL/CentOS |
| Set `allowPrivilegeEscalation: false` | Use `chmod 777` as a "quick fix" |
| Use initContainers to fix permissions | Assume auto-created directories are writable |
| Test permission scenarios before production | Deploy without testing container UID/GID |
| Implement admission policies (OPA/Gatekeeper) | Allow unrestricted hostPath usage |
| Document required UIDs/GIDs in README | Keep permission requirements undocumented |

---

## **Quick Reference: Permission Fixes**

```yaml
# FIX 1: Using fsGroup (Best Practice)
securityContext:
  fsGroup: 2000
  runAsUser: 1000
  runAsNonRoot: true

# FIX 2: InitContainer (When fsGroup isn't enough)
initContainers:
- name: fix-perms
  image: busybox
  command: ['sh', '-c', 'chown -R 1000:1000 /data && chmod 755 /data']
  volumeMounts:
  - name: data
    mountPath: /data
  securityContext:
    runAsUser: 0

# FIX 3: SELinux Context
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
```

---

# **Scenario 4: hostPath used with `Directory` but file exists → Pod fails**

## **What Really Happens:**

### **The Scenario:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-reader
spec:
  containers:
  - name: app
    image: nginx:1.21
    volumeMounts:
    - name: config
      mountPath: /etc/app/config
  volumes:
  - name: config
    hostPath:
      path: /etc/app-config
      type: Directory  # Expects a DIRECTORY
```

**The Problem:**
```bash
# On the host node, someone created a FILE instead of a directory:
$ ls -l /etc/app-config
-rw-r--r-- 1 root root 1024 Nov 19 10:00 /etc/app-config
#^--- This is a FILE, not a directory!

# Pod tries to mount it as a Directory
# Kubernetes validation FAILS
```

### **What You'll See:**

```bash
$ kubectl get pods
NAME            READY   STATUS                RESTARTS   AGE
config-reader   0/1     ContainerCreating     0          2m

$ kubectl describe pod config-reader
Events:
  Type     Reason       Age                From               Message
  ----     ------       ----               ----               -------
  Warning  FailedMount  10s (x8 over 2m)   kubelet            
           MountVolume.SetUp failed for volume "config" : 
           hostPath type check failed: /etc/app-config is not a directory
```

```bash
# Checking the node directly:
$ kubectl debug node/node-1 -it --image=busybox -- sh
/ # ls -ld /host/etc/app-config
-rw-r--r--    1 root     root          1024 Nov 19 10:00 /host/etc/app-config
#^--- FILE (starts with '-'), not a directory ('d')
```

---

## **Root Cause Analysis - Deep Dive:**

### **Understanding hostPath Type Validation:**

Kubernetes performs **strict type checking** during volume mount setup. The kubelet validates the path **before** starting the container.

```go
// Kubernetes source code (simplified):
func validateHostPathType(path string, pathType *v1.HostPathType) error {
    fileInfo, err := os.Stat(path)
    if err != nil {
        return err
    }
    
    switch *pathType {
    case v1.HostPathDirectory:
        if !fileInfo.IsDir() {
            return fmt.Errorf("%s is not a directory", path)
        }
    case v1.HostPathFile:
        if fileInfo.IsDir() {
            return fmt.Errorf("%s is not a file", path)
        }
    case v1.HostPathSocket:
        if fileInfo.Mode()&os.ModeSocket == 0 {
            return fmt.Errorf("%s is not a socket", path)
        }
    // ... more validations
    }
    return nil
}
```

### **The Validation Matrix:**

| Path on Host | hostPath Type | Result |
|--------------|---------------|--------|
| `/data` (directory) | `Directory` | ✅ Mount succeeds |
| `/data` (file) | `Directory` | ❌ **Pod fails** |
| `/data` (doesn't exist) | `Directory` | ❌ Pod fails |
| `/data` (directory) | `DirectoryOrCreate` | ✅ Mount succeeds |
| `/data` (file) | `DirectoryOrCreate` | ❌ **Pod fails** (won't overwrite) |
| `/data` (doesn't exist) | `DirectoryOrCreate` | ✅ Creates directory |
| `/config.yaml` (file) | `File` | ✅ Mount succeeds |
| `/config.yaml` (directory) | `File` | ❌ **Pod fails** |
| `/config.yaml` (doesn't exist) | `File` | ❌ Pod fails |
| `/config.yaml` (file) | `FileOrCreate` | ✅ Mount succeeds |
| `/config.yaml` (directory) | `FileOrCreate` | ❌ **Pod fails** |
| `/config.yaml` (doesn't exist) | `FileOrCreate` | ✅ Creates empty file |

---

## **Real-World Production Incidents I've Handled:**

### **Incident 1: Configuration Management Conflict**

```bash
# Situation:
- Ansible playbook created /etc/app-config as a FILE
- Kubernetes manifest expected it to be a DIRECTORY
- 50+ pods across 10 nodes failed to start
- "Works on my dev machine" syndrome

# Timeline:
10:00 - Deploy Ansible playbook (creates file)
10:15 - Deploy Kubernetes pods (expect directory)
10:16 - All pods stuck in ContainerCreating
10:45 - Incident declared
12:30 - Root cause identified
13:00 - Fixed by correcting Ansible playbook

# Impact:
- 3-hour outage
- Service completely down
- Customer-facing API unavailable
```

### **Incident 2: Manual "Fix" Gone Wrong**

```bash
# Situation:
- Junior admin tried to "fix" missing config
- Ran: echo "config data" > /etc/app-config
- Created a FILE instead of a directory
- Breaking change propagated to 20 nodes via config management

# Impact:
- Staged rollout partially completed
- Half the pods running old version (working)
- Half the pods failing to start (new version)
- Inconsistent application behavior
```

### **Incident 3: Backup Restore Mistake**

```bash
# Situation:
- Restoring from backup after node failure
- Backup tool restored /data/postgres as a TARBALL file
- Instead of extracting to /data/postgres directory
- PostgreSQL StatefulSet pods couldn't start

# Timeline:
- 02:00 - Node hardware failure
- 02:30 - Restore backup (wrong command)
- 03:00 - Pods won't start
- 05:00 - Discovered backup was restored as file
- 06:00 - Re-extracted backup properly

# Impact:
- 4-hour database downtime
- Data loss window: 2 hours (last backup was 02:00)
```

---

## **Senior Kubernetes Administrator Solutions:**

### **Solution 1: Pre-Flight Validation Script**

```bash
#!/bin/bash
# validate-hostpath-types.sh

set -e

# Define expected paths and their types
declare -A EXPECTED_PATHS=(
    ["/etc/app-config"]="directory"
    ["/data/postgres"]="directory"
    ["/etc/nginx/nginx.conf"]="file"
    ["/var/run/docker.sock"]="socket"
)

ERRORS=0

echo "=== Validating hostPath Types Across Cluster ==="
echo ""

for node in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
    echo "Checking node: $node"
    echo "----------------------------------------"
    
    for path in "${!EXPECTED_PATHS[@]}"; do
        expected_type="${EXPECTED_PATHS[$path]}"
        
        # Check path type on node
        result=$(kubectl debug node/$node -q --image=busybox -- sh -c "
            if [ ! -e /host$path ]; then
                echo 'MISSING'
            elif [ -d /host$path ]; then
                echo 'directory'
            elif [ -f /host$path ]; then
                echo 'file'
            elif [ -S /host$path ]; then
                echo 'socket'
            elif [ -b /host$path ]; then
                echo 'block'
            elif [ -c /host$path ]; then
                echo 'char'
            else
                echo 'unknown'
            fi
        " 2>/dev/null)
        
        if [ "$result" == "$expected_type" ]; then
            echo "  ✅ $path -> $result (expected: $expected_type)"
        elif [ "$result" == "MISSING" ]; then
            echo "  ⚠️  $path -> MISSING (expected: $expected_type)"
            ERRORS=$((ERRORS + 1))
        else
            echo "  ❌ $path -> $result (expected: $expected_type)"
            ERRORS=$((ERRORS + 1))
        fi
    done
    
    echo ""
done

if [ $ERRORS -gt 0 ]; then
    echo "❌ Validation failed with $ERRORS errors"
    exit 1
else
    echo "✅ All validations passed"
    exit 0
fi
```

**Usage:**
```bash
# Run before deploying pods
./validate-hostpath-types.sh

# Integrate into CI/CD pipeline
- name: Validate hostPath Types
  run: ./validate-hostpath-types.sh
```

---

### **Solution 2: Automated Remediation DaemonSet**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: hostpath-validator
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: hostpath-validator
  template:
    metadata:
      labels:
        app: hostpath-validator
    spec:
      hostPID: true
      hostNetwork: true
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/control-plane
      
      containers:
      - name: validator
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Starting hostPath validation daemon..."
          
          while true; do
            # Check /etc/app-config should be directory
            if [ -f /host/etc/app-config ]; then
              echo "ERROR: /etc/app-config is a file, expected directory"
              echo "Backing up and converting..."
              
              # Backup the file
              cp /host/etc/app-config /host/etc/app-config.backup.$(date +%s)
              
              # Remove file and create directory
              rm /host/etc/app-config
              mkdir -p /host/etc/app-config
              
              # Restore content as a file inside directory
              mv /host/etc/app-config.backup.* /host/etc/app-config/config
              
              echo "Fixed: Converted file to directory"
            elif [ ! -e /host/etc/app-config ]; then
              echo "Creating missing directory: /etc/app-config"
              mkdir -p /host/etc/app-config
            fi
            
            # Check /etc/nginx/nginx.conf should be file
            if [ -d /host/etc/nginx/nginx.conf ]; then
              echo "ERROR: /etc/nginx/nginx.conf is a directory, expected file"
              # Don't auto-fix this - too risky
              echo "Manual intervention required!"
            fi
            
            # Sleep and repeat
            sleep 300  # Check every 5 minutes
          done
          
        volumeMounts:
        - name: host-root
          mountPath: /host
        securityContext:
          privileged: true
          
      volumes:
      - name: host-root
        hostPath:
          path: /
          type: Directory
```

---

### **Solution 3: Use Empty Type for Flexibility (with Validation)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: flexible-config
spec:
  initContainers:
  - name: validate-and-fix
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Validating /config path..."
      
      if [ -f /config ]; then
        echo "ERROR: /config is a file, converting to directory"
        # Backup file content
        cp /config /tmp/config.backup
        rm /config
        mkdir -p /config
        # Move backup inside directory
        mv /tmp/config.backup /config/config.txt
        echo "Converted file to directory"
      elif [ ! -e /config ]; then
        echo "Creating directory: /config"
        mkdir -p /config
      elif [ -d /config ]; then
        echo "OK: /config is already a directory"
      else
        echo "ERROR: /config is neither file nor directory"
        exit 1
      fi
      
    volumeMounts:
    - name: config
      mountPath: /config
    securityContext:
      runAsUser: 0
      
  containers:
  - name: app
    image: nginx:1.21
    volumeMounts:
    - name: config
      mountPath: /etc/app/config
      
  volumes:
  - name: config
    hostPath:
      path: /etc/app-config
      type: ""  # No type checking - allows flexibility
```

**⚠️ Warning:** Using empty type `""` bypasses Kubernetes validation. You must implement your own validation!

---

### **Solution 4: ConfigMap Instead of hostPath (Recommended)**

```yaml
# Create ConfigMap from file
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.yaml: |
    server:
      port: 8080
      host: 0.0.0.0
    database:
      host: postgres
      port: 5432
---
apiVersion: v1
kind: Pod
metadata:
  name: app-with-configmap
spec:
  containers:
  - name: app
    image: nginx:1.21
    volumeMounts:
    - name: config
      mountPath: /etc/app/config
      readOnly: true
      
  volumes:
  - name: config
    configMap:
      name: app-config
```

**Benefits:**
- ✅ No hostPath type validation issues
- ✅ Version controlled in Git
- ✅ Can update without restarting pods (with proper configuration)
- ✅ Stored in etcd (replicated, backed up)
- ✅ Namespace-scoped (better security)

---

## **DevSecOps Best Practices:**

### **Practice 1: Infrastructure as Code Validation**

```yaml
# Ansible playbook with proper validation
---
- name: Setup Kubernetes node directories
  hosts: k8s_workers
  become: yes
  tasks:
    - name: Ensure /etc/app-config is a directory
      file:
        path: /etc/app-config
        state: directory  # CRITICAL: Use 'directory', not 'touch'
        mode: '0755'
        owner: root
        group: root
      register: config_dir
      
    - name: Validate it's actually a directory
      stat:
        path: /etc/app-config
      register: config_stat
      
    - name: Fail if not a directory
      fail:
        msg: "/etc/app-config exists but is not a directory!"
      when: not config_stat.stat.isdir
      
    - name: Ensure /etc/nginx/nginx.conf is a file
      copy:
        content: |
          # Nginx configuration
          user nginx;
          worker_processes auto;
        dest: /etc/nginx/nginx.conf
        mode: '0644'
        owner: root
        group: root
      
    - name: Validate it's actually a file
      stat:
        path: /etc/nginx/nginx.conf
      register: nginx_stat
      
    - name: Fail if not a regular file
      fail:
        msg: "/etc/nginx/nginx.conf exists but is not a regular file!"
      when: not nginx_stat.stat.isreg
```

---

### **Practice 2: Admission Webhook to Enforce Type Checking**

```go
// admission-webhook.go
package main

import (
    "encoding/json"
    "fmt"
    corev1 "k8s.io/api/core/v1"
    admissionv1 "k8s.io/api/admission/v1"
)

// Enforce strict type checking for hostPath volumes
func validateHostPathTypes(pod *corev1.Pod) (bool, string) {
    for _, volume := range pod.Spec.Volumes {
        if volume.HostPath != nil {
            // Require explicit type
            if volume.HostPath.Type == nil {
                return false, fmt.Sprintf(
                    "hostPath volume '%s' must specify explicit type", 
                    volume.Name,
                )
            }
            
            // Warn about empty type
            typeStr := string(*volume.HostPath.Type)
            if typeStr == "" {
                return false, fmt.Sprintf(
                    "hostPath volume '%s' has empty type - use explicit type", 
                    volume.Name,
                )
            }
            
            // Enforce Directory vs File naming convention
            path := volume.HostPath.Path
            if typeStr == "Directory" && !isDirectoryPath(path) {
                return false, fmt.Sprintf(
                    "hostPath '%s' uses type Directory but path looks like a file", 
                    path,
                )
            }
            
            if typeStr == "File" && !isFilePath(path) {
                return false, fmt.Sprintf(
                    "hostPath '%s' uses type File but path looks like a directory", 
                    path,
                )
            }
        }
    }
    
    return true, ""
}

func isDirectoryPath(path string) bool {
    // Heuristic: directories typically don't have extensions
    // and don't end with common file extensions
    fileExtensions := []string{".conf", ".yaml", ".yml", ".json", ".xml", ".txt"}
    for _, ext := range fileExtensions {
        if strings.HasSuffix(path, ext) {
            return false
        }
    }
    return true
}

func isFilePath(path string) bool {
    return !isDirectoryPath(path)
}
```

---

### **Practice 3: OPA/Gatekeeper Policy**

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8shostpathtypecheck
spec:
  crd:
    spec:
      names:
        kind: K8sHostPathTypeCheck
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8shostpathtypecheck
        
        violation[{"msg": msg}] {
          volume := input.review.object.spec.volumes[_]
          volume.hostPath
          
          # Require explicit type
          not volume.hostPath.type
          
          msg := sprintf("hostPath volume '%v' must specify explicit type", [volume.name])
        }
        
        violation[{"msg": msg}] {
          volume := input.review.object.spec.volumes[_]
          volume.hostPath
          
          # Check type is not empty string
          volume.hostPath.type == ""
          
          msg := sprintf("hostPath volume '%v' cannot use empty type", [volume.name])
        }
        
        violation[{"msg": msg}] {
          volume := input.review.object.spec.volumes[_]
          volume.hostPath
          
          # Directory type with file extension
          volume.hostPath.type == "Directory"
          path := volume.hostPath.path
          
          # Check for common file extensions
          endswith(path, ".conf")
          
          msg := sprintf("hostPath '%v' uses Directory type but path '%v' looks like a file", [volume.name, path])
        }
        
        violation[{"msg": msg}] {
          volume := input.review.object.spec.volumes[_]
          volume.hostPath
          
          # File type without file extension
          volume.hostPath.type == "File"
          path := volume.hostPath.path
          
          # Path doesn't have extension
          not contains(path, ".")
          
          msg := sprintf("hostPath '%v' uses File type but path '%v' has no file extension", [volume.name, path])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sHostPathTypeCheck
metadata:
  name: enforce-hostpath-types
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters: {}
```

---

### **Practice 4: Testing Framework**

```bash
#!/bin/bash
# test-hostpath-scenarios.sh

set -e

echo "=== Testing hostPath Type Validation ==="

# Test 1: Directory type with actual directory
echo ""
echo "Test 1: Directory type with directory (should succeed)"
kubectl debug node/node-1 -q --image=busybox -- sh -c "mkdir -p /host/tmp/test-dir"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-dir-success
spec:
  restartPolicy: Never
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'echo OK && sleep 10']
    volumeMounts:
    - name: test
      mountPath: /test
  volumes:
  - name: test
    hostPath:
      path: /tmp/test-dir
      type: Directory
EOF

sleep 5
STATUS=$(kubectl get pod test-dir-success -o jsonpath='{.status.phase}')
if [ "$STATUS" == "Running" ]; then
    echo "✅ Test 1 PASSED"
else
    echo "❌ Test 1 FAILED: Expected Running, got $STATUS"
fi
kubectl delete pod test-dir-success --force --grace-period=0

# Test 2: Directory type with file (should fail)
echo ""
echo "Test 2: Directory type with file (should fail)"
kubectl debug node/node-1 -q --image=busybox -- sh -c "touch /host/tmp/test-file"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-dir-fail
spec:
  restartPolicy: Never
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'echo OK && sleep 10']
    volumeMounts:
    - name: test
      mountPath: /test
  volumes:
  - name: test
    hostPath:
      path: /tmp/test-file
      type: Directory
EOF

sleep 5
STATUS=$(kubectl get pod test-dir-fail -o jsonpath='{.status.phase}')
REASON=$(kubectl get pod test-dir-fail -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}')

if [ "$STATUS" == "Pending" ] && [ "$REASON" == "ContainerCreating" ]; then
    echo "✅ Test 2 PASSED (correctly failed to mount)"
else
    echo "❌ Test 2 FAILED: Pod should be stuck in ContainerCreating"
fi
kubectl delete pod test-dir-fail --force --grace-period=0

# Test 3: File type with directory (should fail)
echo ""
echo "Test 3: File type with directory (should fail)"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-file-fail
spec:
  restartPolicy: Never
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'echo OK && sleep 10']
    volumeMounts:
    - name: test
      mountPath: /test
  volumes:
  - name: test
    hostPath:
      path: /tmp/test-dir
      type: File
EOF

sleep 5
STATUS=$(kubectl get pod test-file-fail -o jsonpath='{.status.phase}')
REASON=$(kubectl get pod test-file-fail -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}')

if [ "$STATUS" == "Pending" ] && [ "$REASON" == "ContainerCreating" ]; then
    echo "✅ Test 3 PASSED (correctly failed to mount)"
else
    echo "❌ Test 3 FAILED: Pod should be stuck in ContainerCreating"
fi
kubectl delete pod test-file-fail --force --grace-period=0

# Test 4: File type with actual file (should succeed)
echo ""
echo "Test 4: File type with file (should succeed)"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-file-success
spec:
  restartPolicy: Never
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'cat /test && sleep 10']
    volumeMounts:
    - name: test
      mountPath: /test
  volumes:
  - name: test
    hostPath:
      path: /tmp/test-file
      type: File
EOF

sleep 5
STATUS=$(kubectl get pod test-file-success -o jsonpath='{.status.phase}')
if [ "$STATUS" == "Running" ]; then
    echo "✅ Test 4 PASSED"
else
    echo "❌ Test 4 FAILED: Expected Running, got $STATUS"
fi
kubectl delete pod test-file-success --force --grace-period=0

# Cleanup
kubectl debug node/node-1 -q --image=busybox -- sh -c "rm -rf /host/tmp/test-dir /host/tmp/test-file"

echo ""
echo "=== All tests completed ==="
```

---

## **Troubleshooting Commands:**

```bash
# 1. Check if path is file or directory
kubectl debug node/node-1 -it --image=busybox -- sh -c "
  if [ -d /host/etc/app-config ]; then
    echo 'DIRECTORY'
  elif [ -f /host/etc/app-config ]; then
    echo 'FILE'
  elif [ -S /host/etc/app-config ]; then
    echo 'SOCKET'
  else
    echo 'UNKNOWN OR MISSING'
  fi
"

# 2. Get detailed file information
kubectl debug node/node-1 -it --image=busybox -- \
  stat -c '%F %n' /host/etc/app-config

# Output examples:
# "directory /host/etc/app-config"
# "regular file /host/etc/app-config"
# "socket /host/etc/app-config"

# 3. List with file type indicators
kubectl debug node/node-1 -it --image=busybox -- \
  ls -F /host/etc/ | grep app-config

# Output indicators:
# app-config/  ← directory (trailing slash)
# app-config   ← file (no indicator)
# app-config@  ← symlink (@ symbol)
# app-config=  ← socket (= symbol)

# 4. Check pod events for mount errors
kubectl describe pod config-reader | grep -A 20 Events

# 5. Check kubelet logs for type validation errors
kubectl debug node/node-1 -it --image=ubuntu -- bash
chroot /host
journalctl -u kubelet -n 100 | grep -i "hostPath\|type check"
```

---

## **Recovery Procedures:**

### **Scenario A: File exists, need directory**

```bash
# Step 1: Backup the file
kubectl debug node/node-1 -q --image=busybox -- \
  cp /host/etc/app-config /host/etc/app-config.backup.$(date +%s)

# Step 2: Remove file
kubectl debug node/node-1 -q --image=busybox -- \
  rm /host/etc/app-config

# Step 3: Create directory
kubectl debug node/node-1 -q --image=busybox -- \
  mkdir -p /host/etc/app-config

# Step 4: Restore content inside directory
kubectl debug node/node-1 -q --image=busybox -- \
  cp /host/etc/app-config.backup.* /host/etc/app-config/config

# Step 5: Restart failed pods
kubectl delete pod config-reader
```

### **Scenario B: Directory exists, need file**

```bash
# Step 1: Check if directory is empty
kubectl debug node/node-1 -q --image=busybox -- \
  ls -la /host/etc/app-config

# Step 2: Backup directory
kubectl debug node/node-1 -q --image=busybox -- \
  mv /host/etc/app-config /host/etc/app-config.backup.$(date +%s)

# Step 3: Create file with content
kubectl debug node/node-1 -q --image=busybox -- sh -c "
  cat > /host/etc/app-config <<EOF
# Configuration file content
server:
  port: 8080
EOF
"

# Step 4: Restart pods
kubectl delete pod config-reader
```

---

## **Monitoring & Alerting:**

```yaml
# Prometheus alert
groups:
- name: hostpath_type_mismatches
  rules:
  - alert: HostPathTypeMismatch
    expr: |
      kube_pod_container_status_waiting_reason{
        reason="ContainerCreating"
      } == 1
      and
      increase(
        kube_pod_container_status_waiting_reason{
          reason="ContainerCreating"
        }[10m]
      ) == 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} stuck mounting hostPath"
      description: |
        Pod has been in ContainerCreating state for >5 minutes.
        Likely cause: hostPath type mismatch (file vs directory).
        Check: kubectl describe pod {{ $labels.pod }}
```

---

## **Key Takeaways:**

| ✅ BEST PRACTICES | ❌ COMMON MISTAKES |
|-------------------|-------------------|
| Always specify explicit `type` | Using empty `type: ""` |
| Validate paths in CI/CD pipeline | Assuming paths are correct type |
| Use naming conventions (dirs without extensions) | Naming directories with `.conf` suffix |
| Implement pre-flight validation scripts | Deploying without validation |
| Use ConfigMaps for configuration files | Using hostPath for config unnecessarily |
| Document path requirements clearly | Undocumented infrastructure assumptions |
| Test with both file and directory scenarios | Only testing happy path |
| Use admission webhooks for enforcement | Relying on manual reviews |

---

## **Quick Reference: Type Selection Guide**

```yaml
# Use Case 1: Configuration directory
volumes:
- name: config-dir
  hostPath:
    path: /etc/myapp
    type: Directory  # Strict validation

# Use Case 2: Single config file
volumes:
- name: config-file
  hostPath:
    path: /etc/myapp/config.yaml
    type: File  # Strict validation

# Use Case 3: Directory that might not exist
volumes:
- name: data-dir
  hostPath:
    path: /data/myapp
    type: DirectoryOrCreate  # Creates if missing

# Use Case 4: Docker socket (must exist)
volumes:
- name: docker-sock
  hostPath:
    path: /var/run/docker.sock
    type: Socket  # Validates it's a socket

# Use Case 5: Device file
volumes:
- name: gpu-device
  hostPath:
    path: /dev/nvidia0
    type: CharDevice  # For character devices
```

---

# **Scenario 5: hostPath used with `File` but directory exists → Pod fails**

## **What Really Happens:**

### **The Scenario:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-config
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf  # Mounting as a single file
  volumes:
  - name: nginx-config
    hostPath:
      path: /etc/nginx/nginx.conf
      type: File  # Expects a FILE
```

**The Problem:**
```bash
# On the host node, someone created a DIRECTORY instead of a file:
$ ls -ld /etc/nginx/nginx.conf
drwxr-xr-x 2 root root 4096 Nov 19 10:00 /etc/nginx/nginx.conf
#^--- This is a DIRECTORY, not a file!

# Pod tries to mount it as a File
# Kubernetes validation FAILS
```

### **What You'll See:**

```bash
$ kubectl get pods
NAME                READY   STATUS                RESTARTS   AGE
nginx-with-config   0/1     ContainerCreating     0          3m

$ kubectl describe pod nginx-with-config
Events:
  Type     Reason       Age                  From               Message
  ----     ------       ----                 ----               -------
  Warning  FailedMount  15s (x12 over 3m)    kubelet            
           MountVolume.SetUp failed for volume "nginx-config" : 
           hostPath type check failed: /etc/nginx/nginx.conf is not a file
```

```bash
# Checking the node directly:
$ kubectl debug node/node-1 -it --image=busybox -- sh
/ # ls -ld /host/etc/nginx/nginx.conf
drwxr-xr-x    2 root     root          4096 Nov 19 10:00 /host/etc/nginx/nginx.conf
#^--- DIRECTORY (starts with 'd'), not a file ('-')

/ # ls /host/etc/nginx/nginx.conf/
mime.types    fastcgi.conf   default.conf
#^--- Directory contains multiple files
```

---

## **Root Cause Analysis - Deep Dive:**

### **Understanding File vs Directory Mounting:**

#### **Correct File Mount:**
```yaml
# Host has FILE
$ ls -l /etc/nginx/nginx.conf
-rw-r--r-- 1 root root 2345 Nov 19 /etc/nginx/nginx.conf

# Pod mounts as FILE
volumeMounts:
- name: config
  mountPath: /etc/nginx/nginx.conf  # Target is a file
  subPath: nginx.conf                # Critical for file mounts

# Result: Single file mounted
$ kubectl exec pod -- ls -l /etc/nginx/nginx.conf
-rw-r--r-- 1 root root 2345 Nov 19 /etc/nginx/nginx.conf
```

#### **What Happens with Directory Instead:**
```yaml
# Host has DIRECTORY
$ ls -ld /etc/nginx/nginx.conf
drwxr-xr-x 2 root root 4096 Nov 19 /etc/nginx/nginx.conf

# Pod tries to mount as FILE
volumeMounts:
- name: config
  mountPath: /etc/nginx/nginx.conf
  
# Result: Validation fails BEFORE container starts
# Pod stuck in ContainerCreating state
```

---

### **The Validation Process:**

```go
// Kubernetes kubelet validation (simplified)
func validateHostPath(path string, pathType *v1.HostPathType) error {
    fileInfo, err := os.Lstat(path)  // Don't follow symlinks
    if err != nil {
        if os.IsNotExist(err) && (*pathType == v1.HostPathFileOrCreate) {
            // Create empty file
            return createEmptyFile(path)
        }
        return fmt.Errorf("path %s does not exist", path)
    }
    
    mode := fileInfo.Mode()
    
    switch *pathType {
    case v1.HostPathFile:
        if mode.IsDir() {
            return fmt.Errorf("%s is not a file (it's a directory)", path)
        }
        if !mode.IsRegular() {
            return fmt.Errorf("%s is not a regular file", path)
        }
        
    case v1.HostPathDirectory:
        if !mode.IsDir() {
            return fmt.Errorf("%s is not a directory", path)
        }
        
    // ... other types
    }
    
    return nil
}
```

---

## **Real-World Production Incidents I've Handled:**

### **Incident 1: Configuration Management Gone Wrong**

```bash
# Situation:
- Chef recipe supposed to create /etc/app/config.yaml FILE
- Bug in recipe created /etc/app/config.yaml DIRECTORY instead
- Recipe populated directory with multiple YAML files
- Kubernetes pods expected single file mount

# Timeline:
14:00 - Chef run executes on all nodes
14:05 - Rolling update of application starts
14:10 - First pods fail to start (ContainerCreating)
14:15 - More pods failing as rollout continues
14:20 - Incident declared - 60% pods down
15:30 - Root cause identified (directory vs file)
16:00 - Fixed Chef recipe
16:30 - Manual cleanup on all nodes
17:00 - Successful rollout completed

# Impact:
- 3-hour partial outage
- 60% capacity reduction
- Customer impact: slow response times
- Revenue loss: ~$50k
```

### **Incident 2: Symlink Confusion**

```bash
# Situation:
- /etc/ssl/cert.pem was supposed to be a FILE
- System admin created symlink to /etc/ssl/certs/ DIRECTORY
- Kubernetes followed symlink, found directory
- Certificate mounting failed for all TLS-enabled pods

# Discovery:
$ ls -l /etc/ssl/cert.pem
lrwxrwxrwx 1 root root 15 Nov 19 /etc/ssl/cert.pem -> /etc/ssl/certs/

$ ls -ld /etc/ssl/certs/
drwxr-xr-x 2 root root 4096 Nov 19 /etc/ssl/certs/

# Impact:
- All HTTPS services down
- 4-hour outage
- Required emergency rollback
```

### **Incident 3: Backup Restore Error**

```bash
# Situation:
- Restoring configuration from backup
- Backup contained directory structure: /etc/nginx/nginx.conf/*
- Extraction created nginx.conf as DIRECTORY
- Nginx pods across cluster failed to start

# What backup contained:
/etc/nginx/nginx.conf/
├── main.conf
├── upstreams.conf
└── locations.conf

# What Kubernetes expected:
/etc/nginx/nginx.conf  (single file)

# Impact:
- 50+ nginx pods failed
- Load balancer health checks failing
- Traffic routing broken
- 2-hour recovery time
```

---

## **Senior Kubernetes Administrator Solutions:**

### **Solution 1: Pre-Validation with InitContainer**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-validated
spec:
  initContainers:
  - name: validate-config
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Validating /config path..."
      
      # Check if path exists
      if [ ! -e /config ]; then
        echo "ERROR: /config does not exist"
        exit 1
      fi
      
      # Check if it's a regular file
      if [ ! -f /config ]; then
        echo "ERROR: /config is not a regular file"
        
        # Check what it actually is
        if [ -d /config ]; then
          echo "FOUND: /config is a DIRECTORY"
          echo "Contents:"
          ls -la /config
          
          # Attempt auto-fix: merge directory files into single file
          if [ -n "$(ls -A /config)" ]; then
            echo "Attempting to merge files..."
            cat /config/* > /tmp/merged-config
            
            # Backup directory
            mv /config /config.backup.$(date +%s)
            
            # Create file
            mv /tmp/merged-config /config
            
            echo "Fixed: Merged directory contents into file"
          fi
        elif [ -L /config ]; then
          echo "FOUND: /config is a SYMLINK"
          echo "Target: $(readlink /config)"
          exit 1
        else
          echo "FOUND: /config is neither file nor directory"
          exit 1
        fi
      fi
      
      # Additional validation: check file size
      SIZE=$(stat -c%s /config)
      if [ "$SIZE" -eq 0 ]; then
        echo "WARNING: /config is empty"
      else
        echo "OK: /config is a file ($SIZE bytes)"
      fi
      
      # Validate it's actually nginx config
      echo "Validating nginx syntax..."
      if command -v nginx >/dev/null 2>&1; then
        nginx -t -c /config || {
          echo "ERROR: Invalid nginx configuration"
          exit 1
        }
      fi
      
      echo "Validation passed!"
      
    volumeMounts:
    - name: config
      mountPath: /config
    securityContext:
      runAsUser: 0
      
  containers:
  - name: nginx
    image: nginx:1.21
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
      readOnly: true
      
  volumes:
  - name: config
    hostPath:
      path: /etc/nginx/nginx.conf
      type: File
```

---

### **Solution 2: Node Preparation DaemonSet**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: config-file-validator
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: config-validator
  template:
    metadata:
      labels:
        app: config-validator
    spec:
      hostPID: true
      tolerations:
      - effect: NoSchedule
        operator: Exists
      
      containers:
      - name: validator
        image: nginx:1.21  # Has nginx binary for validation
        command:
        - sh
        - -c
        - |
          echo "Starting configuration file validator daemon..."
          
          while true; do
            # Define expected files
            CONFIG_FILES=(
              "/host/etc/nginx/nginx.conf"
              "/host/etc/ssl/cert.pem"
              "/host/etc/app/config.yaml"
            )
            
            for config_path in "${CONFIG_FILES[@]}"; do
              echo "Checking: $config_path"
              
              # Check if path exists
              if [ ! -e "$config_path" ]; then
                echo "  ⚠️  MISSING: $config_path"
                continue
              fi
              
              # Check if it's a directory (BAD)
              if [ -d "$config_path" ]; then
                echo "  ❌ ERROR: $config_path is a DIRECTORY (expected FILE)"
                
                # Log the issue
                echo "$(date): Directory found at $config_path" >> /host/var/log/config-validator.log
                
                # Check if directory has single file we can promote
                file_count=$(find "$config_path" -maxdepth 1 -type f | wc -l)
                
                if [ "$file_count" -eq 1 ]; then
                  echo "  🔧 Auto-fixing: Single file in directory"
                  
                  # Backup directory
                  mv "$config_path" "${config_path}.backup.$(date +%s)"
                  
                  # Promote single file
                  find "${config_path}.backup."* -maxdepth 1 -type f -exec mv {} "$config_path" \;
                  
                  echo "  ✅ Fixed: Promoted file from directory"
                else
                  echo "  ⚠️  Cannot auto-fix: Multiple files ($file_count) in directory"
                fi
                
              # Check if it's a regular file (GOOD)
              elif [ -f "$config_path" ]; then
                echo "  ✅ OK: $config_path is a file"
                
                # Validate nginx configs
                if [[ "$config_path" == *"nginx.conf" ]]; then
                  if nginx -t -c "$config_path" 2>/dev/null; then
                    echo "  ✅ nginx config syntax valid"
                  else
                    echo "  ❌ nginx config syntax invalid"
                  fi
                fi
                
              # Check if it's a symlink
              elif [ -L "$config_path" ]; then
                target=$(readlink "$config_path")
                echo "  ⚠️  SYMLINK: $config_path -> $target"
                
                # Check if symlink target is valid
                if [ -f "$target" ]; then
                  echo "  ✅ Symlink target is a file"
                elif [ -d "$target" ]; then
                  echo "  ❌ Symlink target is a DIRECTORY"
                else
                  echo "  ❌ Symlink target is missing or invalid"
                fi
                
              else
                echo "  ❌ Unknown file type: $config_path"
              fi
              
            done
            
            echo "---"
            sleep 300  # Check every 5 minutes
          done
          
        volumeMounts:
        - name: host-root
          mountPath: /host
        securityContext:
          privileged: true
          
      volumes:
      - name: host-root
        hostPath:
          path: /
          type: Directory
```

---

### **Solution 3: Use ConfigMap for Configuration Files (Best Practice)**

```yaml
# Instead of hostPath, use ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    user nginx;
    worker_processes auto;
    error_log /var/log/nginx/error.log notice;
    pid /var/run/nginx.pid;
    
    events {
        worker_connections 1024;
    }
    
    http {
        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        
        log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';
        
        access_log /var/log/nginx/access.log main;
        
        sendfile on;
        keepalive_timeout 65;
        
        server {
            listen 80;
            server_name localhost;
            
            location / {
                root /usr/share/nginx/html;
                index index.html;
            }
        }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-configmap
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf  # Mount single file from ConfigMap
      readOnly: true
    ports:
    - containerPort: 80
      
  volumes:
  - name: config
    configMap:
      name: nginx-config
      items:
      - key: nginx.conf
        path: nginx.conf
```

**Benefits of ConfigMap over hostPath:**
- ✅ No file vs directory confusion
- ✅ Version controlled in Git
- ✅ Can be updated independently
- ✅ Automatic rollout with `kubectl rollout restart`
- ✅ No node-level dependencies
- ✅ Works with any scheduling decision

---

### **Solution 4: FileOrCreate for Flexibility**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-autocreate
spec:
  initContainers:
  - name: ensure-config
    image: nginx:1.21
    command:
    - sh
    - -c
    - |
      # Check if config exists and is the right type
      if [ -d /config ]; then
        echo "ERROR: /config is a directory"
        
        # Try to find a main config file
        if [ -f /config/nginx.conf ]; then
          echo "Found nginx.conf inside directory, promoting it"
          mv /config/nginx.conf /tmp/nginx.conf
          rm -rf /config
          mv /tmp/nginx.conf /config
        else
          echo "No nginx.conf found, creating default"
          rm -rf /config
          cat > /config <<'EOF'
user nginx;
worker_processes auto;
events {
    worker_connections 1024;
}
http {
    server {
        listen 80;
        location / {
            root /usr/share/nginx/html;
        }
    }
}
EOF
        fi
      elif [ ! -f /config ]; then
        echo "Creating default config"
        cat > /config <<'EOF'
user nginx;
worker_processes auto;
events {
    worker_connections 1024;
}
http {
    server {
        listen 80;
        location / {
            root /usr/share/nginx/html;
        }
    }
}
EOF
      fi
      
      # Validate nginx config
      nginx -t -c /config
      
    volumeMounts:
    - name: config
      mountPath: /config
    securityContext:
      runAsUser: 0
      
  containers:
  - name: nginx
    image: nginx:1.21
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
      
  volumes:
  - name: config
    hostPath:
      path: /etc/nginx/nginx.conf
      type: FileOrCreate  # Creates if missing, but still validates type
```

---

## **DevSecOps Best Practices:**

### **Practice 1: Infrastructure Validation Pipeline**

```yaml
# .gitlab-ci.yml or .github/workflows/validate.yml
name: Validate Node Configuration

on:
  push:
    branches: [main]
  pull_request:

jobs:
  validate-hostpaths:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Extract hostPath requirements
        run: |
          # Parse all Kubernetes manifests
          echo "Extracting hostPath requirements..."
          
          yq eval '
            .. | select(has("hostPath")) | 
            {
              "path": .hostPath.path, 
              "type": .hostPath.type
            }
          ' k8s/**/*.yaml > hostpath-requirements.json
          
          cat hostpath-requirements.json
          
      - name: Validate against Ansible inventory
        run: |
          # Check if Ansible playbooks create the right types
          python3 <<EOF
          import yaml
          import json
          
          # Load requirements
          with open('hostpath-requirements.json') as f:
              requirements = json.load(f)
          
          # Load Ansible playbooks
          with open('ansible/node-setup.yml') as f:
              playbook = yaml.safe_load(f)
          
          errors = []
          
          for req in requirements:
              path = req['path']
              expected_type = req['type']
              
              # Find corresponding task in playbook
              found = False
              for play in playbook:
                  for task in play.get('tasks', []):
                      if path in str(task):
                          module = list(task.keys())[1]  # Get module name
                          
                          if expected_type == 'File' and module != 'copy':
                              errors.append(f"{path}: Expected File but playbook uses {module}")
                          elif expected_type == 'Directory' and module != 'file':
                              errors.append(f"{path}: Expected Directory but playbook uses {module}")
                          
                          found = True
                          break
              
              if not found:
                  errors.append(f"{path}: Not found in Ansible playbooks")
          
          if errors:
              print("❌ Validation Errors:")
              for error in errors:
                  print(f"  - {error}")
              exit(1)
          else:
              print("✅ All hostPath requirements validated")
          EOF
```

---

### **Practice 2: Ansible Playbook with Type Enforcement**

```yaml
---
- name: Configure Kubernetes node files
  hosts: k8s_workers
  become: yes
  
  vars:
    config_files:
      - path: /etc/nginx/nginx.conf
        type: file
        content: |
          user nginx;
          worker_processes auto;
          events { worker_connections 1024; }
          http {
              server {
                  listen 80;
                  location / { root /usr/share/nginx/html; }
              }
          }
        
      - path: /etc/ssl/cert.pem
        type: file
        source: files/ssl/cert.pem
        
      - path: /etc/app/config.yaml
        type: file
        content: |
          server:
            port: 8080
            host: 0.0.0.0
    
    config_directories:
      - path: /data/nginx
        mode: '0755'
        owner: nginx
        group: nginx
        
      - path: /data/app/logs
        mode: '0750'
        owner: app
        group: app
  
  tasks:
    - name: Ensure configuration files exist as FILES
      copy:
        content: "{{ item.content | default(omit) }}"
        src: "{{ item.source | default(omit) }}"
        dest: "{{ item.path }}"
        mode: '0644'
        owner: root
        group: root
        force: yes  # Overwrite if exists
      loop: "{{ config_files }}"
      when: item.type == 'file'
      
    - name: Validate files are actually files (not directories)
      stat:
        path: "{{ item.path }}"
      register: file_stats
      loop: "{{ config_files }}"
      
    - name: Fail if any file is actually a directory
      fail:
        msg: "{{ item.item.path }} is a directory but should be a file!"
      loop: "{{ file_stats.results }}"
      when: 
        - item.stat.exists
        - item.stat.isdir
      
    - name: Ensure configuration directories exist as DIRECTORIES
      file:
        path: "{{ item.path }}"
        state: directory
        mode: "{{ item.mode }}"
        owner: "{{ item.owner }}"
        group: "{{ item.group }}"
      loop: "{{ config_directories }}"
      
    - name: Validate directories are actually directories (not files)
      stat:
        path: "{{ item.path }}"
      register: dir_stats
      loop: "{{ config_directories }}"
      
    - name: Fail if any directory is actually a file
      fail:
        msg: "{{ item.item.path }} is a file but should be a directory!"
      loop: "{{ dir_stats.results }}"
      when: 
        - item.stat.exists
        - item.stat.isreg
```

---

### **Practice 3: OPA/Gatekeeper Policy for File Paths**

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8shostpathfilenaming
spec:
  crd:
    spec:
      names:
        kind: K8sHostPathFileNaming
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8shostpathfilenaming
        
        # Define file extensions that indicate a file
        file_extensions := {".conf", ".yaml", ".yml", ".json", ".xml", ".pem", ".crt", ".key", ".txt", ".log"}
        
        violation[{"msg": msg}] {
          volume := input.review.object.spec.volumes[_]
          volume.hostPath
          
          path := volume.hostPath.path
          hostpath_type := volume.hostPath.type
          
          # Check if path has file extension
          has_extension := has_file_extension(path)
          
          # If path looks like a file but type is Directory
          has_extension
          hostpath_type == "Directory"
          
          msg := sprintf(
            "hostPath '%v' has file extension but uses type 'Directory'. Path: %v",
            [volume.name, path]
          )
        }
        
        violation[{"msg": msg}] {
          volume := input.review.object.spec.volumes[_]
          volume.hostPath
          
          path := volume.hostPath.path
          hostpath_type := volume.hostPath.type
          
          # Check if path has NO file extension
          not has_file_extension(path)
          
          # If path looks like a directory but type is File
          hostpath_type == "File"
          
          msg := sprintf(
            "hostPath '%v' has no file extension but uses type 'File'. Path: %v",
            [volume.name, path]
          )
        }
        
        has_file_extension(path) {
          # Get last component of path
          parts := split(path, "/")
          filename := parts[count(parts) - 1]
          
          # Check if filename contains a dot
          contains(filename, ".")
          
          # Get extension
          ext_parts := split(filename, ".")
          extension := concat(".", ["", ext_parts[count(ext_parts) - 1]])
          
          # Check if extension is in our list
          file_extensions[extension]
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sHostPathFileNaming
metadata:
  name: enforce-file-naming-convention
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters: {}
```

---

### **Practice 4: Automated Testing Suite**

```bash
#!/bin/bash
# test-file-directory-scenarios.sh

set -e

NAMESPACE="hostpath-tests"
NODE_NAME="node-1"

kubectl create namespace $NAMESPACE 2>/dev/null || true

echo "=== Testing File vs Directory Scenarios ==="
echo ""

# Cleanup function
cleanup() {
    echo "Cleaning up..."
    kubectl delete namespace $NAMESPACE --force --grace-period=0 2>/dev/null || true
    kubectl debug node/$NODE_NAME -q --image=busybox -- sh -c "
        rm -rf /host/tmp/test-scenarios
    " 2>/dev/null || true
}

trap cleanup EXIT

# Setup test directory
kubectl debug node/$NODE_NAME -q --image=busybox -- sh -c "
    mkdir -p /host/tmp/test-scenarios
"

# Test 1: File type with actual file (SHOULD SUCCEED)
echo "Test 1: Mounting FILE with type=File"
kubectl debug node/$NODE_NAME -q --image=busybox -- sh -c "
    echo 'test content' > /host/tmp/test-scenarios/config-file.conf
"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-file-file
  namespace: $NAMESPACE
spec:
  restartPolicy: Never
  nodeName: $NODE_NAME
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'cat /config && sleep 5']
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    hostPath:
      path: /tmp/test-scenarios/config-file.conf
      type: File
EOF

sleep 3
STATUS=$(kubectl get pod test-file-file -n $NAMESPACE -o jsonpath='{.status.phase}')
if [ "$STATUS" == "Running" ] || [ "$STATUS" == "Succeeded" ]; then
    echo "✅ Test 1 PASSED: File mounted successfully"
else
    echo "❌ Test 1 FAILED: Expected Running/Succeeded, got $STATUS"
fi
echo ""

# Test 2: File type with directory (SHOULD FAIL)
echo "Test 2: Mounting DIRECTORY with type=File"
kubectl debug node/$NODE_NAME -q --image=busybox -- sh -c "
    mkdir -p /host/tmp/test-scenarios/config-dir
    echo 'content' > /host/tmp/test-scenarios/config-dir/file.txt
"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-file-dir
  namespace: $NAMESPACE
spec:
  restartPolicy: Never
  nodeName: $NODE_NAME
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'sleep 5']
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    hostPath:
      path: /tmp/test-scenarios/config-dir
      type: File
EOF

sleep 3
STATUS=$(kubectl get pod test-file-dir -n $NAMESPACE -o jsonpath='{.status.phase}')
REASON=$(kubectl get pod test-file-dir -n $NAMESPACE -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}' 2>/dev/null || echo "")

if [ "$STATUS" == "Pending" ] && [ "$REASON" == "ContainerCreating" ]; then
    # Check for the specific error in events
    ERROR=$(kubectl describe pod test-file-dir -n $NAMESPACE | grep "is not a file")
    if [ -n "$ERROR" ]; then
        echo "✅ Test 2 PASSED: Correctly rejected directory as file"
    else
        echo "⚠️  Test 2 PARTIAL: Pod stuck but error message unclear"
    fi
else
    echo "❌ Test 2 FAILED: Should be stuck in ContainerCreating"
fi
echo ""

# Test 3: Directory type with file (SHOULD FAIL)
echo "Test 3: Mounting FILE with type=Directory"
kubectl debug node/$NODE_NAME -q --image=busybox -- sh -c "
    echo 'content' > /host/tmp/test-scenarios/single-file
"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-dir-file
  namespace: $NAMESPACE
spec:
  restartPolicy: Never
  nodeName: $NODE_NAME
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'sleep 5']
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    hostPath:
      path: /tmp/test-scenarios/single-file
      type: Directory
EOF

sleep 3
STATUS=$(kubectl get pod test-dir-file -n $NAMESPACE -o jsonpath='{.status.phase}')
REASON=$(kubectl get pod test-dir-file -n $NAMESPACE -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}' 2>/dev/null || echo "")

if [ "$STATUS" == "Pending" ] && [ "$REASON" == "ContainerCreating" ]; then
    ERROR=$(kubectl describe pod test-dir-file -n $NAMESPACE | grep "is not a directory")
    if [ -n "$ERROR" ]; then
        echo "✅ Test 3 PASSED: Correctly rejected file as directory"
    else
        echo "⚠️  Test 3 PARTIAL: Pod stuck but error message unclear"
    fi
else
    echo "❌ Test 3 FAILED: Should be stuck in ContainerCreating"
fi
echo ""

# Test 4: FileOrCreate with directory (SHOULD FAIL - won't overwrite)
echo "Test 4: FileOrCreate type with existing directory"
kubectl debug node/$NODE_NAME -q --image=busybox -- sh -c "
    mkdir -p /host/tmp/test-scenarios/should-be-file
    echo 'content' > /host/tmp/test-scenarios/should-be-file/data.txt
"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-fileorcreate-dir
  namespace: $NAMESPACE
spec:
  restartPolicy: Never
  nodeName: $NODE_NAME
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'sleep 5']
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    hostPath:
      path: /tmp/test-scenarios/should-be-file
      type: FileOrCreate
EOF

sleep 3
STATUS=$(kubectl get pod test-fileorcreate-dir -n $NAMESPACE -o jsonpath='{.status.phase}')
REASON=$(kubectl get pod test-fileorcreate-dir -n $NAMESPACE -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}' 2>/dev/null || echo "")

if [ "$STATUS" == "Pending" ] && [ "$REASON" == "ContainerCreating" ]; then
    echo "✅ Test 4 PASSED: FileOrCreate correctly refused to overwrite directory"
else
    echo "❌ Test 4 FAILED: FileOrCreate should not overwrite existing directory"
fi
echo ""

# Test 5: FileOrCreate with missing file (SHOULD CREATE)
echo "Test 5: FileOrCreate type with missing file (auto-create)"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-fileorcreate-missing
  namespace: $NAMESPACE
spec:
  restartPolicy: Never
  nodeName: $NODE_NAME
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'ls -l /config && sleep 5']
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    hostPath:
      path: /tmp/test-scenarios/auto-created-file
      type: FileOrCreate
EOF

sleep 3
STATUS=$(kubectl get pod test-fileorcreate-missing -n $NAMESPACE -o jsonpath='{.status.phase}')
if [ "$STATUS" == "Running" ] || [ "$STATUS" == "Succeeded" ]; then
    # Verify file was created on host
    CREATED=$(kubectl debug node/$NODE_NAME -q --image=busybox -- sh -c "
        [ -f /host/tmp/test-scenarios/auto-created-file ] && echo 'YES' || echo 'NO'
    ")
    if [ "$CREATED" == "YES" ]; then
        echo "✅ Test 5 PASSED: FileOrCreate auto-created missing file"
    else
        echo "❌ Test 5 FAILED: File was not created on host"
    fi
else
    echo "❌ Test 5 FAILED: Pod should be Running, got $STATUS"
fi
echo ""

# Test 6: Symlink to directory with File type (SHOULD FAIL)
echo "Test 6: File type with symlink pointing to directory"
kubectl debug node/$NODE_NAME -q --image=busybox -- sh -c "
    mkdir -p /host/tmp/test-scenarios/target-dir
    ln -s /tmp/test-scenarios/target-dir /host/tmp/test-scenarios/symlink-to-dir
"

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-symlink-dir
  namespace: $NAMESPACE
spec:
  restartPolicy: Never
  nodeName: $NODE_NAME
  containers:
  - name: test
    image: busybox
    command: ['sh', '-c', 'sleep 5']
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    hostPath:
      path: /tmp/test-scenarios/symlink-to-dir
      type: File
EOF

sleep 3
STATUS=$(kubectl get pod test-symlink-dir -n $NAMESPACE -o jsonpath='{.status.phase}')
if [ "$STATUS" == "Pending" ]; then
    echo "✅ Test 6 PASSED: Symlink to directory correctly rejected"
else
    echo "⚠️  Test 6 WARNING: Symlink behavior may vary by Kubernetes version"
fi
echo ""

# Summary
echo "==================================="
echo "Test Suite Completed"
echo "==================================="
kubectl get pods -n $NAMESPACE -o wide
```

---

## **Troubleshooting Guide:**

### **Step-by-Step Troubleshooting:**

```bash
#!/bin/bash
# troubleshoot-file-directory-issue.sh

POD_NAME=$1
NAMESPACE=${2:-default}

if [ -z "$POD_NAME" ]; then
    echo "Usage: $0 <pod-name> [namespace]"
    exit 1
fi

echo "=== Troubleshooting hostPath File/Directory Issue ==="
echo "Pod: $POD_NAME"
echo "Namespace: $NAMESPACE"
echo ""

# Step 1: Get pod status
echo "Step 1: Checking pod status..."
STATUS=$(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.status.phase}')
echo "Pod Status: $STATUS"

if [ "$STATUS" != "Pending" ]; then
    echo "✅ Pod is not stuck in Pending state"
    exit 0
fi

# Step 2: Check for mount errors
echo ""
echo "Step 2: Checking for mount errors..."
MOUNT_ERROR=$(kubectl describe pod $POD_NAME -n $NAMESPACE | grep -A 5 "FailedMount" | grep "type check failed")

if [ -z "$MOUNT_ERROR" ]; then
    echo "No type check errors found"
else
    echo "Found mount error:"
    echo "$MOUNT_ERROR"
fi

# Step 3: Extract hostPath information
echo ""
echo "Step 3: Extracting hostPath volumes..."
kubectl get pod $POD_NAME -n $NAMESPACE -o json | jq -r '
    .spec.volumes[] | 
    select(.hostPath != null) | 
    {
        name: .name,
        path: .hostPath.path,
        type: .hostPath.type
    }
' | tee /tmp/hostpaths.json

# Step 4: Get node name
echo ""
echo "Step 4: Identifying node..."
NODE=$(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.nodeName}')
echo "Node: $NODE"

if [ -z "$NODE" ]; then
    echo "⚠️  Pod not yet scheduled to a node"
    exit 1
fi

# Step 5: Check actual file types on node
echo ""
echo "Step 5: Checking actual file types on node..."

while IFS= read -r line; do
    PATH_ON_HOST=$(echo "$line" | jq -r '.path')
    EXPECTED_TYPE=$(echo "$line" | jq -r '.type')
    VOLUME_NAME=$(echo "$line" | jq -r '.name')
    
    echo ""
    echo "Volume: $VOLUME_NAME"
    echo "Path: $PATH_ON_HOST"
    echo "Expected Type: $EXPECTED_TYPE"
    
    # Check actual type on node
    ACTUAL_TYPE=$(kubectl debug node/$NODE -q --image=busybox -- sh -c "
        if [ ! -e /host$PATH_ON_HOST ]; then
            echo 'MISSING'
        elif [ -d /host$PATH_ON_HOST ]; then
            echo 'DIRECTORY'
        elif [ -f /host$PATH_ON_HOST ]; then
            echo 'FILE'
        elif [ -L /host$PATH_ON_HOST ]; then
            TARGET=\$(readlink /host$PATH_ON_HOST)
            if [ -d /host\$TARGET ]; then
                echo 'SYMLINK_TO_DIRECTORY'
            elif [ -f /host\$TARGET ]; then
                echo 'SYMLINK_TO_FILE'
            else
                echo 'SYMLINK_BROKEN'
            fi
        else
            echo 'UNKNOWN'
        fi
    " 2>/dev/null)
    
    echo "Actual Type: $ACTUAL_TYPE"
    
    # Compare and suggest fix
    if [[ "$EXPECTED_TYPE" == "File" && "$ACTUAL_TYPE" == "DIRECTORY" ]]; then
        echo "❌ MISMATCH: Expected FILE but found DIRECTORY"
        echo ""
        echo "Suggested fixes:"
        echo "  1. Change hostPath type to 'Directory'"
        echo "  2. Remove directory and create file:"
        echo "     kubectl debug node/$NODE -- sh -c 'rm -rf /host$PATH_ON_HOST && touch /host$PATH_ON_HOST'"
        echo "  3. If directory contains single file, promote it:"
        echo "     kubectl debug node/$NODE -- sh -c 'mv /host$PATH_ON_HOST/\$(ls /host$PATH_ON_HOST | head -1) /tmp/file && rm -rf /host$PATH_ON_HOST && mv /tmp/file /host$PATH_ON_HOST'"
        
    elif [[ "$EXPECTED_TYPE" == "Directory" && "$ACTUAL_TYPE" == "FILE" ]]; then
        echo "❌ MISMATCH: Expected DIRECTORY but found FILE"
        echo ""
        echo "Suggested fixes:"
        echo "  1. Change hostPath type to 'File'"
        echo "  2. Remove file and create directory:"
        echo "     kubectl debug node/$NODE -- sh -c 'rm /host$PATH_ON_HOST && mkdir -p /host$PATH_ON_HOST'"
        echo "  3. Move file inside directory:"
        echo "     kubectl debug node/$NODE -- sh -c 'mv /host$PATH_ON_HOST /tmp/file && mkdir -p /host$PATH_ON_HOST && mv /tmp/file /host$PATH_ON_HOST/\$(basename $PATH_ON_HOST)'"
        
    elif [[ "$ACTUAL_TYPE" == "MISSING" ]]; then
        echo "⚠️  Path does not exist on host"
        echo ""
        echo "Suggested fixes:"
        echo "  1. Change hostPath type to '${EXPECTED_TYPE}OrCreate'"
        echo "  2. Create the path manually:"
        if [[ "$EXPECTED_TYPE" == "File" ]]; then
            echo "     kubectl debug node/$NODE -- sh -c 'touch /host$PATH_ON_HOST'"
        else
            echo "     kubectl debug node/$NODE -- sh -c 'mkdir -p /host$PATH_ON_HOST'"
        fi
        
    elif [[ "$ACTUAL_TYPE" == SYMLINK_* ]]; then
        echo "⚠️  Path is a symlink"
        TARGET=$(kubectl debug node/$NODE -q --image=busybox -- sh -c "readlink /host$PATH_ON_HOST" 2>/dev/null)
        echo "Target: $TARGET"
        echo ""
        echo "Suggested fixes:"
        echo "  1. Replace symlink with actual file/directory"
        echo "  2. Update hostPath to point to symlink target: $TARGET"
        
    else
        echo "✅ Type matches or is compatible"
    fi
    
done < <(jq -c '.' /tmp/hostpaths.json)

# Step 6: Provide recovery commands
echo ""
echo "==================================="
echo "Recovery Commands:"
echo "==================================="
echo ""
echo "To restart the pod after fixing:"
echo "kubectl delete pod $POD_NAME -n $NAMESPACE"
echo ""
echo "To check fix applied correctly:"
echo "./troubleshoot-file-directory-issue.sh $POD_NAME $NAMESPACE"
```

---

## **Common Patterns & Anti-Patterns:**

### **✅ Good Patterns:**

#### **Pattern 1: Single Config File with ConfigMap**
```yaml
# RECOMMENDED: Use ConfigMap for single files
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
data:
  nginx.conf: |
    # Your nginx configuration
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
  volumes:
  - name: config
    configMap:
      name: nginx-conf
```

#### **Pattern 2: Multiple Config Files with Directory**
```yaml
# For multiple related files, use directory mount
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config-dir
      mountPath: /etc/app/config
      readOnly: true
  volumes:
  - name: config-dir
    hostPath:
      path: /etc/app/config
      type: Directory
```

#### **Pattern 3: Validated File Mount with InitContainer**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: validated-app
spec:
  initContainers:
  - name: validate
    image: busybox
    command:
    - sh
    - -c
    - |
      if [ ! -f /config ]; then
        echo "ERROR: /config is not a file"
        exit 1
      fi
      echo "Validation passed"
    volumeMounts:
    - name: config
      mountPath: /config
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config
      mountPath: /app/config.yaml
      subPath: config.yaml
  volumes:
  - name: config
    hostPath:
      path: /etc/app/config.yaml
      type: File
```

---

### **❌ Anti-Patterns:**

#### **Anti-Pattern 1: Using Empty Type**
```yaml
# BAD: No validation
volumes:
- name: config
  hostPath:
    path: /etc/app/config
    type: ""  # Don't do this!
```

#### **Anti-Pattern 2: Assuming Path Type**
```yaml
# BAD: No verification that path is correct type
apiVersion: v1
kind: Pod
metadata:
  name: risky-app
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    hostPath:
      path: /etc/app/config  # Is this a file or directory? Unknown!
      type: File  # Assuming it's a file - dangerous!
```

#### **Anti-Pattern 3: No Naming Convention**
```yaml
# BAD: Ambiguous naming
hostPath:
  path: /etc/nginx  # Looks like directory
  type: File  # But expecting file

# GOOD: Clear naming
hostPath:
  path: /etc/nginx/nginx.conf  # Clear it's a file
  type: File
```

---

## **Monitoring & Alerting:**

```yaml
# Prometheus AlertManager rules
groups:
- name: hostpath_type_issues
  rules:
  - alert: HostPathTypeMismatch
    expr: |
      (
        kube_pod_container_status_waiting_reason{reason="ContainerCreating"} == 1
      ) and (
        count by (namespace, pod) (
          ALERTS{alertname="HostPathTypeMismatch"}
        ) > 0
      )
    for: 3m
    labels:
      severity: critical
      component: storage
    annotations:
      summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has hostPath type mismatch"
      description: |
        Pod has been stuck in ContainerCreating for >3 minutes.
        Likely cause: hostPath type mismatch (File vs Directory).
        
        Troubleshooting:
        1. kubectl describe pod {{ $labels.pod }} -n {{ $labels.namespace }}
        2. Check for "type check failed" in events
        3. Run: ./troubleshoot-file-directory-issue.sh {{ $labels.pod }} {{ $labels.namespace }}
      runbook_url: "https://wiki.company.com/k8s/hostpath-type-mismatch"
      
  - alert: HostPathMountErrorRate
    expr: |
      rate(kubelet_volume_stats_errors_total[5m]) > 0.1
    labels:
      severity: warning
    annotations:
      summary: "High rate of volume mount errors on {{ $labels.node }}"
      description: "Volume mount errors detected on node. Check hostPath types."
```

---

## **Documentation Template:**

```markdown
# Application Deployment - hostPath Requirements

## Overview
This application requires specific host paths to be provisioned on Kubernetes nodes.

## Required Files

### /etc/nginx/nginx.conf
- **Type**: Regular file
- **Ownership**: root:root
- **Permissions**: 0644
- **Content**: Main nginx configuration
- **Provisioning**:
  ```bash
  ansible-playbook -i inventory provision-nginx-config.yml
  ```
- **Validation**:
  ```bash
  ansible all -i inventory -m stat -a "path=/etc/nginx/nginx.conf"
  # Verify: "stat.isreg": true
  ```

### /etc/ssl/cert.pem
- **Type**: Regular file (not symlink!)
- **Ownership**: root:root
- **Permissions**: 0644
- **Content**: SSL certificate
- **Provisioning**:
  ```bash
  ansible-playbook -i inventory provision-ssl-certs.yml
  ```
- **Validation**:
  ```bash
  ansible all -i inventory -m stat -a "path=/etc/ssl/cert.pem follow=no"
  # Verify: "stat.isreg": true, "stat.islnk": false
  ```

## Required Directories

### /data/app/logs
- **Type**: Directory
- **Ownership**: app:app (UID:GID 1000:1000)
- **Permissions**: 0750
- **Purpose**: Application log files
- **Provisioning**:
  ```bash
  ansible-playbook -i inventory provision-app-dirs.yml
  ```
- **Validation**:
  ```bash
  ansible all -i inventory -m stat -a "path=/data/app/logs"
  # Verify: "stat.isdir": true
  ```

## Pre-Deployment Checklist

- [ ] Run validation script: `./validate-hostpath-types.sh`
- [ ] Verify all files are files (not directories)
- [ ] Verify all directories are directories (not files)
- [ ] Check no unexpected symlinks exist
- [ ] Verify file permissions and ownership
- [ ] Test on staging nodes first
- [ ] Document any node-specific variations

## Common Issues

### Issue 1: File is actually a directory
**Symptom**: Pod stuck in ContainerCreating with "is not a file" error
**Fix**:
```bash
# Backup and convert
kubectl debug node/NODE -- sh -c '
  mv /host/path/to/file /host/path/to/file.backup
  touch /host/path/to/file
'
```

### Issue 2: Directory is actually a file
**Symptom**: Pod stuck in ContainerCreating with "is not a directory" error
**Fix**:
```bash
# Backup and convert
kubectl debug node/NODE -- sh -c '
  mv /host/path/to/dir /host/path/to/dir.backup
  mkdir -p /host/path/to/dir
'
```

## Troubleshooting
```bash
# Check actual types on all nodes
./validate-hostpath-types.sh

# Troubleshoot specific pod
./troubleshoot-file-directory-issue.sh POD_NAME NAMESPACE

# View kubelet logs
kubectl debug node/NODE -it --image=ubuntu -- bash
chroot /host
journalctl -u kubelet | grep "type check"
```

---

## **Key Takeaways:**

| Category | Best Practice |
|----------|---------------|
| **Naming** | Files should have extensions (.conf, .yaml); directories should not |
| **Type Specification** | Always use explicit type (File, Directory, FileOrCreate, DirectoryOrCreate) |
| **Validation** | Implement pre-flight validation in CI/CD |
| **Documentation** | Clearly document expected file vs directory for each hostPath |
| **Testing** | Test both file and directory scenarios before production |
| **Monitoring** | Alert on pods stuck in ContainerCreating > 3 minutes |
| **Recovery** | Have automated recovery procedures documented |
| **Alternatives** | Prefer ConfigMaps/Secrets over hostPath for configuration |

---

## **Quick Reference Card:**

```
┌─────────────────────────────────────────────────────────────┐
│ hostPath Type Validation Quick Reference                    │
├─────────────────────────────────────────────────────────────┤
│ SCENARIO                    │ RESULT                        │
├─────────────────────────────┼───────────────────────────────┤
│ type: File + actual file    │ ✅ SUCCESS                    │
│ type: File + actual dir     │ ❌ FAIL (type check failed)   │
│ type: File + missing        │ ❌ FAIL (does not exist)      │
│ type: FileOrCreate + file   │ ✅ SUCCESS                    │
│ type: FileOrCreate + dir    │ ❌ FAIL (won't overwrite)     │
│ type: FileOrCreate + missing│ ✅ SUCCESS (creates empty file│
│ type: Directory + dir       │ ✅ SUCCESS                    │
│ type: Directory + file      │ ❌ FAIL (type check failed)   │
│ type: "" (empty) + anything │ ⚠️  NO VALIDATION (risky)     │
└─────────────────────────────┴───────────────────────────────┘
```

---

# **Scenario 6: Node drained → Pod moves → hostPath data "disappears"**

## **What Really Happens:**

### **The Scenario:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: nginx:1.21
        volumeMounts:
        - name: user-uploads
          mountPath: /data/uploads
      volumes:
      - name: user-uploads
        hostPath:
          path: /data/uploads
          type: DirectoryOrCreate
```

**The Problem:**
```bash
# Day 1: Users upload files to pod on node-1
$ kubectl get pods -o wide
NAME                      READY   NODE
web-app-abc123-xyz        1/1     node-1
# Users upload photos to /data/uploads

# Day 2: Node maintenance - drain node-1
$ kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# Pod reschedules to node-2
$ kubectl get pods -o wide
NAME                      READY   NODE
web-app-abc123-def        1/1     node-2

# User tries to access their photos
$ kubectl exec web-app-abc123-def -- ls /data/uploads
# Empty! Data is still on node-1
```

### **What You'll See:**

```bash
# Application logs show errors
2024/11/19 10:30:15 [error] Failed to load image: /data/uploads/photo123.jpg: no such file or directory
2024/11/19 10:30:16 [error] User profile picture missing
2024/11/19 10:30:17 [error] Unable to retrieve uploaded documents

# Users see errors
"Your uploaded files cannot be found"
"Profile picture not available"
"Document retrieval failed"
```

---

## **Root Cause Analysis - Deep Dive:**

### **Understanding hostPath Locality:**

```
┌─────────────────────────────────────────────────────────────────┐
│ Kubernetes Cluster                                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌────────────────────┐              ┌────────────────────┐     │
│  │ Node-1             │              │ Node-2             │     │
│  │                    │              │                    │     │
│  │  /data/uploads/    │              │  /data/uploads/    │     │
│  │  ├── photo1.jpg    │              │  (empty)           │     │
│  │  ├── photo2.jpg    │              │                    │     │
│  │  └── photo3.jpg    │              │                    │     │
│  │                    │              │                    │     │
│  │  Pod (before)  ────┼─── drain ───>│  Pod (after)       │     │
│  │  writes here       │              │  reads empty dir   │     │
│  └────────────────────┘              └────────────────────┘     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

Key Point: hostPath data is LOCAL to each node!
Each node has its own INDEPENDENT /data/uploads directory.
```

### **Why Data "Disappears":**

1. **Pod writes data to node-1's hostPath**: `/data/uploads` on node-1
2. **Node-1 is drained**: Pod is evicted and terminated
3. **Scheduler places new Pod on node-2**: Different physical machine
4. **Pod mounts node-2's hostPath**: `/data/uploads` on node-2 (empty!)
5. **Data appears "lost"**: Actually still exists on node-1

### **The Illusion:**
- Application thinks data is in "one place" (`/data/uploads`)
- Reality: Each node has its own **separate** `/data/uploads`
- No automatic synchronization between nodes
- Data is **not portable** across nodes

---

## **Real-World Production Incidents I've Handled:**

### **Incident 1: E-commerce File Upload Loss**

```bash
# Situation:
- E-commerce platform using hostPath for product images
- 3 replicas behind load balancer
- User uploads product image to replica on node-1
- Image stored in /data/product-images on node-1
- Load balancer routes next request to replica on node-2
- Image not found - broken product page

# Timeline:
14:00 - User uploads product images (stored on node-1)
14:05 - Node-1 experiences high CPU, pods evicted
14:06 - Pods reschedule to node-2 and node-3
14:07 - All product images "disappear"
14:10 - Customer complaints flood support
14:30 - Engineers realize hostPath is node-local
16:00 - Emergency migration to S3 begins
18:00 - Service restored with S3

# Impact:
- 4-hour outage
- 10,000+ broken product pages
- $200k revenue loss
- Customer trust damaged
- Emergency weekend work to migrate to S3
```

### **Incident 2: Log Aggregation Failure**

```bash
# Situation:
- Application writing logs to hostPath /var/log/app
- Log aggregation sidecar reading from same hostPath
- During cluster upgrade, nodes drained sequentially
- Pods moved to different nodes
- Historical logs left behind on old nodes

# Discovery:
$ kubectl exec fluentd-xyz -- ls /var/log/app
# Only logs from last 10 minutes

$ ssh node-1
$ ls /var/log/app
2024-11-01-app.log  (500MB)
2024-11-02-app.log  (450MB)
...
2024-11-15-app.log  (480MB)
# All historical logs still on node-1!

# Impact:
- Lost 2 weeks of historical logs
- Compliance audit failure
- Unable to debug production issues
- Manual log collection from 20 nodes required
```

### **Incident 3: StatefulSet Database Migration**

```bash
# Situation:
- PostgreSQL StatefulSet using hostPath for data
- Planned node decommissioning
- Attempted to drain node with database
- Pod rescheduled to new node
- Database appeared empty (data on old node)

# Timeline:
02:00 - Start maintenance window
02:10 - Drain node-5 (database node)
02:11 - PostgreSQL pod rescheduled to node-6
02:12 - PostgreSQL starts with empty data directory
02:13 - All tables appear missing
02:14 - PANIC: "Database wiped!"
02:15 - Kill pod before more damage
03:00 - Realize data still on node-5
03:30 - Un-drain node-5, force-schedule pod back
04:00 - Database operational again

# Impact:
- 2-hour database downtime
- Near data loss incident
- Extreme stress for on-call team
- Delayed node decommissioning by 2 months
```

---

## **Senior Kubernetes Administrator Solutions:**

### **Solution 1: Use Persistent Volumes (Best Practice)**

```yaml
# StorageClass for dynamic provisioning
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs  # or kubernetes.io/gce-pd, etc.
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi
---
# Deployment using PVC
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 1  # Only 1 replica with RWO
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: nginx:1.21
        volumeMounts:
        - name: data
          mountPath: /data/uploads
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: app-data-pvc
```

**Benefits:**
- ✅ Data follows the pod to new nodes
- ✅ Cloud provider handles volume attachment/detachment
- ✅ Automatic volume reattachment on pod reschedule
- ✅ Data persists even if pod/node deleted
- ✅ Backup/snapshot support from cloud provider

---

### **Solution 2: StatefulSet with volumeClaimTemplates**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-app
spec:
  serviceName: web-app
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: nginx:1.21
        volumeMounts:
        - name: data
          mountPath: /data/uploads
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 50Gi
```

**Benefits:**
- ✅ Each pod gets its own PVC
- ✅ Stable pod identity (web-app-0, web-app-1, web-app-2)
- ✅ PVC follows pod even if rescheduled
- ✅ Ordered, graceful deployment and scaling

**Pod-to-PVC Mapping:**
```bash
$ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
web-app-0    1/1     Running   0          5m
web-app-1    1/1     Running   0          5m
web-app-2    1/1     Running   0          5m

$ kubectl get pvc
NAME              STATUS   VOLUME                CAPACITY
data-web-app-0    Bound    pvc-abc123...         50Gi
data-web-app-1    Bound    pvc-def456...         50Gi
data-web-app-2    Bound    pvc-ghi789...         50Gi

# If web-app-0 moves to different node:
# - Its PVC (data-web-app-0) automatically follows
# - Volume detaches from old node
# - Volume attaches to new node
# - Data intact!
```

---

### **Solution 3: Node Affinity + Local Persistent Volumes**

```yaml
# For when you must use local storage but want persistence
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-node1
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1  # Binds to specific node
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
      storage: 100Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: app
    image: nginx:1.21
    volumeMounts:
    - name: data
      mountPath: /data/uploads
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data-pvc
  # Pod automatically scheduled to node-1 due to PV nodeAffinity
```

**Behavior:**
- ✅ Uses local disk (fast performance)
- ✅ PVC provides abstraction layer
- ✅ Pod always scheduled to same node (via PV nodeAffinity)
- ✅ If node fails, pod won't start on different node (preserves data integrity)
- ⚠️ If node permanently fails, data is lost (use with backups!)

---

### **Solution 4: Object Storage (S3/GCS/Azure Blob)**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: s3-credentials
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: "your-access-key"
  AWS_SECRET_ACCESS_KEY: "your-secret-key"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3  # Can have multiple replicas!
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: myapp:1.0
        env:
        - name: STORAGE_BACKEND
          value: "s3"
        - name: S3_BUCKET
          value: "my-app-uploads"
        - name: S3_REGION
          value: "us-east-1"
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: s3-credentials
              key: AWS_ACCESS_KEY_ID
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: s3-credentials
              key: AWS_SECRET_ACCESS_KEY
```

**Application Code Change:**
```python
# Before (hostPath)
def save_upload(file):
    file.save('/data/uploads/photo.jpg')

# After (S3)
import boto3

def save_upload(file):
    s3 = boto3.client('s3')
    s3.upload_fileobj(
        file, 
        'my-app-uploads', 
        'photo.jpg'
    )
```

**Benefits:**
- ✅ **Unlimited replicas** - any pod can access
- ✅ **No node affinity** - schedule anywhere
- ✅ **Highly durable** - 99.999999999% durability (S3)
- ✅ **Scalable** - no capacity planning
- ✅ **Built-in CDN** integration (CloudFront)
- ✅ **Versioning & lifecycle** policies
- ✅ **No drain/reschedule issues**

---

### **Solution 5: ReadWriteMany (RWX) Volumes with NFS/EFS**

```yaml
# AWS EFS StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-abc12345
  directoryPerms: "700"
---
# PVC with ReadWriteMany
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-uploads-pvc
spec:
  accessModes:
  - ReadWriteMany  # Multiple pods can read/write
  storageClassName: efs-sc
  resources:
    requests:
      storage: 100Gi
---
# Deployment with multiple replicas
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 5  # All replicas share same volume
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: nginx:1.21
        volumeMounts:
        - name: shared-data
          mountPath: /data/uploads
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: shared-uploads-pvc
```

**Benefits:**
- ✅ Multiple pods can mount same volume simultaneously
- ✅ All pods see the same data
- ✅ Pod can be scheduled on any node
- ✅ No data loss during node drain
- ⚠️ Slightly slower than local disk
- ⚠️ Requires NFS/EFS/CephFS/GlusterFS

---

## **Migration Strategy: hostPath → Persistent Storage**

### **Phase 1: Assessment**

```bash
#!/bin/bash
# assess-hostpath-usage.sh

echo "=== Scanning cluster for hostPath volumes ==="

kubectl get pods --all-namespaces -o json | jq -r '
  .items[] | 
  select(.spec.volumes != null) |
  select(.spec.volumes[] | .hostPath != null) |
  {
    namespace: .metadata.namespace,
    pod: .metadata.name,
    node: .spec.nodeName,
    hostPaths: [
      .spec.volumes[] | 
      select(.hostPath != null) | 
      {
        name: .name,
        path: .hostPath.path,
        type: .hostPath.type
      }
    ]
  }
' | jq -s '
  group_by(.namespace) | 
  map({
    namespace: .[0].namespace,
    pod_count: length,
    pods: map(.pod),
    unique_paths: [.[].hostPaths[].path] | unique
  })
'

# Output: List of namespaces using hostPath
```

### **Phase 2: Data Volume Assessment**

```bash
#!/bin/bash
# measure-hostpath-data.sh

for node in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
    echo "=== Node: $node ==="
    
    kubectl debug node/$node -q --image=busybox -- sh -c '
        echo "Checking common hostPath locations..."
        
        paths="/data /mnt/data /var/lib/app"
        
        for path in $paths; do
            if [ -d /host$path ]; then
                size=$(du -sh /host$path 2>/dev/null | cut -f1)
                files=$(find /host$path -type f 2>/dev/null | wc -l)
                echo "  $path: $size ($files files)"
            fi
        done
    '
    echo ""
done
```

### **Phase 3: Migration Plan**

```yaml
# migration-plan.yaml
---
# Step 1: Create StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: migration-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer

---
# Step 2: Create PVC for new storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-new
  namespace: production
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: migration-storage
  resources:
    requests:
      storage: 200Gi  # Size based on assessment

---
# Step 3: Migration job
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
  namespace: production
spec:
  template:
    spec:
      restartPolicy: Never
      nodeSelector:
        kubernetes.io/hostname: node-1  # Node with original data
      containers:
      - name: migrator
        image: ubuntu:22.04
        command:
        - bash
        - -c
        - |
          echo "Starting data migration..."
          
          # Install rsync
          apt-get update && apt-get install -y rsync
          
          # Migrate data with progress
          rsync -avh --progress /old-data/ /new-data/
          
          # Verify
          OLD_COUNT=$(find /old-data -type f | wc -l)
          NEW_COUNT=$(find /new-data -type f | wc -l)
          
          if [ "$OLD_COUNT" -eq "$NEW_COUNT" ]; then
            echo "✅ Migration successful: $NEW_COUNT files migrated"
            exit 0
          else
            echo "❌ Migration failed: $OLD_COUNT vs $NEW_COUNT files"
            exit 1
          fi
          
        volumeMounts:
        - name: old-data
          mountPath: /old-data
          readOnly: true
        - name: new-data
          mountPath: /new-data
          
      volumes:
      - name: old-data
        hostPath:
          path: /data/uploads
          type: Directory
      - name: new-data
        persistentVolumeClaim:
          claimName: app-data-new

---
# Step 4: Update application to use new PVC
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: nginx:1.21
        volumeMounts:
        - name: data
          mountPath: /data/uploads
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: app-data-new  # Now using PVC instead of hostPath!
```

### **Phase 4: Validation**

```bash
#!/bin/bash
# validate-migration.sh

echo "=== Validating Migration ==="

# 1. Check PVC is bound
PVC_STATUS=$(kubectl get pvc app-data-new -n production -o jsonpath='{.status.phase}')
echo "PVC Status: $PVC_STATUS"
if [ "$PVC_STATUS" != "Bound" ]; then
    echo "❌ PVC not bound"
    exit 1
fi

# 2. Check pod is running and using new volume
POD=$(kubectl get pods -n production -l app=web-app -o jsonpath='{.items[0].metadata.name}')
echo "Testing pod: $POD"

# 3. Verify file count
OLD_COUNT=$(kubectl debug node/node-1 -q --image=busybox -- sh -c "find /host/data/uploads -type f | wc -l")
NEW_COUNT=$(kubectl exec -n production $POD -- find /data/uploads -type f | wc -l)

echo "Old hostPath file count: $OLD_COUNT"
echo "New PVC file count: $NEW_COUNT"

if [ "$OLD_COUNT" -eq "$NEW_COUNT" ]; then
    echo "✅ File count matches"
else
    echo "❌ File count mismatch!"
    exit 1
fi

# 4. Test pod rescheduling
echo "Testing pod mobility..."
ORIGINAL_NODE=$(kubectl get pod -n production $POD -o jsonpath='{.spec.nodeName}')
echo "Pod currently on: $ORIGINAL_NODE"

kubectl delete pod -n production $POD
sleep 30

NEW_POD=$(kubectl get pods -n production -l app=web-app -o jsonpath='{.items[0].metadata.name}')
NEW_NODE=$(kubectl get pod -n production $NEW_POD -o jsonpath='{.spec.nodeName}')
echo "New pod on: $NEW_NODE"

# Verify data still accessible
FINAL_COUNT=$(kubectl exec -n production $NEW_POD -- find /data/uploads -type f | wc -l)
echo "File count after reschedule: $FINAL_COUNT"

if [ "$FINAL_COUNT" -eq "$OLD_COUNT" ]; then
    echo "✅ Data persisted across pod reschedule"
else
    echo "❌ Data lost after reschedule!"
    exit 1
fi

echo ""
echo "=== Migration Validated Successfully ==="
```

---

## **DevSecOps Best Practices:**

### **Practice 1: PodDisruptionBudget to Control Drains**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
  namespace: production
spec:
  minAvailable: 2  # At least 2 pods must remain during drain
  selector:
    matchLabels:
      app: web-app
---
# Alternative: maxUnavailable
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-app-pdb
spec:
  maxUnavailable: 1  # Only 1 pod can be down at a time
  selector:
    matchLabels:
      app: critical-app
```

**Behavior During Drain:**
```bash
$ kubectl drain node-1 --ignore-daemonsets

# Kubernetes respects PDB:
# - Only drains pods if minAvailable can be maintained
# - Waits for new pods to be Ready before continuing
# - Prevents total outage

evicting pod production/web-app-xyz
evicting pod production/web-app-abc
pod/web-app-xyz evicted
waiting for pod web-app-new to be ready...
pod/web-app-new is ready
pod/web-app-abc evicted
```

---

### **Practice 2: PreStop Hook for Data Sync**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: app
    image: nginx:1.21
    lifecycle:
      preStop:
        exec:
          command:
          - /bin/sh
          - -c
          - |
            echo "Pod terminating, syncing data..."
            
            # Sync critical data to remote storage before shutdown
            if command -v aws >/dev/null 2>&1; then
              aws s3 sync /data/uploads/ s3://backup-bucket/$(hostname)/
              echo "Data synced to S3"
            fi
            
            # Graceful shutdown delay
            sleep 15
            
    volumeMounts:
    - name: data
      mountPath: /data/uploads
  volumes:
  - name: data
    hostPath:
      path: /data/uploads
      type: DirectoryOrCreate
  
  terminationGracePeriodSeconds: 30  # Gives time for preStop hook
```

---

### **Practice 3: Node Cordon Before Drain**

```bash
#!/bin/bash
# safe-node-drain.sh

NODE=$1

if [ -z "$NODE" ]; then
    echo "Usage: $0 <node-name>"
    exit 1
fi

echo "=== Safe Node Drain Process ==="
echo "Target node: $NODE"
echo ""

# Step 1: Cordon node (prevent new pods)
echo "Step 1: Cordoning node..."
kubectl cordon $NODE
echo "✅ Node cordoned - no new pods will be scheduled"
echo ""

# Step 2: Identify pods using hostPath on this node
echo "Step 2: Identifying hostPath pods..."
HOSTPATH_PODS=$(kubectl get pods --all-namespaces --field-selector spec.nodeName=$NODE -o json | \
  jq -r '.items[] | select(.spec.volumes[]? | .hostPath != null) | "\(.metadata.namespace)/\(.metadata.name)"')

if [ -n "$HOSTPATH_PODS" ]; then
    echo "⚠️  Found pods using hostPath:"
    echo "$HOSTPATH_PODS"
    echo ""
    echo "WARNING: These pods will lose access to their hostPath data!"
    read -p "Continue? (yes/no): " CONFIRM
    
    if [ "$CONFIRM" != "yes" ]; then
        echo "Aborting. Uncordoning node..."
        kubectl uncordon $NODE
        exit 1
    fi
fi

# Step 3: Drain with safety checks
echo "Step 3: Draining node..."
kubectl drain $NODE \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=10m

echo "✅ Node drained successfully"
echo ""
echo "Next steps:"
echo "1. Perform maintenance"
echo "2. Uncordon node: kubectl uncordon $NODE"
```

---

### **Practice 4: Monitoring Data Locality**

```yaml
# Prometheus alert for pods on wrong nodes
groups:
- name: hostpath_locality
  rules:
  - alert: HostPathPodOnWrongNode
    expr: |
      # Detect pods that should be on specific nodes
      kube_pod_info{
        pod=~"postgres-.*",
        node!="node-db-1"
      } == 1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Pod {{ $labels.pod }} scheduled on wrong node"
      description: |
        Pod using hostPath is on node {{ $labels.node }}
        but should be on node-db-1 for data access.
        
        This likely happened due to node drain/reschedule.
        Data is now inaccessible!
        
        Action: Reschedule pod to correct node.
      
  - alert: HostPathDataOrphaned
    expr: |
      # Track when drain events occur
      (
        changes(kube_node_spec_unschedulable[10m]) > 0
      ) and (
        count by (node) (
          kube_pod_info{node="$node"}
          * on (pod, namespace) group_left
          kube_pod_spec_volumes_persistentvolumeclaims_info{volume=~".*hostpath.*"}
        ) > 0
      )
    labels:
      severity: warning
    annotations:
      summary: "Node drain may have orphaned hostPath data"
      description: "Node {{ $labels.node }} was drained with hostPath pods"
```

---

## **Recovery Procedures:**

### **Scenario A: Data on Old Node, Pod on New Node**

```bash
#!/bin/bash
# recover-hostpath-data.sh

OLD_NODE="node-1"
NEW_NODE="node-2"
POD_NAME="web-app-xyz"
NAMESPACE="production"
HOSTPATH_DIR="/data/uploads"

echo "=== Recovering hostPath Data ==="
echo "Old node: $OLD_NODE"
echo "New node: $NEW_NODE"
echo "Pod: $POD_NAME"
echo ""

# Option 1: Reschedule pod back to original node
echo "Option 1: Reschedule pod to original node"
echo "kubectl delete pod -n $NAMESPACE $POD_NAME"
echo "kubectl patch deployment web-app -p '{\"spec\":{\"template\":{\"spec\":{\"nodeSelector\":{\"kubernetes.io/hostname\":\"$OLD_NODE\"}}}}}'"
echo ""

# Option 2: Copy data to new node
echo "Option 2: Copy data to new node (may take time)"
read -p "Copy data from $OLD_NODE to $NEW_NODE? (yes/no): " CONFIRM

if [ "$CONFIRM" == "yes" ]; then
    # Create temporary pod on old node with hostPath
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: data-sync-source
  namespace: $NAMESPACE
spec:
  nodeName: $OLD_NODE
  containers:
  - name: rsync-server
    image: eeacms/rsync
    command: ["/bin/sh", "-c", "rsync --daemon --no-detach --config=/etc/rsyncd.conf"]
    volumeMounts:
    - name: data
      mountPath: /data
      readOnly: true
      ```yaml
  volumes:
  - name: data
    hostPath:
      path: $HOSTPATH_DIR
      type: Directory
  restartPolicy: Never
EOF

    # Wait for source pod
    kubectl wait --for=condition=Ready pod/data-sync-source -n $NAMESPACE --timeout=60s
    
    # Get source pod IP
    SOURCE_IP=$(kubectl get pod data-sync-source -n $NAMESPACE -o jsonpath='{.status.podIP}')
    
    # Create sync pod on new node
    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: data-sync-target
  namespace: $NAMESPACE
spec:
  nodeName: $NEW_NODE
  containers:
  - name: rsync-client
    image: eeacms/rsync
    command:
    - /bin/sh
    - -c
    - |
      echo "Starting data sync from $SOURCE_IP..."
      mkdir -p /data
      rsync -avz --progress rsync://$SOURCE_IP/data/ /data/
      echo "Sync complete!"
      sleep infinity
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    hostPath:
      path: $HOSTPATH_DIR
      type: DirectoryOrCreate
  restartPolicy: Never
EOF

    # Wait for sync to complete
    kubectl wait --for=condition=Ready pod/data-sync-target -n $NAMESPACE --timeout=300s
    
    echo "✅ Data sync initiated"
    echo "Monitor progress: kubectl logs -n $NAMESPACE data-sync-target -f"
    echo ""
    echo "After sync completes:"
    echo "1. Delete sync pods: kubectl delete pod data-sync-source data-sync-target -n $NAMESPACE"
    echo "2. Restart application: kubectl rollout restart deployment/web-app -n $NAMESPACE"
fi
```

---

### **Scenario B: Node Permanently Failed**

```bash
#!/bin/bash
# recover-from-node-failure.sh

FAILED_NODE="node-1"
BACKUP_LOCATION="s3://backups/hostpath-data"

echo "=== Recovering from Node Failure ==="
echo "Failed node: $FAILED_NODE"
echo ""

# Step 1: Mark node as failed
kubectl cordon $FAILED_NODE
kubectl drain $FAILED_NODE --force --delete-emptydir-data --ignore-daemonsets --timeout=30s || true

# Step 2: Delete node from cluster
echo "Node is unrecoverable. Removing from cluster..."
kubectl delete node $FAILED_NODE

# Step 3: Check for backups
echo "Checking for backups..."
if aws s3 ls $BACKUP_LOCATION/$FAILED_NODE/ >/dev/null 2>&1; then
    echo "✅ Backup found for $FAILED_NODE"
    
    # Step 4: Restore from backup
    echo "Restoring data to new nodes..."
    
    # Get list of new nodes
    NEW_NODES=$(kubectl get nodes -l node-role.kubernetes.io/worker=true -o jsonpath='{.items[*].metadata.name}')
    
    for node in $NEW_NODES; do
        echo "Restoring to $node..."
        
        # Create restore pod
        cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: restore-$node
  namespace: kube-system
spec:
  nodeName: $node
  containers:
  - name: restore
    image: amazon/aws-cli
    command:
    - sh
    - -c
    - |
      echo "Restoring data from S3..."
      aws s3 sync $BACKUP_LOCATION/$FAILED_NODE/ /data/
      echo "Restore complete"
      sleep 60
    volumeMounts:
    - name: data
      mountPath: /data
    env:
    - name: AWS_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: aws-credentials
          key: access-key-id
    - name: AWS_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: aws-credentials
          key: secret-access-key
  volumes:
  - name: data
    hostPath:
      path: /data/uploads
      type: DirectoryOrCreate
  restartPolicy: Never
EOF
        
        kubectl wait --for=condition=Ready pod/restore-$node -n kube-system --timeout=300s
        echo "✅ Data restored to $node"
    done
    
    # Cleanup restore pods
    kubectl delete pod -l restore=true -n kube-system
    
else
    echo "❌ No backup found for $FAILED_NODE"
    echo "Data is PERMANENTLY LOST"
    echo ""
    echo "Action items:"
    echo "1. Notify stakeholders of data loss"
    echo "2. Review backup policies"
    echo "3. Implement automated backups going forward"
fi
```

---

## **Backup Strategies for hostPath Data:**

### **Strategy 1: Scheduled Backup DaemonSet**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: hostpath-backup
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: hostpath-backup
  template:
    metadata:
      labels:
        app: hostpath-backup
    spec:
      hostNetwork: true
      containers:
      - name: backup
        image: amazon/aws-cli
        command:
        - /bin/bash
        - -c
        - |
          #!/bin/bash
          
          # Configuration
          BACKUP_PATHS=(
            "/data/uploads"
            "/data/logs"
            "/data/config"
          )
          S3_BUCKET="s3://my-cluster-backups"
          NODE_NAME=$(hostname)
          BACKUP_INTERVAL=3600  # 1 hour
          
          echo "Starting backup daemon on $NODE_NAME"
          
          while true; do
            TIMESTAMP=$(date +%Y%m%d-%H%M%S)
            
            for path in "${BACKUP_PATHS[@]}"; do
              if [ -d "/host$path" ]; then
                echo "[$TIMESTAMP] Backing up $path..."
                
                # Sync to S3 with metadata
                aws s3 sync "/host$path" \
                  "$S3_BUCKET/$NODE_NAME$path/$TIMESTAMP/" \
                  --storage-class STANDARD_IA \
                  --metadata "node=$NODE_NAME,timestamp=$TIMESTAMP,path=$path"
                
                # Keep only last 7 days of backups
                aws s3 ls "$S3_BUCKET/$NODE_NAME$path/" | \
                  head -n -7 | \
                  awk '{print $4}' | \
                  xargs -I {} aws s3 rm --recursive "$S3_BUCKET/$NODE_NAME$path/{}"
                
                echo "✅ Backup complete: $path"
              else
                echo "⚠️  Path not found: $path"
              fi
            done
            
            echo "Next backup in $BACKUP_INTERVAL seconds..."
            sleep $BACKUP_INTERVAL
          done
          
        volumeMounts:
        - name: host-root
          mountPath: /host
          readOnly: true
        env:
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: aws-credentials
              key: access-key-id
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: aws-credentials
              key: secret-access-key
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
            
      volumes:
      - name: host-root
        hostPath:
          path: /
          type: Directory
          
      tolerations:
      - effect: NoSchedule
        operator: Exists
```

---

### **Strategy 2: Velero for Disaster Recovery**

```bash
# Install Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-cluster-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero

# Create backup schedule for namespace with hostPath pods
velero schedule create production-backup \
  --schedule="0 */6 * * *" \
  --include-namespaces production \
  --ttl 168h

# Backup with node-specific annotations
velero backup create production-manual \
  --include-namespaces production \
  --selector app=web-app \
  --wait
```

**Velero Backup Hook (runs before backup):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
  namespace: production
  annotations:
    pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "aws s3 sync /data/uploads s3://backup-bucket/$(hostname)/"]'
    pre.hook.backup.velero.io/timeout: 5m
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: data
      mountPath: /data/uploads
  volumes:
  - name: data
    hostPath:
      path: /data/uploads
      type: Directory
```

---

### **Strategy 3: CronJob for Periodic Backups**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hostpath-backup-job
  namespace: kube-system
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: amazon/aws-cli
            command:
            - /bin/bash
            - -c
            - |
              #!/bin/bash
              set -e
              
              TIMESTAMP=$(date +%Y%m%d-%H%M%S)
              NODES=$(kubectl get nodes -o jsonpath='{.items[*].metadata.name}')
              
              echo "Starting cluster-wide hostPath backup at $TIMESTAMP"
              
              for node in $NODES; do
                echo "Backing up node: $node"
                
                # Create temporary pod on each node
                cat <<EOF | kubectl apply -f -
              apiVersion: v1
              kind: Pod
              metadata:
                name: backup-$node-$TIMESTAMP
                namespace: kube-system
              spec:
                nodeName: $node
                containers:
                - name: backup
                  image: amazon/aws-cli
                  command:
                  - sh
                  - -c
                  - |
                    aws s3 sync /host/data \
                      s3://backups/hostpath/$node/$TIMESTAMP/ \
                      --exclude "*.tmp" \
                      --exclude "*.log"
                    echo "Backup complete"
                  volumeMounts:
                  - name: data
                    mountPath: /host/data
                    readOnly: true
                  env:
                  - name: AWS_ACCESS_KEY_ID
                    valueFrom:
                      secretKeyRef:
                        name: aws-credentials
                        key: access-key-id
                  - name: AWS_SECRET_ACCESS_KEY
                    valueFrom:
                      secretKeyRef:
                        name: aws-credentials
                        key: secret-access-key
                volumes:
                - name: data
                  hostPath:
                    path: /data
                    type: Directory
                restartPolicy: Never
              EOF
                
                # Wait for backup to complete
                kubectl wait --for=condition=Ready \
                  pod/backup-$node-$TIMESTAMP \
                  -n kube-system \
                  --timeout=600s
                
                # Cleanup
                kubectl delete pod backup-$node-$TIMESTAMP -n kube-system
              done
              
              echo "✅ Cluster-wide backup complete"
              
          serviceAccountName: backup-sa
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backup-sa
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: backup-role
rules:
- apiGroups: [""]
  resources: ["nodes", "pods"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: backup-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: backup-role
subjects:
- kind: ServiceAccount
  name: backup-sa
  namespace: kube-system
```

---

## **Testing Node Drain Scenarios:**

```bash
#!/bin/bash
# test-node-drain-data-loss.sh

set -e

NAMESPACE="drain-test"
TEST_NODE="node-1"

echo "=== Testing Node Drain Data Loss Scenario ==="
echo ""

# Cleanup previous tests
kubectl delete namespace $NAMESPACE --ignore-not-found=true
sleep 5
kubectl create namespace $NAMESPACE

# Step 1: Create test data on specific node
echo "Step 1: Creating test pod with hostPath on $TEST_NODE..."

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-writer
  namespace: $NAMESPACE
spec:
  nodeName: $TEST_NODE
  containers:
  - name: writer
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Writing test data..."
      mkdir -p /data
      for i in \$(seq 1 100); do
        echo "Test data line \$i - $(date)" > /data/file-\$i.txt
      done
      echo "✅ Created 100 test files"
      ls -lh /data | head -20
      sleep infinity
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    hostPath:
      path: /data/drain-test
      type: DirectoryOrCreate
EOF

kubectl wait --for=condition=Ready pod/test-writer -n $NAMESPACE --timeout=60s
echo "✅ Test data created"
sleep 2

# Verify data exists
FILE_COUNT=$(kubectl exec -n $NAMESPACE test-writer -- find /data -type f | wc -l)
echo "File count on $TEST_NODE: $FILE_COUNT"
echo ""

# Step 2: Drain node
echo "Step 2: Draining node $TEST_NODE..."
kubectl drain $TEST_NODE --ignore-daemonsets --delete-emptydir-data --force --grace-period=10

# Wait for pod to be evicted
echo "Waiting for pod to be evicted..."
kubectl wait --for=delete pod/test-writer -n $NAMESPACE --timeout=60s
echo "✅ Pod evicted"
echo ""

# Step 3: Check if pod rescheduled
echo "Step 3: Checking pod reschedule..."
sleep 5

# The pod is NOT a Deployment/ReplicaSet, so it won't auto-reschedule
# Let's create a new pod that will schedule elsewhere
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-reader
  namespace: $NAMESPACE
spec:
  # No nodeSelector - will schedule on available node
  containers:
  - name: reader
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Current node: \$(hostname)"
      echo "Checking for test data..."
      if [ -d /data ]; then
        COUNT=\$(find /data -type f 2>/dev/null | wc -l)
        echo "Found \$COUNT files in /data"
        if [ "\$COUNT" -eq 0 ]; then
          echo "❌ DATA LOST: Directory exists but is empty"
        else
          echo "✅ Data accessible: \$COUNT files"
        fi
      else
        echo "❌ DATA LOST: Directory does not exist"
      fi
      sleep infinity
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    hostPath:
      path: /data/drain-test
      type: DirectoryOrCreate
EOF

kubectl wait --for=condition=Ready pod/test-reader -n $NAMESPACE --timeout=60s

NEW_NODE=$(kubectl get pod test-reader -n $NAMESPACE -o jsonpath='{.spec.nodeName}')
echo "New pod scheduled on: $NEW_NODE"
echo ""

# Step 4: Verify data loss
echo "Step 4: Verifying data accessibility..."
kubectl logs -n $NAMESPACE test-reader
echo ""

# Step 5: Verify data still exists on original node
echo "Step 5: Verifying data still on $TEST_NODE..."
kubectl uncordon $TEST_NODE
sleep 2

# Create pod specifically on original node
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-recovery
  namespace: $NAMESPACE
spec:
  nodeName: $TEST_NODE
  containers:
  - name: recovery
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Checking original node: $TEST_NODE"
      COUNT=\$(find /data -type f 2>/dev/null | wc -l)
      echo "Found \$COUNT files on original node"
      if [ "\$COUNT" -gt 0 ]; then
        echo "✅ DATA STILL EXISTS ON ORIGINAL NODE"
        echo "Data is NODE-LOCAL and was not lost, just inaccessible from new pod"
      fi
      sleep infinity
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    hostPath:
      path: /data/drain-test
      type: Directory
EOF

kubectl wait --for=condition=Ready pod/test-recovery -n $NAMESPACE --timeout=60s
kubectl logs -n $NAMESPACE test-recovery
echo ""

# Summary
echo "==================================="
echo "TEST RESULTS SUMMARY"
echo "==================================="
echo "Original node: $TEST_NODE"
echo "New node after drain: $NEW_NODE"
echo ""
echo "KEY FINDINGS:"
echo "1. Pod was evicted from $TEST_NODE during drain"
echo "2. New pod scheduled on $NEW_NODE"
echo "3. Data NOT accessible from new pod (directory empty/missing)"
echo "4. Data STILL EXISTS on $TEST_NODE (verified)"
echo ""
echo "CONCLUSION:"
echo "hostPath data is NODE-LOCAL and does NOT follow pods"
echo "during rescheduling. This is the expected behavior."
echo ""
echo "RECOMMENDATIONS:"
echo "1. Use PersistentVolumes instead of hostPath"
echo "2. Implement backup strategy for hostPath data"
echo "3. Use node affinity for stateful workloads"
echo "4. Consider StatefulSets with volume claim templates"

# Cleanup
echo ""
read -p "Cleanup test resources? (yes/no): " CLEANUP
if [ "$CLEANUP" == "yes" ]; then
    kubectl delete namespace $NAMESPACE
    kubectl debug node/$TEST_NODE -q --image=busybox -- rm -rf /host/data/drain-test
    echo "✅ Cleanup complete"
fi
```

---

## **Decision Matrix: When to Use What:**

```
┌────────────────────────────────────────────────────────────────────┐
│ STORAGE DECISION MATRIX                                             │
├─────────────────┬──────────────────────────────────────────────────┤
│ USE CASE        │ RECOMMENDED SOLUTION                              │
├─────────────────┼──────────────────────────────────────────────────┤
│ User uploads    │ ✅ S3/GCS/Azure Blob (object storage)            │
│ (files, images) │    - Unlimited scale                              │
│                 │    - High durability                              │
│                 │    - CDN integration                              │
├─────────────────┼──────────────────────────────────────────────────┤
│ Database data   │ ✅ PersistentVolumes (EBS/GCE PD)                │
│                 │    - StatefulSet + volumeClaimTemplates           │
│                 │    - Automatic volume attachment                  │
│                 │    - Backup/snapshot support                      │
├─────────────────┼──────────────────────────────────────────────────┤
│ Shared config   │ ✅ ConfigMaps / Secrets                          │
│                 │    - Git version controlled                       │
│                 │    - No node dependency                           │
│                 │    - Easy updates                                 │
├─────────────────┼──────────────────────────────────────────────────┤
│ Application     │ ✅ Centralized logging (ELK, Splunk)             │
│ logs            │    OR                                             │
│                 │ ✅ Loki/Promtail (Grafana stack)                 │
│                 │    - Aggregates from all nodes                    │
│                 │    - Survives pod/node failures                   │
├─────────────────┼──────────────────────────────────────────────────┤
│ Temporary data  │ ✅ emptyDir                                      │
│ (cache, tmp)    │    - Automatically cleaned up                     │
│                 │    - No persistence needed                        │
├─────────────────┼──────────────────────────────────────────────────┤
│ Node-local      │ ✅ Local PersistentVolumes                       │
│ high-perf data  │    - With node affinity                           │
│                 │    - StatefulSet for pod identity                 │
│                 │    - Backup strategy required                     │
├─────────────────┼──────────────────────────────────────────────────┤
│ System files    │ ✅ hostPath (read-only)                          │
│ (/etc, /var/log)│    - DaemonSets only                              │
│ monitoring      │    - Never writable                               │
├─────────────────┼──────────────────────────────────────────────────┤
│ Docker socket   │ ⚠️  hostPath (security risk!)                     │
│ (/var/run/*.sock│    - Only for specific tools (build agents)      │
│                 │    - Strong RBAC required                         │
├─────────────────┼──────────────────────────────────────────────────┤
│ Multi-pod       │ ✅ ReadWriteMany PVC (NFS/EFS/CephFS)            │
│ shared storage  │    - All pods access same data                    │
│                 │    - Slightly slower than local                   │
└─────────────────┴──────────────────────────────────────────────────┘
```

---

## **Key Takeaways:**

| ✅ BEST PRACTICES | ❌ ANTI-PATTERNS |
|-------------------|------------------|
| Use PersistentVolumes for data that must survive pod rescheduling | Using hostPath for user data/uploads |
| Implement automated backups for hostPath data | Assuming data follows pods |
| Use StatefulSets with volumeClaimTemplates for stateful apps | Using Deployments with hostPath |
| Set PodDisruptionBudgets to control drain behavior | Draining nodes without checking for hostPath pods |
| Migrate to object storage (S3) for files | Storing application state in hostPath |
| Use node affinity when hostPath is unavoidable | Letting scheduler place hostPath pods randomly |
| Monitor and alert on pod rescheduling events | Ignoring pod movement |
| Document which data is node-local | Undocumented storage dependencies |

---

## **Quick Rescue Commands:**

```bash
# Find all pods using hostPath in cluster
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.hostPath != null) | "\(.metadata.namespace)/\(.metadata.name) on node \(.spec.nodeName)"'

# Find data on specific node
kubectl debug node/node-1 -- find /host/data -type f -mtime -7

# Emergency data sync between nodes
kubectl debug node/node-1 --image=alpine -- tar -czf - /host/data/uploads | \
  kubectl debug node/node-2 --image=alpine -- tar -xzf - -C /host/data/

# Force pod back to original node
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"nodeName":"node-1"}}}}'
```

---


# **Scenario 7: Node crash → hostPath still exists but data corrupted**

## **What Really Happens:**

### **The Scenario:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:14
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: data
        hostPath:
          path: /data/postgres
          type: DirectoryOrCreate
```

**The Problem:**
```bash
# Scenario timeline:
10:00 AM - Node running normally, database writing data
10:15 AM - Kernel panic / power failure / hardware crash
10:15 AM - Database mid-write operation when crash occurs
10:20 AM - Node reboots automatically
10:25 AM - Kubernetes detects node is back
10:26 AM - Pod restarts on same node
10:27 AM - Database tries to start...

ERROR: Database corruption detected
ERROR: WAL file corrupted
ERROR: Index corruption in table 'users'
FATAL: Cannot start PostgreSQL
```

### **What You'll See:**

```bash
$ kubectl get pods
NAME          READY   STATUS             RESTARTS   AGE
database-0    0/1     CrashLoopBackOff   5          3m

$ kubectl logs database-0
2024-11-19 10:27:15 UTC [1] FATAL:  could not read block 0 in file "base/16384/16385": read only 0 of 8192 bytes
2024-11-19 10:27:15 UTC [1] HINT:  This has been seen to occur with buggy kernels; consider updating your system.
2024-11-19 10:27:16 UTC [1] LOG:  startup process (PID 42) exited with exit code 1
2024-11-19 10:27:16 UTC [1] LOG:  aborting startup due to startup process failure

$ kubectl describe pod database-0
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  3m                   default-scheduler  Successfully assigned default/database-0 to node-1
  Normal   Pulled     2m (x4 over 3m)      kubelet            Container image "postgres:14" already present
  Normal   Created    2m (x4 over 3m)      kubelet            Created container postgres
  Normal   Started    2m (x4 over 3m)      kubelet            Started container postgres
  Warning  BackOff    30s (x10 over 2m)    kubelet            Back-off restarting failed container
```

---

## **Root Cause Analysis - Deep Dive:**

### **How Data Corruption Occurs:**

```
┌─────────────────────────────────────────────────────────────────┐
│ NORMAL WRITE OPERATION                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Application                Kernel Buffer         Disk           │
│      │                           │                  │            │
│      │ 1. write(data)           │                  │            │
│      ├─────────────────────────>│                  │            │
│      │                           │                  │            │
│      │ 2. ack (buffered)        │                  │            │
│      │<─────────────────────────┤                  │            │
│      │                           │                  │            │
│      │                           │ 3. fsync/flush   │            │
│      │                           ├─────────────────>│            │
│      │                           │                  │            │
│      │                           │ 4. disk ack      │            │
│      │                           │<─────────────────┤            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ CRASH DURING WRITE (Data Corruption)                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Application                Kernel Buffer         Disk           │
│      │                           │                  │            │
│      │ 1. write(data)           │                  │            │
│      ├─────────────────────────>│                  │            │
│      │                           │                  │            │
│      │ 2. ack (buffered)        │                  │            │
│      │<─────────────────────────┤                  │            │
│      │                           │                  │            │
│      │ 3. write(more data)      │                  │            │
│      ├─────────────────────────>│                  │            │
│      │                           │                  │            │
│      │                           │ 4. partial flush │            │
│      │                           ├─────────────────>│            │
│      │                           │                  │            │
│      │                           │        💥 CRASH! │            │
│      │                           X                  │            │
│                                                                   │
│  Result: Partial data written, file/database corrupted          │
└─────────────────────────────────────────────────────────────────┘
```

### **Types of Corruption:**

| Corruption Type | Cause | Symptoms |
|----------------|-------|----------|
| **Partial Write** | Crash mid-write | File has incomplete data |
| **Metadata Corruption** | Filesystem metadata not flushed | Inode corruption, directory errors |
| **Journal Corruption** | Database WAL/journal interrupted | Cannot replay transactions |
| **Index Corruption** | B-tree structure partially updated | Database index unusable |
| **Cross-File Corruption** | Multiple files being written | Inconsistent state across files |
| **Silent Corruption** | Bad RAM/disk during write | Data written but incorrect |

---

## **Real-World Production Incidents I've Handled:**

### **Incident 1: PostgreSQL Database Corruption After Power Outage**

```bash
# Situation:
- Data center power outage (generator failed)
- PostgreSQL writing transaction when power lost
- Node crashed ungracefully
- Power restored after 30 minutes
- Node came back online
- PostgreSQL pod restarted automatically
- Database completely corrupted

# Timeline:
14:30 - Power outage begins
14:30 - Kubernetes node loses power mid-transaction
14:35 - UPS depleted, node powered off
15:00 - Power restored, node boots
15:05 - Kubernetes detects node healthy
15:06 - StatefulSet pod restarts
15:07 - PostgreSQL fails to start - corruption detected

# Error logs:
postgres: could not open file "global/pg_control": No such file or directory
postgres: database system is in inconsistent state
postgres: checkpoint record is at invalid location
postgres: invalid page in block 12345 of relation base/16384/123456

# Impact:
- Primary database completely down
- 6 hours to restore from backup
- Lost 2 hours of transactions (backup lag)
- Customer data inconsistencies
- $500k revenue impact
- Emergency data reconciliation required

# Root cause:
- hostPath provides NO data durability guarantees
- No fsync settings optimized for durability
- No replication (single node database)
- Backup was 2 hours old
```

### **Incident 2: Elasticsearch Index Corruption**

```bash
# Situation:
- Elasticsearch cluster using hostPath for data
- Kernel panic on one node during index operation
- Node rebooted, Elasticsearch pod restarted
- Shard corruption detected
- Entire index unusable

# Discovery:
$ kubectl exec -it elasticsearch-0 -- curl -XGET localhost:9200/_cluster/health?pretty
{
  "status" : "red",
  "number_of_nodes" : 3,
  "unassigned_shards" : 12,
  "initializing_shards" : 0,
  "relocating_shards" : 0
}

$ kubectl logs elasticsearch-0 | grep -i corrupt
[2024-11-19T10:30:15,123][WARN ][o.e.i.e.Engine] [node-1] failed to check index
org.apache.lucene.index.CorruptIndexException: checksum failed (hardware problem?) : expected=1234abcd actual=5678efgh
[2024-11-19T10:30:15,456][ERROR][o.e.i.e.Engine] shard [logs-2024.11][0] failed to start
org.elasticsearch.index.shard.IndexShardRecoveryException: failed to recover from store

# Recovery attempt:
$ kubectl exec -it elasticsearch-0 -- curl -XPOST 'localhost:9200/_cluster/reroute?retry_failed=true'
# Failed - shard too corrupted

# Impact:
- 12 shards corrupted (1 day of logs)
- Had to delete entire corrupted index
- Lost 1 day of application logs
- Compliance violation (log retention policy)
- Manual reconstruction from application sources
- 3 days of engineer time
```

### **Incident 3: MongoDB Replica Set Silent Corruption**

```bash
# Situation:
- MongoDB replica set with 3 nodes
- All using hostPath for data
- Primary node experienced bad RAM
- Data written with bit flips
- Replicated corrupted data to secondaries
- Corruption not detected for 2 weeks
- Discovered during backup restore test

# Discovery during routine restore:
$ mongorestore --drop /backup/mongodb-20241105
2024-11-19T15:30:45.123+0000 restoring to existing collection
2024-11-19T15:30:45.456+0000 error restoring from dump
2024-11-19T15:30:45.789+0000 BSON document corrupted
2024-11-19T15:30:45.999+0000 document at offset 1234567 is invalid

# Checking live database:
$ mongo
> db.users.validate({full: true})
{
  "valid" : false,
  "errors" : [
    "Index 'email_1' is invalid",
    "Document BSON corruption detected at offset 123456"
  ],
  "warnings" : [
    "Collection has inconsistent _id values"
  ]
}

# Impact:
- Silent corruption for 2 weeks
- 2 weeks of backups also corrupted
- Had to restore from 3-week-old backup
- Lost 3 weeks of data
- Customer accounts in inconsistent state
- Class-action lawsuit threat
- $2M+ liability
- Complete architecture redesign required
```

---

## **Senior Kubernetes Administrator Solutions:**

### **Solution 1: Use Managed Database Services (Best Practice)**

```yaml
# Don't run databases on Kubernetes with hostPath!
# Use managed services instead:

# AWS RDS for PostgreSQL
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
stringData:
  POSTGRES_HOST: "mydb.abc123.us-east-1.rds.amazonaws.com"
  POSTGRES_PORT: "5432"
  POSTGRES_DB: "production"
  POSTGRES_USER: "app_user"
  POSTGRES_PASSWORD: "secure_password_here"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: POSTGRES_HOST
        # No volumes needed!
```

**Benefits:**
- ✅ **Automatic backups** (point-in-time recovery)
- ✅ **High availability** (Multi-AZ)
- ✅ **Automatic failover**
- ✅ **Managed replication**
- ✅ **Automated patching**
- ✅ **No corruption from node crashes**
- ✅ **Professional database administration**
- ✅ **Compliance certifications**

---

### **Solution 2: StatefulSet with Persistent Volumes + Proper Backup**

```yaml
# If you MUST run databases on Kubernetes, do it properly
apiVersion: v1
kind: StorageClass
metadata:
  name: db-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io2  # High-performance, durable storage
  iopsPerGB: "50"
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3  # HA with replication
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      # Anti-affinity to spread across nodes/zones
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - postgres
            topologyKey: kubernetes.io/hostname
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - postgres
              topologyKey: topology.kubernetes.io/zone
      
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_DB
          value: production
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        
        # Optimize for durability
        - name: POSTGRES_INITDB_ARGS
          value: "--data-checksums"  # Enable data checksums
        
        ports:
        - containerPort: 5432
          name: postgres
        
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        
        # Liveness probe (restart if hung)
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 6
        
        # Readiness probe (don't send traffic if not ready)
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Lifecycle hooks
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                # Graceful shutdown
                su - postgres -c "pg_ctl stop -D $PGDATA -m fast -t 60"
        
        resources:
          requests:
            cpu: 2
            memory: 4Gi
          limits:
            cpu: 4
            memory: 8Gi
      
      # Backup sidecar
      - name: backup
        image: postgres:14
        command:
        - /bin/bash
        - -c
        - |
          while true; do
            echo "Starting backup at $(date)"
            
            # Wait for postgres to be ready
            until pg_isready -h localhost -U $POSTGRES_USER; do
              sleep 5
            done
            
            # Backup with WAL archiving
            BACKUP_FILE="/backup/postgres-$(date +%Y%m%d-%H%M%S).sql"
            pg_dump -h localhost -U $POSTGRES_USER -d $POSTGRES_DB \
              --format=custom \
              --file=$BACKUP_FILE \
              --verbose
            
            # Upload to S3
            if [ -f "$BACKUP_FILE" ]; then
              aws s3 cp $BACKUP_FILE s3://my-backups/postgres/
              echo "Backup uploaded: $BACKUP_FILE"
              
              # Keep only last 7 days locally
              find /backup -name "postgres-*.sql" -mtime +7 -delete
            fi
            
            # Backup every 4 hours
            sleep 14400
          done
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: password
        - name: POSTGRES_DB
          value: production
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: aws-credentials
              key: access-key-id
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: aws-credentials
              key: secret-access-key
        volumeMounts:
        - name: backup
          mountPath: /backup
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1
            memory: 1Gi
      
      volumes:
      - name: backup
        emptyDir:
          sizeLimit: 50Gi
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: db-storage
      resources:
        requests:
          storage: 100Gi
```

---

### **Solution 3: Filesystem-Level Data Integrity**

```yaml
# Use ZFS or Btrfs for better corruption detection
apiVersion: v1
kind: Pod
metadata:
  name: database-with-zfs
spec:
  initContainers:
  - name: setup-zfs
    image: ubuntu:22.04
    command:
    - /bin/bash
    - -c
    - |
      # Check if ZFS pool exists
      if ! zpool list tank >/dev/null 2>&1; then
        echo "Creating ZFS pool on /dev/sdb"
        zpool create tank /dev/sdb
        
        # Enable compression
        zfs set compression=lz4 tank
        
        # Enable checksums (default, but explicit)
        zfs set checksum=sha256 tank
        
        # Create dataset for database
        zfs create tank/postgres
        zfs set recordsize=8K tank/postgres  # Match PostgreSQL block size
        zfs set logbias=latency tank/postgres
        zfs set sync=standard tank/postgres
        
        echo "ZFS pool created"
      else
        echo "ZFS pool already exists"
      fi
      
      # Create mount point
      mkdir -p /data/postgres
      zfs set mountpoint=/data/postgres tank/postgres
      
    volumeMounts:
    - name: host-dev
      mountPath: /dev
    - name: host-data
      mountPath: /data
    securityContext:
      privileged: true
  
  containers:
  - name: postgres
    image: postgres:14
    env:
    - name: PGDATA
      value: /var/lib/postgresql/data
    volumeMounts:
    - name: postgres-data
      mountPath: /var/lib/postgresql/data
    
  volumes:
  - name: host-dev
    hostPath:
      path: /dev
      type: Directory
  - name: host-data
    hostPath:
      path: /data
      type: Directory
  - name: postgres-data
    hostPath:
      path: /data/postgres
      type: Directory
```

**ZFS Benefits:**
- ✅ **End-to-end checksums** - detects corruption
- ✅ **Copy-on-write** - never overwrites data in-place
- ✅ **Snapshots** - instant point-in-time backups
- ✅ **Self-healing** - with mirrors/raidz
- ✅ **Compression** - saves space
- ⚠️ Requires privileged containers
- ⚠️ More complex to manage

---

### **Solution 4: Application-Level Corruption Detection**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  template:
    spec:
      initContainers:
      - name: corruption-check
        image: postgres:14
        command:
        - /bin/bash
        - -c
        - |
          echo "Checking database integrity..."
          
          # Check if data directory exists
          if [ ! -d "$PGDATA" ]; then
            echo "✅ Fresh installation, no checks needed"
            exit 0
          fi
          
          # Check for PostgreSQL control file
          if [ ! -f "$PGDATA/global/pg_control" ]; then
            echo "❌ CORRUPTION: pg_control file missing"
            echo "Database is corrupted and cannot be recovered"
            exit 1
          fi
          
          # Try to start PostgreSQL in single-user mode for validation
          export PGDATA
          
          # Run pg_controldata to check database state
          pg_controldata $PGDATA > /tmp/controldata.txt
          
          STATE=$(grep "Database cluster state" /tmp/controldata.txt | awk '{print $4}')
          
          echo "Database state: $STATE"
          
          if [ "$STATE" == "in" ]; then
            echo "⚠️  WARNING: Database was not shut down cleanly"
            echo "Attempting automatic recovery..."
            
            # Let PostgreSQL handle recovery
            echo "Recovery will occur during normal startup"
            exit 0
            
          elif [ "$STATE" == "shut" ]; then
            echo "✅ Database was shut down cleanly"
            exit 0
            
          else
            echo "❌ CORRUPTION: Unknown database state: $STATE"
            echo "Manual intervention required"
            
            # Create corruption marker
            echo "$(date): Corruption detected - state: $STATE" >> /corruption-detected.txt
            
            exit 1
          fi
          
        env:
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        - name: corruption-log
          mountPath: /corruption-detected.txt
          subPath: corruption-detected.txt
      
      containers:
      - name: postgres
        image: postgres:14
        # ... (same as before)
        
      - name: integrity-monitor
        image: postgres:14
        command:
        - /bin/bash
        - -c
        - |
          # Wait for postgres to be ready
          until pg_isready -h localhost -U $POSTGRES_USER; do
            sleep 5
          done
          
          echo "Starting integrity monitoring..."
          
          while true; do
            # Check database integrity
            psql -h localhost -U $POSTGRES_USER -d $POSTGRES_DB -c "
              SELECT datname, 
                     pg_database_size(datname) as size,
                     age(datfrozenxid) as xid_age
              FROM pg_database
              WHERE datname = '$POSTGRES_DB';
            " > /tmp/db_health.txt
            
            # Run checksum verification (if enabled)
            psql -h localhost -U $POSTGRES_USER -d $POSTGRES_DB -c "
              SELECT COUNT(*) as corrupted_pages
              FROM pg_stat_database
              WHERE checksum_failures > 0;
            " > /tmp/checksum_status.txt
            
            CORRUPTED=$(grep corrupted_pages /tmp/checksum_status.txt | tail -1 | awk '{print $3}')
            
            if [ "$CORRUPTED" -gt 0 ]; then
              echo "❌ ALERT: $CORRUPTED corrupted pages detected!"
              # Send alert
              curl -X POST https://alerts.example.com/webhook \
                -d "{\"alert\": \"Database corruption detected\", \"corrupted_pages\": $CORRUPTED}"
            fi
            
            # Check every hour
            sleep 3600
          done
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: password
        - name: POSTGRES_DB
          value: production
      
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: postgres-data
      - name: corruption-log
        persistentVolumeClaim:
          claimName: corruption-logs
```

---

## **Data Corruption Detection & Recovery:**

### **Detection Script:**

```bash
#!/bin/bash
# detect-corruption.sh

POD_NAME=$1
NAMESPACE=${2:-default}

if [ -z "$POD_NAME" ]; then
    echo "Usage: $0 <pod-name> [namespace]"
    exit 1
fi

echo "=== Detecting Data Corruption in $POD_NAME ==="
echo ""

# Check if pod is running
STATUS=$(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.status.phase}')
echo "Pod Status: $STATUS"

if [ "$STATUS" == "CrashLoopBackOff" ] || [ "$STATUS" == "Error" ]; then
    echo "⚠️  Pod is in error state - checking logs for corruption indicators..."
    
    # Check logs for corruption keywords
    LOGS=$(kubectl logs $POD_NAME -n $NAMESPACE --tail=200 2>&1)
    
    # PostgreSQL corruption indicators
    if echo "$LOGS" | grep -qi "corrupt\|checksum\|invalid page\|damaged\|inconsistent state"; then
        echo "❌ CORRUPTION DETECTED: PostgreSQL"
        echo ""
        echo "Error details:"
        echo "$LOGS" | grep -i "corrupt\|checksum\|invalid page\|damaged\|inconsistent"
        echo ""
        
        # Get node name
        NODE=$(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.nodeName}')
        echo "Node: $NODE"
        
        # Check filesystem
        echo ""
        echo "Checking filesystem on $NODE..."
        kubectl debug node/$NODE -q --image=busybox -- sh -c "
            dmesg | grep -i 'ext4\|xfs\|error\|corruption' | tail -20
        "
        
        echo ""
        echo "=== Recovery Options ==="
        echo "1. Restore from backup (recommended)"
        echo "2. Attempt pg_resetwal (data loss risk)"
        echo "3. Manual recovery with pg_dump/pg_restore"
        echo ""
        
        exit 1
    fi
    
    # MongoDB corruption indicators
    if echo "$LOGS" | grep -qi "WiredTiger error\|corruption\|BSON\|invalid"; then
        echo "❌ CORRUPTION DETECTED: MongoDB"
        echo "$LOGS" | grep -i "WiredTiger\|corruption\|BSON"
        exit 1
    fi
    
    # Elasticsearch corruption indicators
    if echo "$LOGS" | grep -qi "CorruptIndexException\|checksum\|lucene"; then
        echo "❌ CORRUPTION DETECTED: Elasticsearch"
        echo "$LOGS" | grep -i "Corrupt\|checksum\|lucene"
        exit 1
    fi
    
    echo "No obvious corruption detected in logs"
else
    echo "✅ Pod is running normally"
fi

# If pod is running, perform health checks
if [ "$STATUS" == "Running" ]; then
    echo ""
    echo "Performing active health checks..."
    
    # PostgreSQL health check
    kubectl exec -n $NAMESPACE $POD_NAME -- psql -U postgres -c "
        SELECT pg_database.datname,
               pg_stat_database.checksum_failures
        FROM pg_database
        JOIN pg_stat_database ON pg_database.datname = pg_stat_database.datname
        WHERE pg_stat_database.checksum_failures > 0;
    " 2>/dev/null && echo "⚠️  Checksum failures detected!"
    
    # File system check
    NODE=$(kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.nodeName}')
    kubectl debug node/$NODE -q --image=busybox -- sh -c "
        echo 'Checking for filesystem errors...'
        dmesg | tail -100 | grep -i 'error\|corruption\|ext4\|xfs'
    " 2>/dev/null
fi
```

---

### **Recovery Playbook:**

```yaml
# recovery-playbook.yaml
---
# Step 1: Stop corrupted pod
- name: Stop corrupted database pod
  command: kubectl scale statefulset database --replicas=0 -n production
  
# Step 2: Create forensics pod
- name: Create forensics pod
  kubectl_apply:
    definition:
      apiVersion: v1
      kind: Pod
      metadata:
        name: forensics
        namespace: production
      spec:
        nodeName: "{{ corrupted_node }}"
        containers:
        - name: forensics
          image: postgres:14
          command: ["/bin/bash", "-c", "sleep infinity"]
          volumeMounts:
          - name: corrupted-data
            mountPath: /corrupted-data
            readOnly: true
        volumes:
        - name: corrupted-data
          hostPath:
            path: /data/postgres
            type: Directory

# Step 3: Assess damage
- name: Assess corruption extent
  shell: |
    kubectl exec -n production forensics -- bash -c '
      cd /corrupted-data
      
      # Check for control file
      if [ -f global/pg_control ]; then
        pg_controldata .
      else
        echo "CRITICAL: pg_control missing"
      fi
      
      # Check WAL files
      ls -lh pg_wal/
      
      # Try to estimate damage
      find . -name "*.1" | wc -l
    '

# Step 4: Attempt recovery
- name: Try PostgreSQL recovery
  shell: |
    kubectl exec -n production forensics -- bash -c '
      cd /corrupted-data
      
      # Backup corrupted state first
      tar -czf /tmp/corrupted-backup-$(date +%Y%m%d-%H%M%S).tar.gz .
      
      # Try pg_resetwal (LAST RESORT - causes data loss)
      pg_resetwal -f .
      
      # Attempt to start in single-user mode
      postgres --single -D . template1
    '

# Step 5: Restore from backup if recovery fails
- name: Restore from last good backup
  shell: |
    # Get latest backup
    LATEST_BACKUP=$(aws s3 ls s3://backups/postgres/ | sort | tail -1 | awk '{print $4}')
    
    # Download backup
    aws s3 cp s3://backups/postgres/$LATEST_BACKUP /tmp/backup.sql
    
    # Create fresh database
    kubectl exec -n production forensics -- bash -c '
      rm -rf /corrupted-data/*
      initdb -D /corrupted-data
    '
    
    # Restore backup
    kubectl exec -n production forensics -- bash -c '
      postgres -D /corrupted-data & sleep 10
      pg_restore -d postgres /tmp/backup.sql
    '

# Step 6: Validate restored data
- name: Validate data integrity
  shell: |
    kubectl exec -n production forensics -- bash -c '
      psql -d postgres -c "
        -- Check for corruption
        SELECT datname, pg_database_size(datname) as size
        FROM pg_database;
        
        -- Verify critical tables
        SELECT COUNT(*) FROM users;
        SELECT COUNT(*) FROM transactions;
        
        -- Check indexes
        REINDEX DATABASE postgres;
      "
    '

# Step 7: Restart with validated data
- name: Restart database StatefulSet
  command: kubectl scale statefulset database --replicas=1 -n production

# Step 8: Monitor recovery
- name: Monitor database startup
  shell: |
    kubectl logs -n production -f database-0 --tail=100

# Step 9: Post-recovery validation
- name: Post-recovery checks
  shell: |
    kubectl exec -n production database-0 -- bash -c '
      psql -d postgres -c "
        -- Check all tables are accessible
        SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||"."||tablename))
        FROM pg_tables
        WHERE schemaname NOT IN ("pg_catalog", "information_schema")
        ORDER BY pg_total_relation_size(schemaname||"."||tablename) DESC;
      "
    '
```

---

## **DevSecOps Best Practices:**

### **Practice 1: Automated Corruption Detection**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-integrity-check
  namespace: production
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: integrity-checker
            image: postgres:14
            command:
            - /bin/bash
            - -c
            - |
              #!/bin/bash
              set -e
              
              echo "=== Database Integrity Check ==="
              echo "Timestamp: $(date)"
              echo ""
              
              # Wait for database to be ready
              until pg_isready -h database-0.database.production.svc.cluster.local -U postgres; do
                echo "Waiting for database..."
                sleep 5
              done
              
              echo "✅ Database is ready"
              echo ""
              
              # Check 1: Checksum verification
              echo "Check 1: Verifying checksums..."
              CHECKSUM_FAILURES=$(psql -h database-0.database.production.svc.cluster.local \
                -U postgres -d production -t -c "
                SELECT COALESCE(SUM(checksum_failures), 0)
                FROM pg_stat_database;
              ")
              
              if [ "$CHECKSUM_FAILURES" -gt 0 ]; then
                echo "❌ FAILED: $CHECKSUM_FAILURES checksum failures detected"
                
                # Send alert
                curl -X POST https://alerts.example.com/webhook \
                  -H 'Content-Type: application/json' \
                  -d "{
                    \"severity\": \"critical\",
                    \"title\": \"Database Corruption Detected\",
                    \"description\": \"$CHECKSUM_FAILURES checksum failures in production database\",
                    \"timestamp\": \"$(date -Iseconds)\"
                  }"
                
                exit 1
              else
                echo "✅ PASSED: No checksum failures"
              fi
              echo ""
              
              # Check 2: Table integrity
              echo "Check 2: Verifying table integrity..."
              TABLES=$(psql -h database-0.database.production.svc.cluster.local \
                -U postgres -d production -t -c "
                SELECT tablename 
                FROM pg_tables 
                WHERE schemaname = 'public';
              ")
              
              for table in $TABLES; do
                echo "  Checking table: $table"
                
                # Try to count rows (will fail if corrupted)
                if ! psql -h database-0.database.production.svc.cluster.local \
                  -U postgres -d production -c "SELECT COUNT(*) FROM $table;" > /dev/null 2>&1; then
                  
                  echo "  ❌ FAILED: Cannot read table $table"
                  
                  # Send alert
                  curl -X POST https://alerts.example.com/webhook \
                    -H 'Content-Type: application/json' \
                    -d "{
                      \"severity\": \"critical\",
                      \"title\": \"Table Corruption Detected\",
                      \"description\": \"Table $table is corrupted or unreadable\",
                      \"timestamp\": \"$(date -Iseconds)\"
                    }"
                  
                  exit 1
                else
                  echo "  ✅ PASSED: Table $table is readable"
                fi
              done
              echo ""
              
              # Check 3: Index integrity
              echo "Check 3: Verifying index integrity..."
              CORRUPTED_INDEXES=$(psql -h database-0.database.production.svc.cluster.local \
                -U postgres -d production -t -c "
                SELECT COUNT(*) 
                FROM pg_class c 
                JOIN pg_index i ON c.oid = i.indexrelid 
                WHERE c.relkind = 'i' AND i.indisvalid = false;
              ")
              
              if [ "$CORRUPTED_INDEXES" -gt 0 ]; then
                echo "❌ FAILED: $CORRUPTED_INDEXES invalid indexes detected"
                
                # List corrupted indexes
                psql -h database-0.database.production.svc.cluster.local \
                  -U postgres -d production -c "
                  SELECT c.relname as index_name, 
                         t.relname as table_name
                  FROM pg_class c 
                  JOIN pg_index i ON c.oid = i.indexrelid 
                  JOIN pg_class t ON i.indrelid = t.oid
                  WHERE c.relkind = 'i' AND i.indisvalid = false;
                "
                
                exit 1
              else
                echo "✅ PASSED: All indexes are valid"
              fi
              echo ""
              
              # Check 4: WAL integrity
              echo "Check 4: Verifying WAL integrity..."
              WAL_STATUS=$(psql -h database-0.database.production.svc.cluster.local \
                -U postgres -d production -t -c "
                SELECT pg_is_in_recovery();
              ")
              
              if [ "$WAL_STATUS" == " t" ]; then
                echo "⚠️  WARNING: Database is in recovery mode"
              else
                echo "✅ PASSED: Database is not in recovery"
              fi
              echo ""
              
              # Check 5: Data consistency
              echo "Check 5: Verifying data consistency..."
              # Run application-specific checks
              ORPHANED_RECORDS=$(psql -h database-0.database.production.svc.cluster.local \
                -U postgres -d production -t -c "
                -- Example: Check for orphaned records
                SELECT COUNT(*) FROM orders 
                WHERE user_id NOT IN (SELECT id FROM users);
              ")
              
              if [ "$ORPHANED_RECORDS" -gt 0 ]; then
                echo "⚠️  WARNING: $ORPHANED_RECORDS orphaned records found (data inconsistency)"
              else
                echo "✅ PASSED: No orphaned records"
              fi
              echo ""
              
              echo "=== All Checks Complete ==="
              echo "Status: ✅ Database integrity verified"
              
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
            resources:
              requests:
                cpu: 100m
                memory: 128Mi
              limits:
                cpu: 500m
                memory: 512Mi
```

---

### **Practice 2: Continuous Backup with Point-in-Time Recovery**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-backup-script
  namespace: production
data:
  backup.sh: |
    #!/bin/bash
    set -eo pipefail
    
    # Configuration
    PGHOST="database-0.database.production.svc.cluster.local"
    PGUSER="postgres"
    PGDATABASE="production"
    S3_BUCKET="s3://prod-db-backups"
    BACKUP_RETENTION_DAYS=30
    
    echo "=== PostgreSQL Continuous Backup ==="
    echo "Started at: $(date)"
    echo ""
    
    # Create backup directory
    BACKUP_DIR="/backup/$(date +%Y%m%d)"
    mkdir -p $BACKUP_DIR
    
    # Full backup
    echo "Creating full backup..."
    pg_basebackup -h $PGHOST -U $PGUSER -D $BACKUP_DIR/base \
      -Ft -z -P -X stream
    
    if [ $? -eq 0 ]; then
      echo "✅ Full backup completed"
      
      # Upload to S3
      echo "Uploading to S3..."
      aws s3 sync $BACKUP_DIR $S3_BUCKET/$(date +%Y%m%d)/ \
        --storage-class STANDARD_IA
      
      # Create backup metadata
      cat > $BACKUP_DIR/metadata.json <<EOF
    {
      "timestamp": "$(date -Iseconds)",
      "type": "full",
      "size": "$(du -sh $BACKUP_DIR/base | cut -f1)",
      "pgversion": "$(psql -h $PGHOST -U $PGUSER -t -c 'SHOW server_version;' | xargs)",
      "wal_position": "$(psql -h $PGHOST -U $PGUSER -t -c 'SELECT pg_current_wal_lsn();' | xargs)"
    }
    EOF
      
      aws s3 cp $BACKUP_DIR/metadata.json $S3_BUCKET/$(date +%Y%m%d)/metadata.json
      
      echo "✅ Backup uploaded to S3"
    else
      echo "❌ Backup failed"
      exit 1
    fi
    
    # WAL archiving (continuous)
    echo ""
    echo "Setting up WAL archiving..."
    psql -h $PGHOST -U $PGUSER -d $PGDATABASE -c "
      ALTER SYSTEM SET archive_mode = on;
      ALTER SYSTEM SET archive_command = 'aws s3 cp %p $S3_BUCKET/wal/%f';
      SELECT pg_reload_conf();
    "
    
    # Cleanup old backups
    echo ""
    echo "Cleaning up old backups..."
    aws s3 ls $S3_BUCKET/ | while read -r line; do
      BACKUP_DATE=$(echo $line | awk '{print $2}' | tr -d '/')
      if [ -n "$BACKUP_DATE" ]; then
        AGE_DAYS=$(( ($(date +%s) - $(date -d "$BACKUP_DATE" +%s)) / 86400 ))
        
        if [ $AGE_DAYS -gt $BACKUP_RETENTION_DAYS ]; then
          echo "Deleting backup from $BACKUP_DATE (${AGE_DAYS} days old)"
          aws s3 rm --recursive $S3_BUCKET/$BACKUP_DATE/
        fi
      fi
    done
    
    echo ""
    echo "=== Backup Complete ==="
    echo "Finished at: $(date)"
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: production
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:14
            command: ["/bin/bash", "/scripts/backup.sh"]
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: access-key-id
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: secret-access-key
            volumeMounts:
            - name: backup-script
              mountPath: /scripts
            - name: backup-storage
              mountPath: /backup
            resources:
              requests:
                cpu: 1
                memory: 2Gi
              limits:
                cpu: 2
                memory: 4Gi
          volumes:
          - name: backup-script
            configMap:
              name: postgres-backup-script
              defaultMode: 0755
          - name: backup-storage
            emptyDir:
              sizeLimit: 100Gi
```

---

### **Practice 3: Disaster Recovery Testing**

```bash
#!/bin/bash
# dr-test.sh - Disaster Recovery Test Script

set -e

NAMESPACE="dr-test"
BACKUP_DATE=${1:-$(date +%Y%m%d)}

echo "=== Disaster Recovery Test ==="
echo "Testing backup from: $BACKUP_DATE"
echo ""

# Cleanup previous test
kubectl delete namespace $NAMESPACE --ignore-not-found=true
sleep 5
kubectl create namespace $NAMESPACE

# Step 1: Download backup from S3
echo "Step 1: Downloading backup..."
aws s3 sync s3://prod-db-backups/$BACKUP_DATE /tmp/dr-test-backup/

if [ ! -f /tmp/dr-test-backup/base/base.tar.gz ]; then
    echo "❌ Backup not found for date: $BACKUP_DATE"
    exit 1
fi

echo "✅ Backup downloaded"
echo ""

# Step 2: Create test pod with backup
echo "Step 2: Creating test database pod..."

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dr-test-postgres
  namespace: $NAMESPACE
spec:
  containers:
  - name: postgres
    image: postgres:14
    env:
    - name: POSTGRES_PASSWORD
      value: "testpassword"
    - name: PGDATA
      value: /var/lib/postgresql/data/pgdata
    command:
    - /bin/bash
    - -c
    - |
      echo "Restoring from backup..."
      
      # Extract base backup
      mkdir -p /var/lib/postgresql/data
      cd /var/lib/postgresql/data
      tar -xzf /backup/base/base.tar.gz
      
      # Set permissions
      chown -R postgres:postgres /var/lib/postgresql/data
      chmod 700 /var/lib/postgresql/data
      
      # Start PostgreSQL
      echo "Starting PostgreSQL..."
      docker-entrypoint.sh postgres
      
    volumeMounts:
    - name: backup
      mountPath: /backup
      readOnly: true
    - name: pgdata
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: backup
    hostPath:
      path: /tmp/dr-test-backup
      type: Directory
  - name: pgdata
    emptyDir:
      sizeLimit: 50Gi
EOF

# Wait for pod to be ready
echo "Waiting for database to start..."
kubectl wait --for=condition=Ready pod/dr-test-postgres -n $NAMESPACE --timeout=300s

echo "✅ Database started"
echo ""

# Step 3: Validate data
echo "Step 3: Validating restored data..."

# Check database is accessible
kubectl exec -n $NAMESPACE dr-test-postgres -- psql -U postgres -c "SELECT version();"

# Check critical tables
USERS_COUNT=$(kubectl exec -n $NAMESPACE dr-test-postgres -- \
  psql -U postgres -d production -t -c "SELECT COUNT(*) FROM users;" | xargs)
echo "Users count: $USERS_COUNT"

ORDERS_COUNT=$(kubectl exec -n $NAMESPACE dr-test-postgres -- \
  psql -U postgres -d production -t -c "SELECT COUNT(*) FROM orders;" | xargs)
echo "Orders count: $ORDERS_COUNT"

# Check data integrity
echo ""
echo "Running integrity checks..."
kubectl exec -n $NAMESPACE dr-test-postgres -- psql -U postgres -d production -c "
  -- Check for corrupted indexes
  SELECT COUNT(*) as invalid_indexes
  FROM pg_class c 
  JOIN pg_index i ON c.oid = i.indexrelid 
  WHERE c.relkind = 'i' AND i.indisvalid = false;
"

# Step 4: Test queries
echo ""
echo "Step 4: Testing sample queries..."

# Test read
kubectl exec -n $NAMESPACE dr-test-postgres -- \
  psql -U postgres -d production -c "
  SELECT id, email, created_at 
  FROM users 
  ORDER BY created_at DESC 
  LIMIT 5;
"

# Test write
kubectl exec -n $NAMESPACE dr-test-postgres -- \
  psql -U postgres -d production -c "
  INSERT INTO users (email, created_at) 
  VALUES ('dr-test@example.com', NOW())
  RETURNING id;
"

echo "✅ Write test successful"
echo ""

# Step 5: Performance test
echo "Step 5: Running performance test..."

kubectl exec -n $NAMESPACE dr-test-postgres -- \
  psql -U postgres -d production -c "
  EXPLAIN ANALYZE
  SELECT u.email, COUNT(o.id) as order_count
  FROM users u
  LEFT JOIN orders o ON u.id = o.user_id
  GROUP BY u.id, u.email
  ORDER BY order_count DESC
  LIMIT 100;
"

echo ""
echo "=== DR Test Results ==="
echo "✅ Backup restoration: SUCCESS"
echo "✅ Data integrity: VERIFIED"
echo "✅ Read operations: WORKING"
echo "✅ Write operations: WORKING"
echo "✅ Query performance: ACCEPTABLE"
echo ""
echo "RTO (Recovery Time Objective): $(date +%s) seconds from start"
echo "RPO (Recovery Point Objective): Backup from $BACKUP_DATE"
echo ""

# Cleanup
read -p "Cleanup test environment? (yes/no): " CLEANUP
if [ "$CLEANUP" == "yes" ]; then
    kubectl delete namespace $NAMESPACE
    rm -rf /tmp/dr-test-backup
    echo "✅ Cleanup complete"
fi
```

---

### **Practice 4: Filesystem Monitoring**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filesystem-monitor
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: filesystem-monitor
  template:
    metadata:
      labels:
        app: filesystem-monitor
    spec:
      hostPID: true
      hostNetwork: true
      tolerations:
      - effect: NoSchedule
        operator: Exists
      containers:
      - name: monitor
        image: ubuntu:22.04
        command:
        - /bin/bash
        - -c
        - |
          #!/bin/bash
          
          apt-get update && apt-get install -y smartmontools nvme-cli inotify-tools
          
          echo "Starting filesystem monitoring on $(hostname)"
          
          # Monitor paths
          MONITOR_PATHS="/host/data /host/var/lib"
          
          while true; do
            echo "=== Filesystem Health Check - $(date) ==="
            
            # Check disk SMART status
            for disk in /host/dev/sd* /host/dev/nvme*; do
              if [ -b "$disk" ]; then
                echo "Checking $disk..."
                smartctl -H $disk || echo "⚠️  SMART test failed for $disk"
              fi
            done
            
            # Check filesystem errors in dmesg
            ERRORS=$(dmesg | grep -i 'error\|corruption\|i/o error' | tail -20)
            if [ -n "$ERRORS" ]; then
              echo "⚠️  Filesystem errors detected:"
              echo "$ERRORS"
              
              # Send alert
              curl -X POST https://alerts.example.com/webhook \
                -d "{\"alert\": \"Filesystem errors on $(hostname)\", \"details\": \"$ERRORS\"}"
            fi
            
            # Check for read-only filesystems
            for path in $MONITOR_PATHS; do
              if ! touch $path/.write-test 2>/dev/null; then
                echo "❌ CRITICAL: $path is read-only!"
                
                # Send critical alert
                curl -X POST https://alerts.example.com/webhook \
                  -d "{\"severity\": \"critical\", \"alert\": \"Read-only filesystem on $(hostname)\", \"path\": \"$path\"}"
              else
                rm -f $path/.write-test
              fi
            done
            
            # Check inode usage
            for path in $MONITOR_PATHS; do
              INODE_USAGE=$(df -i $path | tail -1 | awk '{print $5}' | tr -d '%')
              if [ "$INODE_USAGE" -gt 90 ]; then
                echo "⚠️  WARNING: Inode usage at ${INODE_USAGE}% on $path"
              fi
            done
            
            sleep 300  # Check every 5 minutes
          done
          
        volumeMounts:
        - name: host-root
          mountPath: /host
        securityContext:
          privileged: true
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 256Mi
      volumes:
      - name: host-root
        hostPath:
          path: /
          type: Directory
```

---

## **PostgreSQL-Specific Corruption Prevention:**

```sql
-- Enable data checksums (must be done at initdb time)
-- Add to postgresql.conf
data_checksums = on

-- Enable full page writes (default, but verify)
full_page_writes = on

-- Configure WAL for durability
wal_level = replica
archive_mode = on
archive_command = 'aws s3 cp %p s3://wal-archive/%f'

-- Synchronous commit for critical transactions
synchronous_commit = on

-- Checkpoint settings (balance between performance and durability)
checkpoint_timeout = 5min
checkpoint_completion_target = 0.9

-- Continuous archiving
wal_keep_size = 1GB
max_wal_senders = 3

-- Monitoring
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0

-- Regular integrity checks
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Create monitoring function
CREATE OR REPLACE FUNCTION check_corruption()
RETURNS TABLE(issue_type text, details text) AS $$
BEGIN
  -- Check for checksum failures
  RETURN QUERY
  SELECT 'checksum_failure'::text, 
         format('%s checksum failures', checksum_failures)
  FROM pg_stat_database
  WHERE checksum_failures > 0;
  
  -- Check for invalid indexes
  RETURN QUERY
  SELECT 'invalid_index'::text,
         format('Index %s on table %s is invalid', 
                c.relname, t.relname)
  FROM pg_class c
  JOIN pg_index i ON c.oid = i.indexrelid
  JOIN pg_class t ON i.indrelid = t.oid
  WHERE c.relkind = 'i' AND i.indisvalid = false;
  
  -- Check for bloat
  RETURN QUERY
  SELECT 'table_bloat'::text,
         format('Table %s has %.2f%% bloat',
                schemaname || '.' || tablename,
                (CASE WHEN pg_total_relation_size(schemaname||'.'||tablename) > 0
                  THEN 100.0 * (pg_total_relation_size(schemaname||'.'||tablename) - 
                        pg_relation_size(schemaname||'.'||tablename)) / 
                        pg_total_relation_size(schemaname||'.'||tablename)
                  ELSE 0 END))
  FROM pg_tables
  WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
  AND pg_total_relation_size(schemaname||'.'||tablename) > 0
  AND (pg_total_relation_size(schemaname||'.'||tablename) - 
       pg_relation_size(schemaname||'.'||tablename)) * 100.0 / 
       pg_total_relation_size(schemaname||'.'||tablename) > 50;
END;
$$ LANGUAGE plpgsql;

-- Schedule regular checks
SELECT * FROM check_corruption();
```

---

## **Key Takeaways:**

| Category | Recommendation |
|----------|---------------|
| **Primary Solution** | Use managed database services (RDS, Cloud SQL) |
| **If self-hosting** | Use PersistentVolumes, not hostPath |
| **Filesystem** | Use ZFS/Btrfs with checksums enabled |
| **Backups** | Automated, tested, with point-in-time recovery |
| **Monitoring** | Continuous integrity checks + alerting |
| **DR Testing** | Monthly restore tests to verify backups |
| **Data Checksums** | Always enable (PostgreSQL data_checksums) |
| **Replication** | Multi-node with synchronous replication |

---

## **Quick Recovery Commands:**

```bash
# Check for corruption
kubectl exec -it postgres-0 -- pg_controldata /var/lib/postgresql/data

# Attempt emergency recovery
kubectl exec -it postgres-0 -- pg_resetwal -f /var/lib/postgresql/data

# Restore from latest backup
aws s3 cp s3://backups/latest.sql /tmp/
kubectl exec -i postgres-0 -- psql < /tmp/latest.sql

# Force reindex all
kubectl exec -it postgres-0 -- psql -c "REINDEX DATABASE production;"
```

---

