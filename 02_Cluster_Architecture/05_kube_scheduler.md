![](../images/01.png)

# Kube Scheduler: 
- `kube-scheduler` is responsible for scheduling pods on nodes.
- The `kube-scheduler` is only responsible for deciding which pod goes on which node. It doesn't actually place the pod on the nodes, that's the job of the `kubelet`.
- It ensures that workloads are distributed efficiently and fairly across nodes — maintaining optimal performance, resource balance, and availability.

## What `kube-scheduler` Does

* The kube-scheduler **monitors the API server** for any **new Pods** that are created but **don’t yet have a node assigned** (`.spec.nodeName` is empty).
* Once detected, the scheduler:  
  1. **Filters (shortlists)** nodes that can run that Pod.
  2. **Scores (ranks)** the shortlisted nodes based on predefined policies and metrics.
  3. **Selects the best node** (highest score) for that Pod.
  4. **Binds** the Pod to the chosen node by updating the API Server (via a Binding object).

> In simple words — kube-scheduler = *Pod Placement Brain* 🧩

---

## Working of kube-scheduler

Let’s go through its working in **logical stages** 
### **1. Watch Phase – Detect Unscheduled Pods**

* kube-scheduler **continuously watches** the kube-apiserver for **Pods without node assignments.**
* Whenever a new Pod object appears in etcd (via API Server) with no `.spec.nodeName`, it gets picked up for scheduling.

---

### **2. Filtering Phase – Find Eligible Nodes**

* The scheduler **filters** all cluster nodes to find **those that meet Pod requirements**.
* It uses a set of **predicates (rules)** to check compatibility.

#### Common Filtering Rules:

| Filter Type                  | Description                                         |
| ---------------------------- | --------------------------------------------------- |
| **Node Unschedulable**       | Skips nodes marked as `unschedulable`.              |
| **PodFitsResources**         | Ensures node has enough CPU, memory, etc.           |
| **PodFitsHostPorts**         | Checks if requested ports are available.            |
| **PodFitsHost**              | If Pod has `nodeName` specified, matches that node. |
| **PodToleratesNodeTaints**   | Ensures Pod tolerates node taints.                  |
| **NodeAffinity/PodAffinity** | Matches Pods to nodes based on affinity rules.      |
| **Volume Zone Fit**          | Ensures Pod volumes are in the same zone as node.   |

After this stage, the scheduler has a **Filtered Node List** = nodes that *can* run the Pod.

---

### **3. Scoring Phase – Rank Suitable Nodes**

* Once the eligible nodes are known, kube-scheduler **scores each node** on various parameters like:
  * Available CPU/Memory
  * Network locality (Pod-to-Pod communication)
  * Volume proximity
  * Resource balancing
  * User-defined policies (e.g., prefer zone A over B)

> Each node gets a **score (0–100)** and the **highest-scoring node wins.**

#### Example Scoring Policies:

| Policy                         | Meaning                                       |
| ------------------------------ | --------------------------------------------- |
| **LeastRequestedPriority**     | Prefers nodes with the least used CPU/memory. |
| **BalancedResourceAllocation** | Prefers nodes with balanced CPU/mem usage.    |
| **SelectorSpreadPriority**     | Spreads Pods evenly across zones.             |
| **TaintTolerationPriority**    | Scores nodes based on taint toleration.       |

---

### **4. Binding Phase – Assign Pod to Node**
* Once the best node is chosen:
  * kube-scheduler creates a **Binding object** and sends it to the **API Server**.
  * The API Server **updates the Pod’s `.spec.nodeName`** field.
  * The **kubelet on that node** detects the Pod assignment and **starts the container creation** process.

> So the scheduler **only decides where to run**, not **how to run** — that’s kubelet’s job.

---

### **5. Continuous Monitoring**
* Scheduler continuously monitors:
  * Node resource availability.
  * Pod scheduling success/failure.
  * Cluster topology changes (node joins/leaves).
