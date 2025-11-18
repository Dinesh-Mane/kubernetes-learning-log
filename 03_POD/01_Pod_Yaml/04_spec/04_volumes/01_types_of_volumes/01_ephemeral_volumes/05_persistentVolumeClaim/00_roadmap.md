# **1. Master List — Areas You MUST Understand**

These are the high-level areas you must know before touching PVCs:

### **1.1 Fundamental Concepts**

* What is a PersistentVolume (PV)?
* What is a PersistentVolumeClaim (PVC)?
* How PVC binds to PV (binding process)
* dynamic provisioning vs static provisioning
* Access Modes (RWO, ROX, RWX)
* StorageClasses
* Reclaim Policies (Retain, Delete, Recycle - deprecated)
* Volume Modes (Filesystem vs Block)
* PVC lifecycle (Pending → Bound → Released → Failed)
* PV lifecycle
* How pods use PVCs
* How ReadWriteOnce affects scheduling

---

### **1.2 PVC in Pod Spec**

Understand the **exact mechanics**:

| Component                                          | Purpose                                |
| -------------------------------------------------- | -------------------------------------- |
| `pod.spec.volumes.persistentVolumeClaim.claimName` | Which PVC to use                       |
| `readOnly`                                         | Restrict pod to read-only mode         |
| `volumeMounts`                                     | Path inside container where PV appears |

---

### **1.3 CSI Drivers**

* What is CSI?
* Why CSI replaced in-tree volume plugins?
* How CSI manages provisioning
* Driver-specific capabilities
* Common CSI drivers: EBS, GCE-PD, AzureDisk, Ceph, NFS, Longhorn, Portworx

---

### **1.4 StorageClasses**

You MUST know:

* What StorageClass does
* `provisioner` field
* Parameters (type=gp2 for AWS etc.)
* `allowVolumeExpansion`
* `reclaimPolicy`
* `volumeBindingMode` (“Immediate” vs “WaitForFirstConsumer”)
* Topology constraints (AZ-aware provisioning)

---

### **1.5 Dynamic Provisioning**

Deep understanding required:

* PVC creates PV automatically
* StorageClass decides how
* Node scheduling uses access mode + topology
* Why WaitForFirstConsumer prevents wrong AZ provisioning

---

### **1.6 Static Provisioning**

Know when to use it:

* Pre-created PV
* Manually bind PVC to PV using:

  * `claimRef`
  * matching storage size
  * matching access modes

---

### **1.7 Volume Expansion**

* Expanding PVC size
* Requirements:

  * StorageClass must allow `allowVolumeExpansion: true`
  * Filesystem supports online expansion
  * CSI plugin must support resize
* Offline vs online expansion
* Pod restart cases

---

### **1.8 Access Modes**

Extremely important:

| Mode | Meaning       | Common Providers |
| ---- | ------------- | ---------------- |
| RWO  | ReadWriteOnce | EBS, GCE-PD      |
| RWX  | ReadWriteMany | NFS, CephFS, EFS |
| ROX  | ReadOnlyMany  | Rare             |

Access mode decides:

* Multi-pod sharing
* Scheduling
* Which storage backend possible

---

### **1.9 Volume Modes**

| VolumeMode | Description                      |
| ---------- | -------------------------------- |
| Filesystem | Default mode, ext4/xfs formatted |
| Block      | Raw block device, no filesystem  |

Block mode requires:

* Apps that expect block devices (e.g. databases)
* Special mount handling

---

# **2. Core Concepts You MUST Master (Deep Knowledge)**

### **2.1 How Binding Actually Works**

PV binding rules:

* Match storage size
* Match access mode
* Match volumeMode
* Match label selectors
* If multiple PVs match → smallest available is picked

Dynamic binding:

* PVC triggers storage provisioning
* CSI driver creates cloud/block-based volume
* PV object gets generated
* PVC becomes Bound

---

### **2.2 Node Scheduling Logic**

For ReadWriteOnce:

* Pod MUST be scheduled on the same node as the PV
  For cloud providers:
* PV is AZ-specific → pod must schedule in same AZ

That's why **WaitForFirstConsumer** StorageClass is recommended.

---

### **2.3 Retain Policy**

