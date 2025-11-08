![](../images/01.png)

# Kubelet in Kubernetes

## 1. What is Kubelet?

* **Kubelet** is a node-level agent in Kubernetes that ensures containers defined in **PodSpecs** are running and healthy.
* It acts as a **bridge between the node and the control plane (API Server)**.
* Automatically handles **pod lifecycle**, **health checks**, **log reporting**, and **garbage collection**.

* It interacts with:

  * **API Server** → to receive PodSpecs and report status.
  * **Container Runtime (CRI)** → to start, stop, and monitor containers.
  * **cAdvisor** → to collect resource metrics.
  * **Volume plugins** → to manage storage.

---

## 2. Kubelet Workflow (Step-by-Step)

1. **Registers the node** with the API Server.
2. **Watches for PodSpecs** assigned by the Scheduler.
3. **Starts containers** through the Container Runtime Interface (CRI).
4. **Mounts volumes** as needed.
5. **Performs health checks** on containers.
6. **Reports status** of Pods and Node to API Server.
7. **Collects metrics** using cAdvisor.
8. **Handles node pressure & evictions** if resources are low.

---

## 3. Kubelet Architecture Components

| Component                                | Function                                                 |
| ---------------------------------------- | -------------------------------------------------------- |
| **Pod Manager**                          | Manages Pod lifecycle.                                   |
| **Container Runtime Interface (CRI)**    | Talks to Docker, containerd, or CRI-O to run containers. |
| **Volume Manager**                       | Mounts/unmounts persistent volumes.                      |
| **cAdvisor**                             | Collects CPU, memory, and network metrics.               |
| **Status Manager**                       | Reports Node and Pod status to API Server.               |
| **Probe Manager**                        | Runs liveness, readiness, and startup probes.            |
| **PLEG (Pod Lifecycle Event Generator)** | Detects state changes in Pods/containers.                |
| **Image Manager**                        | Ensures required container images are present.           |

---

## 4. Configuration Files and Flags

| File/Flag                      | Purpose                     | Default / Manual                      |
| ------------------------------ | --------------------------- | ------------------------------------- |
| `/var/lib/kubelet/config.yaml` | Main configuration file     | 🧑‍💻 Manual setup                    |
| `--kubeconfig`                 | API Server credentials      | 🧑‍💻 Manual                          |
| `--pod-manifest-path`          | Static pod location         | 🧑‍💻 Manual                          |
| `--register-node`              | Auto node registration      | ✅ Automatic (default)                 |
| `--fail-swap-on`               | Prevent start if swap is on | 🧑‍💻 Manual                          |
| `--read-only-port=0`           | Disable insecure port       | 🧑‍💻 Manual (security best practice) |

---

## 5. Types of Pods Managed by Kubelet

| Type            | Created By                 | Automatic / Manual                             | Notes                                |
| --------------- | -------------------------- | ---------------------------------------------- | ------------------------------------ |
| **Static Pods** | File placed on node        | 🧑‍💻 Manual placement, ✅ Automatic management | Common for control plane components. |
| **Mirror Pods** | Auto-created by Kubelet    | ✅ Automatic                                    | API representation of static pods.   |
| **Normal Pods** | Via Scheduler (API Server) | ✅ Automatic                                    | Typical application pods.            |

---

## 6. Probes and Health Checks

| Probe               | Purpose                                | Automatic / Manual                 | Behavior                                         |
| ------------------- | -------------------------------------- | ---------------------------------- | ------------------------------------------------ |
| **Liveness Probe**  | Checks if container is alive           | ✅ Auto-run, 🧑‍💻 Defined manually | Restarts container if it fails.                  |
| **Readiness Probe** | Checks if container can serve requests | ✅ Auto-run, 🧑‍💻 Defined manually | Removes Pod from service endpoints if not ready. |
| **Startup Probe**   | Ensures container has started properly | ✅ Auto-run, 🧑‍💻 Defined manually | Prevents premature restart during startup.       |

---

## 7. Kubelet Communication Flow

| Direction                   | From → To             | Purpose          |
| --------------------------- | --------------------- | ---------------- |
| API Server → Kubelet        | Sends PodSpecs        | Assign workloads |
| Kubelet → API Server        | Sends Pod/Node status | Health reporting |
| Kubelet → Container Runtime | Local communication   | Run containers   |
| Kubelet → cAdvisor          | Local                 | Collect metrics  |

**Ports:**

* Secure port: `10250`
* Deprecated insecure port: `10255`

---

## 8. Security in Kubelet