* It recalculates scores dynamically as the cluster state changes.

---

## Example – How It Works in Practice

Let’s say you create a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx
      resources:
        requests:
          cpu: "200m"
          memory: "256Mi"
```

### Flow:

1. **You run:** `kubectl apply -f nginx-pod.yaml`
2. **kube-apiserver** validates & stores the Pod object in etcd.
3. **kube-scheduler** sees this unscheduled Pod.
4. It filters out nodes that lack 200m CPU or 256Mi memory.
5. Scores remaining nodes (say Node2 gets highest score).
6. Creates a binding → Pod is assigned to **Node2**.
7. **kubelet on Node2** starts pulling image and running Pod.

---

## Scheduling Policies and Plugins

Modern kube-scheduler (v1.19+) is **extensible and pluggable** through the **Scheduling Framework**, which has **Plugins** for each stage:

| Stage     | Plugin Type    | Purpose                            |
| --------- | -------------- | ---------------------------------- |
| Queue     | QueueSort      | Defines Pod scheduling order.      |
| Filtering | Filter         | Filters out unsuitable nodes.      |
| Scoring   | Score          | Scores the feasible nodes.         |
| Binding   | Bind           | Performs the actual binding.       |
| Post      | Reserve/Permit | Hooks for custom scheduling logic. |

You can **customize or write your own plugins** for specific business logic.

---

## Key Features of kube-scheduler

* **Automated Scheduling:** No manual Pod-to-node assignment required.
* **Extensible via Plugins:** Custom logic for filters/scoring.
* **Respects Node and Pod Affinity/Anti-Affinity rules.**
* **Considers Resource Requests, Limits, and Taints/Tolerations.**
* **Optimizes cluster performance by load balancing.**
* **Reschedules pending Pods if resources free up (in newer versions).**
* **High availability** — multiple schedulers can be run (only one active leader).

---

## Components in kube-scheduler Architecture

| Component                | Function                                                           |
| ------------------------ | ------------------------------------------------------------------ |
| **Informer/Cache**       | Watches API Server for unscheduled Pods.                           |
| **Scheduling Queue**     | Maintains pending Pods waiting to be scheduled.                    |
| **Scheduling Algorithm** | Core logic – filtering, scoring, ranking.                          |
| **Binder**               | Sends binding info back to API Server.                             |
| **Extender**             | Optional external scoring/filtering logic (for custom schedulers). |

---

## High Availability (HA)

* You can deploy **multiple kube-scheduler instances** for redundancy.
* All instances connect to the **same API server**.
* **Leader election** ensures only one is active at a time.
* If active scheduler fails, another takes over automatically.

---

## Troubleshooting Commands

**Check scheduler status:**
```bash
kubectl get componentstatus
```
**View scheduler logs:**
```bash
journalctl -u kube-scheduler -f
```
**Health check:**
```bash
curl -k https://localhost:10251/healthz
```

**View leader election info:**
```bash
kubectl get endpoints kube-scheduler -n kube-system -o yaml
```

---

## Summary Table

| Property                 | Detail                              |
| ------------------------ | ----------------------------------- |
| **Component Type**       | Control Plane                       |
| **Main Role**            | Assign Pods to Nodes                |
| **Input Source**         | kube-apiserver (unscheduled Pods)   |
| **Output**               | Pod bound to node                   |
| **Default Port**         | 10251 (non-secure) / 10259 (secure) |
| **Leader Election**      | Yes                                 |
| **Communication**        | Talks to API Server only            |
| **Stateless**            | Yes                                 |
| **Scoring Range**        | 0–100                               |
| **Extensible Framework** | Yes (Plugins & Extenders)           |

---

## Mnemonic Tip to Remember Scheduler Flow

> **“Watch → Filter → Score → Bind → Notify”**

1. **Watch** for unscheduled Pods
2. **Filter** nodes that can’t host
3. **Score** feasible ones
4. **Bind** Pod to best node
5. **Notify** kubelet (via API server update)

---



