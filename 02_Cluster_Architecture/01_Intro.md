![](../images/01.png)


## 🧠 **CONTROL PLANE COMPONENTS**

### **1️⃣ etcd**

* etcd is a **highly available key–value database** that stores **all cluster data** — like pod status, config, secrets, and metadata.
* It acts as the **single source of truth** for Kubernetes.
* Every cluster change (Pod creation, deletion, scaling) is **persisted in etcd**.
* Only the **API Server interacts directly** with etcd.
* Supports **leader election** and **data replication** for HA setups.
* Uses **Raft consensus algorithm** to maintain consistency across nodes.
* If etcd fails, Kubernetes **loses its state memory**, even though nodes may still be running.

---

### **2️⃣ Kube API Server**

* The **Kubernetes front-end and brain** — all components and users talk to it.
* It handles **REST API requests** from kubectl, controllers, and other components.
* Performs **authentication, authorization, and validation** of requests.
* **Acts as a bridge** between etcd (data layer) and cluster components.
* Stores all cluster state and configuration data **in etcd**.
* Every CRUD operation on Kubernetes objects goes through the API Server.
* It ensures consistent and secure communication across the cluster.

---

### **3️⃣ Controller Manager**

* The **automation brain** of Kubernetes — ensures the actual cluster state matches the desired state.
* It runs various **controllers** like Node Controller, ReplicaSet Controller, Endpoint Controller, etc.
* Constantly **monitors the cluster state via API Server**.
* If something deviates (like a Pod crash), it takes corrective action automatically.
* Example: If 3 replicas are desired but 1 fails, it **creates a new Pod**.
* Acts like a **cluster autopilot**, maintaining stability and reliability.
* Operates in a **continuous reconciliation loop**.

---

### **4️⃣ Kube Scheduler**

* The **component responsible for assigning Pods to Nodes**.
* It watches for **unscheduled Pods** and picks the best node for them.
* Uses factors like **CPU, memory, taints, affinity/anti-affinity, and topology** to decide placement.
* Ensures **optimal workload distribution** and resource utilization.
* Avoids scheduling Pods on **unhealthy or overused nodes**.
* Once a decision is made, it updates the Pod spec in the API Server.
* In short, it decides **“where each Pod should run.”**

---

## ⚙️ **WORKER NODE COMPONENTS**

### **1️⃣ Kubelet**

* The **primary node agent** running on every worker node.
* It ensures containers described in the PodSpecs are **running and healthy**.
* Watches for Pod assignments from the API Server and starts them using the runtime (like containerd).
* Continuously **monitors Pod status and reports** back to the API Server.
* Performs **liveness/readiness probe checks** and restarts failed containers if needed.
* Doesn’t manage containers not created by Kubernetes.
* Acts as a **bridge between control plane and container runtime** on each node.

---

### **2️⃣ Kube-Proxy**

* The **networking component** that handles **Service-based communication** within the cluster.
* Ensures traffic sent to a Service (ClusterIP, NodePort, LoadBalancer) reaches the correct backend Pod.
* Implements **load balancing and network routing** using iptables or IPVS rules.
* Runs on every node and watches for **Service and Endpoint updates** from the API Server.
* Automatically updates routing rules when Pods are added or removed.
* Works at **Layer 4 (TCP/UDP)** for in-cluster traffic distribution.
* Without kube-proxy, **Service-to-Pod communication would fail.**

---

### **3️⃣ Container Runtime**

* The software responsible for **running containers** on each node.
* Examples: **containerd**, **CRI-O**, or older **Docker (deprecated)**.
* Kubelet communicates with it through the **Container Runtime Interface (CRI)**.
* It pulls container images, starts containers, and manages their lifecycle.
* Provides container-level **isolation, logging, and networking setup**.
* Works behind the scenes — Kubelet only instructs, runtime executes.
* Without it, Kubernetes can’t actually run any Pods.

---

✅ **Quick Recap Table:**

| Layer             | Component          | Core Responsibility                       |
| ----------------- | ------------------ | ----------------------------------------- |
| **Control Plane** | etcd               | Stores cluster state & configuration data |
| **Control Plane** | API Server         | Central communication hub & entry point   |
| **Control Plane** | Controller Manager | Maintains desired vs actual state         |
| **Control Plane** | Scheduler          | Assigns Pods to Nodes                     |
| **Worker Node**   | Kubelet            | Ensures Pods are running properly         |
| **Worker Node**   | Kube-Proxy         | Manages network routing & Service traffic |
| **Worker Node**   | Container Runtime  | Runs and manages containers               |

---
