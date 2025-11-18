# **Persistent Volume Plugins — What You Must Understand**

First:
**Persistent Volume Plugins** = older, in-tree volume types that allow Pods to directly mount storage backends **WITHOUT using PVC/PV mechanism**.

These plugins include (legacy but still widely used in on-prem clusters):

* awsElasticBlockStore
* gcePersistentDisk
* azureDisk
* azureFile
* nfs
* iscsi
* glusterfs
* cephfs
* cinder
* vsphereVolume
* flexVolume
* fc (fibre channel)
* flocker
* portworxVolume
* quobyte
* rbd
* scaleIO
* storageOS

These volume types appear directly inside:

```yaml
pod.spec.volumes:
  - name: myvol
    nfs:
      server: 10.0.0.1
      path: /data
```

---

# **Core Areas You Must Master (Deep Understanding Required)**

## **1. The difference between PV plugins vs PVC/PV**

You must know:

* PV plugins = Pod directly mounts backend storage (legacy approach).
* PVC/PV = Dynamic + recommended approach; abstracts backend.
* PV plugins are **deprecated** in many platforms.
* Some still used heavily on-prem: NFS, iSCSI, CephFS, GlusterFS.

---

## **2. Which plugins are in-tree vs CSI**

You must understand:

* Kubernetes is removing *in-tree* plugins.
* Migration to CSI = modern, supported method.
* Interviewers often ask about deprecation.

Example:

* `awsElasticBlockStore` → replaced by **EBS CSI Driver**
* `gcePersistentDisk` → replaced by **GCE PD CSI Driver**
* `azureDisk` → replaced by **Azure Disk CSI Driver**

---

## **3. How each plugin connects to backend**

You must know the **connection model**:

| Plugin               | Connection Mechanism       |
| -------------------- | -------------------------- |
| NFS                  | TCP-based network mount    |
| iSCSI                | Block-level SCSI over IP   |
| CephFS               | Distributed filesystem     |
| RBD                  | Block storage from Ceph    |
| glusterfs            | Scale-out filesystem       |
| fc                   | Fibre channel block device |
| azureFile            | SMB 3.0 share              |
| awsElasticBlockStore | EBS block device attach    |
| azureDisk            | Azure-managed disk         |
| gcePersistentDisk    | GCP persistent disk        |

---

## **4. Node-level requirements**

Most PV plugins require:

* Packages installed on node
* Kernel modules
* Network routes
* Storage drivers

Example:
iSCSI plugin needs:

```
iscsid systemd service
open-iscsi package
```

---

## **5. Access Modes per plugin**

Each plugin supports different access modes:

* ReadWriteOnce (RWO)
* ReadWriteMany (RWX)
* ReadOnlyMany (ROX)

Examples:

| Plugin    | RWX?              |
| --------- | ----------------- |
| NFS       | Yes               |
| CephFS    | Yes               |
| GlusterFS | Yes               |
| awsEBS    | **No** (RWO only) |
| AzureDisk | **No** (RWO only) |
| GCE PD    | **No** (RWO only) |

This is **critical** for architecture and interviews.

---

## **6. Performance differences**

You must know:

* Block storage → EBS, GCE PD, iSCSI, fc → high performance
* Shared storage → NFS, AzureFile → slower
* Distributed storage → CephFS/GlusterFS → scalable but CPU-heavy

---

## **7. Data Persistence Lifecycle**

You must understand:

* PV plugins mount **backend storage directly**, so:

  * Pod restart → data persists
  * Node restart → may require reattachment
  * Volume deletion → might destroy backend storage (depends)

---

## **8. Security considerations**

Each plugin has specific security requirements:

* NFS: no encryption, open file permissions
* AzureFile: SMB credentials (secret)
* CephFS/RBD: access keyrings (secret)
* iSCSI: CHAP authentication
* fc: WWNs, zoning required
* EBS: full node access to disk

---

## **9. Real-world use cases**

You must know which plugin is used where:

* **On-prem**: NFS, iSCSI, CephFS, FC
* **Cloud**: EBS/PD/AzureDisk
* **Scalable apps**: CephFS, GlusterFS
* **Legacy apps**: iSCSI, FC, NFS

---

# **Concepts You Must Master**

Here is the **master list** of concepts required to truly understand Persistent Volume Plugins:

### **1. Volume lifecycle (backend vs Pod lifecycle)**

Understand how backend managed separately from Pod.

### **2. Device attachment vs filesystem mount**

Block devices require:

* Attach → Format → Mount
* Detach → Unmount

Plugins automate these.

### **3. Mount options and fsGroup**

Know how to configure:

```yaml
mountOptions:
  - noatime
  - rw
```

### **4. NodeAffinity constraints for storage**

EBS disks restricted to same AZ → Pods must use nodeAffinity.

### **5. Static provisioning vs dynamic**

PV plugins often require **static provisioning**, unlike PVC dynamic provisioning.

### **6. CSI migration**

In-tree plugin replaced by CSI drivers — important for interviews.

### **7. Failover behavior**

NFS can failover gracefully.
EBS/GCE PD might fail on node/zone change.

### **8. StorageClasses**

When using PVC/PV around these plugins.

### **9. SELinux challenges**

Especially for NFS mounts.

### **10. Encryption**

* EBS, GCE PD, Azure Disk → encrypted at rest
* NFS, CephFS → depends on backend config

---

# **Interview Questions You Must Handle**

### **Beginner**

1. What are persistent volume plugins?
2. Name commonly used persistent volume plugins.
3. How is awsElasticBlockStore different from NFS?
4. What access modes are supported by EBS?

### **Intermediate**

1. Why are in-tree PV plugins deprecated?
2. What is CSI? Why was it introduced?
3. Explain how NFS mount works inside Kubernetes.
4. What node-level configurations are required for iSCSI volumes?
5. Why can’t you mount an EBS volume on two Pods simultaneously?
6. Difference between block storage and network file storage?

### **Advanced**

1. How does Kubernetes handle attachment of cloud block volumes across nodes?
2. Explain how CSI migration works technically.
3. What happens when a pod using an iSCSI backend is rescheduled to another node?
4. Explain how RWX is achieved through CephFS or GlusterFS.
5. What are the challenges of using NFS for containerized workloads?
6. How would you debug a failing flexVolume mount?
7. What is the difference between RBD (Ceph) and CephFS?

---

# **Pitfalls & Real-World Failure Scenarios You Must Be Aware Of**

### **1. Plugin not installed on nodes**

Common with:

* iSCSI
* CephFS
* GlusterFS

Pod fails with:

```
MountVolume.SetUp failed: executable file not found
```

---

### **2. Storage backend availability issues**

NFS server down → Pod startup hangs.
iSCSI target unreachable → node timeouts.

---

### **3. Filesystem permissions issues**

* NFS mounted as nobody:nogroup
* fsGroup not applied
* App cannot write → crashloop

---

### **4. Node restart or reattachment failure**

Cloud block storage volumes attach VERY slowly on node reboot.
Pods stay in **ContainerCreating**.

---

### **5. Wrong access modes**

Trying RWX on EBS → Pod fails to start.

---

### **6. AZ/Zone mismatch (Cloud block storage)**

EBS in `us-east-1a`
Node in `us-east-1b`
→ Volume cannot attach.

---

### **7. FlexVolume failures due to broken scripts**

FlexVolume uses external scripts → prone to:

* permission issues
* missing binaries
* wrong environment

---

### **8. Performance degradation**

NFS + high IOPS workload → slow
CephFS with small files + SSD mismatch → high latency
iSCSI without multipathing → single point bottleneck

---

### **9. DHCP or network changes break mounts**

Especially for:

* NFS
* iSCSI
* Ceph

---

### **10. Volume stuck in "Terminating"**

Pods using PV plugins can block termination until backend unmount operation completes.

---

# **Real-World Scenarios You Should Understand**

### **1. Running databases with block storage (EBS, PD, AzureDisk)**

You must know:

* RWO volumes
* Node affinity
* Failover impact
* Data consistency constraints

---

### **2. On-prem shared storage with NFS**

Used for:

* CI/CD artifacts
* Prometheus TSDB
* Logging
* File uploads

---

### **3. Using CephFS for multi-tenant app storage**

RWX requirement → CephFS perfect.

---

### **4. Using iSCSI for legacy enterprise apps**

High-performance block storage for custom workloads.

---

### **5. FlexVolume in old clusters**

Used when CSI wasn’t available.

---