| Feature                         | Description                      | Manual / Automatic                                       |
| ------------------------------- | -------------------------------- | -------------------------------------------------------- |
| TLS encryption                  | Secure Kubelet–API communication | ✅ Automatic in kubeadm clusters                          |
| Client certs or tokens          | Used for authentication          | ✅ Auto-generated (kubeadm) / 🧑‍💻 Manual (custom setup) |
| RBAC authorization              | Limits API permissions           | ✅ Automatic enforcement                                  |
| Disable insecure read-only port | Prevents unauthenticated access  | 🧑‍💻 Manual (`--read-only-port=0`)                      |

---

## 9. Automatic vs Manual Tasks (Full Overview)

| Category                | Automatic       | Manual                               | Explanation                           |
| ----------------------- | --------------- | ------------------------------------ | ------------------------------------- |
| Node registration       | ✅               | 🧑‍💻 (if `--register-node=false`)   | Kubelet usually self-registers.       |
| Running scheduled pods  | ✅               | -                                    | Kubelet auto-runs assigned Pods.      |
| Static pod execution    | ✅               | 🧑‍💻                                | Auto-managed after you drop manifest. |
| Image pulling           | ✅               | 🧑‍💻 Define pull policy.            |                                       |
| Volume mounting         | ✅               | 🧑‍💻 Define in PodSpec.             |                                       |
| Health probes           | ✅               | 🧑‍💻 Define in PodSpec.             |                                       |
| Status reporting        | ✅               | -                                    | Periodic API updates.                 |
| Resource monitoring     | ✅               | -                                    | Handled via cAdvisor.                 |
| Evicting under pressure | ✅               | 🧑‍💻 Optional thresholds.           |                                       |
| Logging                 | ✅               | 🧑‍💻 Central aggregation config.    |                                       |
| Metrics export          | ✅               | 🧑‍💻 Configure Prometheus scraping. |                                       |
| Certificates            | ✅ (auto-rotate) | 🧑‍💻 Custom setups.                 |                                       |
| Config reading          | ✅               | 🧑‍💻 Editing flags/config.yaml.     |                                       |

---

## 10. Node Pressure & Eviction Handling

* Kubelet continuously checks node resource usage.
* If **memory/disk** crosses thresholds:

  * It marks node condition (`MemoryPressure`, `DiskPressure`).
  * Scheduler avoids placing new pods there.
  * Kubelet **evicts non-critical pods** automatically.

**Manual tuning:** You can set eviction thresholds using flags like
`--eviction-hard` or `--eviction-soft`.

---

## 11. Logs and Troubleshooting

**Commands:**

```bash
systemctl status kubelet
journalctl -u kubelet -f
cat /var/lib/kubelet/config.yaml
```

**Common issues:**

| Issue                            | Cause                              | Fix                                         |
| -------------------------------- | ---------------------------------- | ------------------------------------------- |
| Node `NotReady`                  | Kubelet stopped or API unreachable | Restart service or fix connectivity.        |
| Pod stuck in `ContainerCreating` | Image pull/volume issue            | Check container runtime logs.               |
| Swap enabled                     | Kubelet won’t start                | Disable swap or use `--fail-swap-on=false`. |

---

## 12. Real-World Use Cases

| Use Case                      | Automatic / Manual        | Explanation                                                   |
| ----------------------------- | ------------------------- | ------------------------------------------------------------- |
| **Pod Lifecycle Management**  | ✅                         | Kubelet starts, stops, and restarts containers as per policy. |
| **Cluster Health Monitoring** | ✅                         | Reports Node and Pod status to API Server.                    |
| **Static Pod Management**     | 🧑‍💻 Setup + ✅ Execution | Used for control-plane components.                            |
| **Node Self-Registration**    | ✅                         | Auto-registers with cluster.                                  |
| **Resource Enforcement**      | ✅                         | Applies resource limits (CPU/memory).                         |
| **Metrics Export**            | ✅                         | Provides metrics via cAdvisor.                                |

---

## 13. Common Interview Questions

| Question                             | Short, Strong Answer                                                      |
| ------------------------------------ | ------------------------------------------------------------------------- |
| What is Kubelet?                     | Node agent ensuring containers in Pods are running and healthy.           |
| How does Kubelet get PodSpecs?       | From API Server (via Scheduler) or from static pod manifests.             |
| What are static and mirror pods?     | Static pods run directly on nodes; mirror pods are their API reflections. |
| What is PLEG?                        | Pod Lifecycle Event Generator — detects state changes in pods.            |
| Which port does Kubelet use?         | 10250 (secure).                                                           |
| How does Kubelet ensure security?    | TLS, client certs, RBAC, disable read-only port.                          |
| Can Kubelet work without API Server? | Yes, for static pods.                                                     |
| What happens if Kubelet dies?        | Node becomes “NotReady”; pods may get rescheduled.                        |
| How are probes handled?              | Defined manually, executed automatically by Kubelet.                      |