Reclaim Policy = Retain:

* PV NOT deleted after PVC deletion
* Volume remains in cloud provider
* Admin must clean manually
* Used for forensic retention

---

### **2.4 Data Persistence**

Understand:

* Pods can restart → data persists
* Deployments reschedule pods on new nodes:

  * For RWO, only one node
  * For RWX, multiple nodes allowed

---

### **2.5 PVC and StatefulSets**

Know how PVC templates work:

* `volumeClaimTemplates`
* Each replica gets its own PVC
* PVC name includes StatefulSet name and index
* Persistent identity for pods

---

### **2.6 Cross-node Failover**

Different providers behave differently:

* EBS: NOT multi-AZ, NOT multi-node
* EFS: RWX, multi-AZ
* CephFS: RWX
* NFS: RWX

Know which storage supports which scenarios.

---

# **3. Interview Questions You MUST Be Ready For**

### **3.1 Basic Questions**

* What is a PVC and how is it used?
* Difference between PV and PVC?
* Access modes in Kubernetes?
* Dynamic vs static provisioning?
* What happens if PVC requests more size than PV?
* How do you mount PVC in a pod?

---

### **3.2 Intermediate Questions**

* Explain PV and PVC lifecycle.
* Explain access modes with real examples.
* What is a StorageClass and why do we need it?
* When does PVC enter Pending state?
* How do you expand a PVC?
* Why does RWO restrict scheduling?
* Explain WaitForFirstConsumer.

---

### **3.3 Senior-Level Questions**

* In-depth PVC binding algorithm
* CSI architecture: controller, node plugin
* How storage topology affects pod placement
* Why do StatefulSets need volumeClaimTemplates?
* RWX vs RWO: impact on distributed apps
* Raw block volume use cases
* Reclaim policy Retain vs Delete
* How to troubleshoot PVC stuck in Released state
* Why pod stuck in ContainerCreating when PVC not bound
* StorageClass parameters for cloud providers

---

### **3.4 DevSecOps Questions**

* How do you ensure encryption at rest?
* How do you ensure encryption in transit?
* Storage isolation between tenants?
* Data remanence after PV deletion?
* Backup & restore strategies for PV data?
* How to protect storage credentials in CSI drivers?

---

# **4. Common Pitfalls, Mistakes, & Dangerous Scenarios**

### **4.1 Requesting RWX but StorageClass Doesn't Support RWX**

Example:

* AWS EBS → RWO only
* PVC requesting RWX stays Pending forever

---

### **4.2 Wrong AZ Scheduling**

Pod can't start because:

* PV is in us-east-1a
* Pod scheduling attempts in us-east-1b
* PVC stuck in Pending/ContainerCreating

Solution:
Use `volumeBindingMode: WaitForFirstConsumer`.

---

### **4.3 Expanding PVC Without Supported Storage**

PVC resize will:

* Show "FileSystemResizePending"
* Pod must restart
* CSI driver must support expansion

Common failure source.

---

### **4.4 Using Static PV with Wrong Size**

PVC requests 10Gi
PV has 5Gi
→ PVC stuck in Pending forever

---

### **4.5 Deleting PVC with Retain Policy**

PV moves to Released state
Data remains on disk/cloud
Admin must:

* Reclaim
* Cleanup
* Reset PV

A common production problem.

---

### **4.6 Mounting PVC Over Existing App Directory**

Mounting over `/var/lib/mysql` without careful planning →
default DB files hidden →
DB won’t start →
Misconfigured deployments

---

### **4.7 Using Same PVC in Multiple Pods (RWO)**

Mount in two pods simultaneously →
One pod will fail scheduling, or fail to start →
Pod stuck in ContainerCreating

---

### **4.8 Accidentally Selecting Wrong StorageClass**

example:
Stateful apps → gp3
But developer sets → sc=efs
→ low latency required, but EFS is high latency
→ huge performance degradation

---

### **4.9 Deleting PV Before Deleting PVC**

PVC cannot detach
PV stuck forever
Must manually patch

---

### **4.10 Expecting Data Persistence With Ephemeral PVC**

Longhorn, CephFS, EFS can be persistent
However ephemeral PVCs (alpha feature) do NOT persist

---

