![](../images/01.png)
## Pre-requisite:
### 1. Controllers :
- In kubernetes terms, a controller are the processes that continuously monitors the state of the components within the system and works towards bringing the whole system to the desired functioning state.
- They are the brain behind orchestration.
- They are responsible for noticing and responding when nodes, containers or endpoints goes down. The controllers makes decisions to bring up new containers in such cases.
- Logically, each controller is a separate process, but to reduce complexity, they are all compiled into a single binary and run in a single process known as the **Kubernetes Controller Manager** or `kube-controller-manager`.
- There are many different types of controllers. Some examples of them are:
    - **Node controller**: Responsible for noticing and responding when nodes go down.
    - **Job controller**: Watches for Job objects that represent one-off tasks, then creates Pods to run those tasks to completion.
    - **EndpointSlice controller**: Populates EndpointSlice objects (to provide a link between Services and Pods).
    - **ServiceAccount controller**: Create default ServiceAccounts for new namespaces.

#### Kube Controller Manager:  `kube-controller-manager` manages various controllers in kubernetes.

## **Main types of controllers**

### 1️⃣ **Node Controller**

* Monitors the health of nodes.
* Detects and marks nodes as **NotReady** if they stop responding.
* If a node remains unreachable beyond a timeout (default ~5 minutes), it:

  * Evicts pods running on that node.
  * Triggers rescheduling of pods to other healthy nodes.

---

### 2️⃣ **Replication Controller (and ReplicaSet Controller)**

* Ensures that the desired number of **Pod replicas** are running.
* If there are fewer pods than desired → creates new pods.
* If more → deletes the extra pods.

---

### 3️⃣ **Deployment Controller**

* Manages **rolling updates** and **rollbacks** for Deployments.
* Ensures zero downtime updates.
* Controls scaling and versioning of application pods.

---

### 4️⃣ **DaemonSet Controller**

* Ensures that **a specific pod runs on all (or selected) nodes**.
* Used for cluster-level agents like log collectors, monitoring daemons, etc.
  (e.g., `kube-proxy`, `fluentd`)

---

### 5️⃣ **Job & CronJob Controllers**

* **Job Controller:** Ensures pods for batch jobs complete successfully.
* **CronJob Controller:** Manages periodic (scheduled) job creation based on cron expressions.

---

### 6️⃣ **Namespace Controller**

* Watches for namespace deletion events.
* Ensures that when a namespace is deleted, **all objects inside it (pods, services, secrets)** are also removed.

---

### 7️⃣ **ServiceAccount & Token Controllers**

* Automatically creates **default ServiceAccounts** and **API tokens** for pods to authenticate with the API server.

---

### 8️⃣ **ResourceQuota Controller**

* Enforces **resource usage limits** (CPU, memory, storage, etc.) per namespace.
* Ensures users/projects don’t consume more than their assigned quota.

---

### 9️⃣ **EndpointSlice / Service Controller**

* Maintains **network service endpoints** to route traffic correctly between pods.

---

### 🔟 **Volume Controller**

* Manages the lifecycle of **PersistentVolume (PV)** and **PersistentVolumeClaim (PVC)** objects.
* Handles volume binding, attachment, and reclamation when pods or claims are deleted.

---

## **Architecture & Deployment**

* Runs as a **single process (`kube-controller-manager`)** on the **control-plane node**.
* Manages multiple logical controllers internally (as threads/goroutines).
* Configuration path (for kubeadm setups):
  `/etc/kubernetes/manifests/kube-controller-manager.yaml`
* Default communication:

  * With kube-apiserver → over HTTPS
  * No direct communication with etcd or kubelets.

---

## **High Availability (HA) Behavior**

* In multi-master setups, multiple kube-controller-manager instances run.
* However, **only one acts as the leader** at a time.
* Leader election mechanism ensures:
  * Only one instance actively controls the cluster.
  * Others remain in standby and automatically take over if the leader fails.

---

## **Key Flags & Configurations**

| Flag                          | Description                                            |
| ----------------------------- | ------------------------------------------------------ |
| `--controllers`               | Specify which controllers to enable/disable.           |
| `--leader-elect`              | Enables leader election for HA setups.                 |
| `--node-monitor-period`       | Interval to check node health.                         |
| `--pod-eviction-timeout`      | Time after which pods are evicted from failed nodes.   |
| `--cluster-signing-cert-file` | Used for generating certificates for service accounts. |

---

## **Interaction Example**

When you run:

```bash
kubectl scale deployment nginx-deployment --replicas=4
```

Here’s what happens:

1. **kubectl** sends the update to the **API Server**.
2. The **Deployment Controller** (inside kube-controller-manager) notices the change.
3. It compares:

   * Desired replicas: 4
   * Actual replicas: 2
4. It creates two more pods to match the desired count.
5. Once created, etcd is updated through the API Server.

---

## **Monitoring and Troubleshooting**

**Check component health:**

```bash
kubectl get componentstatus
```

**View controller logs:**

```bash
journalctl -u kube-controller-manager -f
```

**Inspect pod (kubeadm clusters):**

```bash
kubectl -n kube-system get pods | grep controller
kubectl -n kube-system logs <controller-manager-pod-name>
```

---

## **Mnemonic Recap**

> **“Kube Controller Manager = C.H.E.N.N.D.R.S.”**

Each letter helps recall major controllers:

| Letter | Stands For                        | Function                        |
| ------ | --------------------------------- | ------------------------------- |
| **C**  | CronJob Controller                | Schedule periodic jobs          |
| **H**  | HorizontalPodAutoscaler           | Scale pods based on metrics     |
| **E**  | Endpoint/Service Controller       | Maintain network endpoints      |
| **N**  | Node Controller                   | Node health and eviction        |
| **N**  | Namespace Controller              | Cleanup namespace objects       |
| **D**  | DaemonSet Controller              | Run pods on all nodes           |
| **R**  | ReplicaSet/Replication Controller | Maintain pod replicas           |
| **S**  | ServiceAccount Controller         | Manage default service accounts |

---