---

## 14. Key One-Liners (Quick Revision)

* Kubelet is **node-scoped**, not cluster-scoped.
* It **does not manage containers** created outside Kubernetes.
* Communicates via **gRPC** with container runtimes.
* Control-plane components (like API Server) are **static pods** managed by Kubelet.
* Works with **cAdvisor** to expose metrics.
* Handles **node pressure eviction** automatically.

---

## 15. Real-World Example (Auto vs Manual)

Let’s deploy a Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
    - name: nginx
      image: nginx:latest
      livenessProbe:
        httpGet:
          path: /
          port: 80
```

### Manual actions you do:

* Define YAML.
* Add liveness probe.
* Apply via `kubectl apply`.

### Automatic actions by Kubelet:

* Registers node (if needed).
* Receives PodSpec from API Server.
* Kubelet then instructs the container runtime (like containerd, Docker, or CRI-O) to:
    - Pull the required image (e.g., nginx:latest) from the container registry.
    - Create and start the container from that image.
* Runs liveness probe periodically.
* Restarts container if probe fails.
* Reports status to API Server continuously.

---

# Final Summary Table

| Function            | Description                 | Automatic | Manual                  |
| ------------------- | --------------------------- | --------- | ----------------------- |
| Node Registration   | Registers with API Server   | ✅         | 🧑‍💻 (custom setup)    |
| Pod Execution       | Runs containers per PodSpec | ✅         | -                       |
| Static Pod Setup    | Manages local YAML pods     | ✅         | 🧑‍💻 Place YAML        |
| Volume Management   | Mounts storage volumes      | ✅         | 🧑‍💻 Define volumes    |
| Health Probes       | Runs checks                 | ✅         | 🧑‍💻 Define in PodSpec |
| Status Reporting    | Sends node/pod state        | ✅         | -                       |
| Resource Monitoring | Tracks metrics              | ✅         | -                       |
| Pod Eviction        | Under resource pressure     | ✅         | 🧑‍💻 Tune thresholds   |
| Security            | TLS & auth                  | ✅         | 🧑‍💻 Set flags         |
| Configuration       | Reads config.yaml           | ✅         | 🧑‍💻 Modify values     |

---

Here’s a **concise and clear summary** of the **Kubelet in Kubernetes**:

### **Kubelet Overview**

* **Kubelet** is an **agent** running on every Kubernetes node.
* It ensures that **containers described in PodSpecs are running and healthy**.
* It acts as a **bridge between the node and the control plane (API Server)**.

---

###**Automatic Tasks (Done by Kubelet Itself)**

1. **Pod Management:** Starts, stops, and restarts containers as per PodSpec.
2. **Health Monitoring:** Continuously checks liveness and readiness probes.
3. **Sync Loop:** Keeps node state consistent with desired cluster state.
4. **Log & Metrics Reporting:** Sends container status and resource usage to API Server.
5. **Garbage Collection:** Removes unused containers, images, and pod sandboxes automatically.
6. **Certificate Rotation:** Automatically rotates its TLS certificates for secure communication.
7. **CNI & CSI Plugins Integration:** Automatically interacts with networking and storage plugins once configured.

---

### **Manual or Admin-Controlled Tasks**

1. **Configuration Setup:** Installing and configuring kubelet (kubelet.service, flags, kubeconfig, etc.).
2. **Node Registration:** Manual node join command (`kubeadm join`) or bootstrap config.
3. **PodSpec Creation:** Defining pods, probes, and volumes via manifests or API.
4. **Custom Probes & Resource Limits:** Must be defined by the user in PodSpec.
5. **Static Pods Setup (optional):** Admins manually place YAMLs in `/etc/kubernetes/manifests/`.

---

### **Key Facts for Interviews**

* Kubelet talks to the **API Server** via REST calls using certificates.
* It uses **container runtime (Docker, containerd, CRI-O)** to manage containers.
* It maintains **Pod Lifecycle** and updates **NodeStatus** to API Server.
* If the **API Server is down**, Kubelet still manages existing pods locally.
* **Kubelet logs** are crucial for debugging pod scheduling or node-level issues.

---

### **Real-World Use Cases**

* Auto-restart of crashed containers.
* Health monitoring of microservices via probes.
* Managing system pods like kube-proxy and CoreDNS as static pods.
* Automatic cleanup of terminated pods or images to save disk space.

---

✅ **In short:**
Kubelet is the **heart of every node**, performing 90% of tasks **automatically** once configured.
Your role as a DevOps Engineer is to **define specs, flags, and policies manually**, while Kubelet does the heavy lifting of execution, health checking, monitoring, and reporting.

---
