
![](../images/01.png)

# **Kube-Proxy in Kubernetes**
## 1. **What is Kube-Proxy?**

* **Kube-Proxy** is a **networking component** that runs on **each node** in a Kubernetes cluster.
* It is responsible for **implementing the Service abstraction** — meaning it makes sure that traffic sent to a Kubernetes **Service** reaches one of the healthy backend **Pods**.
* It handles **load balancing, network routing, and service discovery** at the node level.
* It watches the **API Server** for `Service` and `Endpoint` objects and updates routing rules accordingly.

---

## 2. **Kube-Proxy Workflow (Step-by-Step)**

1. **Watches API Server** for new or updated `Service` and `Endpoint` objects.
2. **Creates routing rules** on the node (iptables, IPVS, or userspace mode).
3. When a client sends a request to the **Service** (like `ClusterIP`, `NodePort`, or `LoadBalancer`),
   kube-proxy redirects it to one of the backend Pods.
4. **Performs simple round-robin load balancing** (in IPVS mode).
5. **Updates rules dynamically** as Pods scale up/down or become unavailable.
6. Ensures **high availability** — if a Pod dies, traffic automatically redirects to healthy Pods.

---

## 3. **Kube-Proxy Architecture Modes**

| Mode               | Description                                                               | Performance | Default                         |
| ------------------ | ------------------------------------------------------------------------- | ----------- | ------------------------------- |
| **Userspace Mode** | Old mode; kube-proxy itself handled forwarding in userspace.              | Slow        | ❌ Deprecated                    |
| **iptables Mode**  | Uses Linux `iptables` rules for routing packets directly in kernel space. | Faster      | ✅ Default in most clusters      |
| **IPVS Mode**      | Uses Linux IP Virtual Server for connection-based load balancing.         | Fastest     | ⚙️ Preferred for large clusters |

---

## 4. **Kube-Proxy Architecture Components**

| Component           | Description                                                    |
| ------------------- | -------------------------------------------------------------- |
| **Service Watcher** | Watches `Service` and `EndpointSlice` updates from API Server. |
| **Rule Syncer**     | Creates/deletes routing rules using iptables or IPVS.          |
| **Load Balancer**   | Distributes incoming traffic to Pods.                          |
| **Health Monitor**  | Removes unreachable endpoints from routing rules.              |

---

## 5. **Kube-Proxy Configuration and Files**

| File/Flag                         | Purpose                            | Default / Manual    |
| --------------------------------- | ---------------------------------- | ------------------- |
| `/var/lib/kube-proxy/config.conf` | Main kube-proxy configuration file | 🧑‍💻 Manual setup  |
| `--proxy-mode`                    | Defines mode: `iptables` or `ipvs` | ✅ Default: iptables |
| `--cluster-cidr`                  | Specifies cluster IP range         | 🧑‍💻 Manual        |
| `--metrics-bind-address`          | Exposes Prometheus metrics         | 🧑‍💻 Optional      |
| `--hostname-override`             | Override node hostname             | 🧑‍💻 Optional      |

---

## 6. **Kube-Proxy Communication Flow**

| Direction                     | From → To                          | Purpose                   |
| ----------------------------- | ---------------------------------- | ------------------------- |
| API Server → Kube-Proxy       | Sends Service and Endpoint updates | Keep routing info current |
| Client → Service (ClusterIP)  | Traffic enters the cluster         | Kube-Proxy routes it      |
| Kube-Proxy → Pod              | Forwards requests to backend Pod   | Load balancing            |
| Kube-Proxy → Metrics endpoint | Provides Prometheus data           | Monitoring                |

**Ports:**

* **Service ClusterIP Range:** e.g., `10.96.0.0/12`
* **Metrics Port:** `10249` (default)

---

## 7. **Automatic vs Manual Tasks**

| Task                       | Automatic   | Manual                     | Description                                  |
| -------------------------- | ----------- | -------------------------- | -------------------------------------------- |
| Watch Services & Endpoints | ✅           | -                          | Continuously watches API Server for updates. |
| Rule Creation              | ✅           | -                          | Automatically creates iptables/IPVS rules.   |
| Load Balancing             | ✅           | -                          | Auto-distributes traffic among Pods.         |
| Service Creation           | -           | 🧑‍💻                      | You define Service YAML.                     |
| Proxy Mode Selection       | ✅ (default) | 🧑‍💻 (override with flag) | Selects `iptables` or `ipvs` mode.           |
| Metrics Enablement         | -           | 🧑‍💻                      | Optional for monitoring tools.               |

---

## 8. **Kube-Proxy Workflow Example**

**Manual Actions (You Do):**

1. Create a Service YAML file:

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: webapp-service
   spec:
     selector:
       app: webapp
     ports:
     - port: 80
       targetPort: 8080
   ```
2. Apply it:

   ```bash
   kubectl apply -f service.yaml
   ```

**Automatic Actions by Kube-Proxy:**

* Detects the new Service and Endpoint objects from API Server.
* Updates routing rules (iptables/IPVS) on each node.
* Routes traffic sent to `ClusterIP` → to one of the backend Pods.
* If a Pod dies, kube-proxy removes its endpoint and updates rules instantly.

---

## 9. **Security in Kube-Proxy**

| Feature          | Description                              | Manual / Automatic       |
| ---------------- | ---------------------------------------- | ------------------------ |
| API Server Auth  | Uses kubeconfig for secure communication | ✅ Automatic              |
| RBAC Permissions | Restricts API access                     | ✅ Automatic              |
| TLS Certificates | Used for encryption                      | ✅ Auto-managed (kubeadm) |
| Metrics Security | Can be secured via HTTPS                 | 🧑‍💻 Manual setup       |

---

## 10. **Logs and Troubleshooting**

**Commands:**

```bash
systemctl status kube-proxy
journalctl -u kube-proxy -f
kubectl -n kube-system logs -l k8s-app=kube-proxy
```

**Common Issues:**

| Issue                 | Cause                     | Fix                     |
| --------------------- | ------------------------- | ----------------------- |
| Service not reachable | iptables/IPVS not updated | Restart kube-proxy      |
| ClusterIP unreachable | Wrong cluster CIDR        | Check kube-proxy config |
| Pod IP not listed     | Endpoint not created      | Check labels/selectors  |
| Slow load balancing   | Userspace mode enabled    | Switch to IPVS mode     |

---

## 11. **Real-World Use Cases**

| Use Case                    | Description                                                |
| --------------------------- | ---------------------------------------------------------- |
| **Internal Load Balancing** | Distributes traffic among backend Pods inside the cluster. |
| **Service Discovery**       | Enables pods to reach services via DNS names.              |
| **High Availability**       | Automatically reroutes traffic from failed Pods.           |
| **Scalable Networking**     | Efficient routing in large clusters (IPVS).                |

---

## 12. **Key Facts for Interviews**

* Kube-Proxy **implements Kubernetes Services** by managing node-level routing rules.
* Works at the **network layer (L4)** using kernel-level mechanisms (iptables/IPVS).
* Does **not perform ingress routing** (Ingress Controllers handle L7).
* For each new Service or Pod, kube-proxy updates local rules automatically.
* It **does not handle actual packet forwarding**; the Linux kernel does.
* Without kube-proxy, **Service ClusterIPs** won’t function.

---

## 13. **Common Interview Questions**

| Question                                              | Short Answer                                                              |
| ----------------------------------------------------- | ------------------------------------------------------------------------- |
| What is kube-proxy?                                   | A node-level component that routes Service traffic to Pods.               |
| Which modes are supported?                            | Userspace, iptables, and IPVS.                                            |
| What’s the difference between kube-proxy and Ingress? | kube-proxy handles L4 routing (TCP/UDP); Ingress handles L7 (HTTP/HTTPS). |
| Can kube-proxy run in IPVS mode?                      | Yes, it provides faster connection-based load balancing.                  |
| What happens when a Pod dies?                         | Kube-proxy removes its endpoint and updates routing rules automatically.  |
| Does kube-proxy manage external traffic?              | No, it manages internal Service traffic only.                             |

---

## 14. **Quick One-Liners for Revision**

* Kube-Proxy = **Network traffic manager for Kubernetes Services**.
* It **translates Service ClusterIP → Pod IPs** using iptables/IPVS rules.
* It **watches Services and Endpoints** from the API Server continuously.
* Operates at **Layer 4 (Transport Layer)** of the OSI model.
* **IPVS mode** is the most efficient and production-recommended.
* Without kube-proxy, **in-cluster communication via Services will fail**.

---

✅ **In short:**

> **Kube-Proxy is the traffic director of Kubernetes.**
> It automatically maintains the routing and load balancing rules on each node so that when a Service is accessed, the request reaches one of the healthy Pods seamlessly.

---

# Kubelet vs Kube-Proxy


| **Feature / Aspect**                     | **Kubelet**                                                                                                                                                        | **Kube-Proxy**                                                                                                                                         |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Definition**                           | Kubelet is the **node agent** that ensures Pods are running as per the PodSpec.                                                                                      | Kube-Proxy is the **networking component** that manages Service routing and load balancing.                                                              |
| **Primary Role**                         | Manages **Pod lifecycle** (creation, health checks, restarts, deletion).                                                                                             | Manages **network traffic routing** between Services and Pods.                                                                                           |
| **Runs On**                              | Every node (control plane and worker nodes).                                                                                                                         | Every node (primarily worker nodes).                                                                                                                     |
| **Layer of Operation (OSI Model)**       | Application layer (manages workloads).                                                                                                                               | Network/Transport layer (L3/L4 — handles TCP/UDP routing).                                                                                               |
| **Interaction With API Server**          | Watches for PodSpecs and Node objects from the API Server.                                                                                                           | Watches for Service and Endpoint objects from the API Server.                                                                                            |
| **Responsible For**                      | - Creating and monitoring Pods. <br> - Running containers via container runtime (Docker/containerd). <br> - Reporting Pod and Node status.                           | - Creating iptables/IPVS rules for Services. <br> - Load balancing traffic among Pods. <br> - Handling Service networking inside the cluster.            |
| **Core Functionality**                   | Ensures that **desired state = actual state** for Pods.                                                                                                              | Ensures that **Service traffic reaches correct backend Pods**.                                                                                           |
| **Communicates With**                    | - Container runtime (via CRI). <br> - API Server. <br> - Node OS.                                                                                                    | - Linux networking stack (iptables/IPVS). <br> - API Server.                                                                                             |
| **Type of Tasks**                        | Node and Pod management tasks.                                                                                                                                       | Network and routing tasks.                                                                                                                               |
| **Automatic Tasks**                      | - Registers node automatically. <br> - Pulls images (via container runtime). <br> - Starts and monitors containers. <br> - Performs health checks and restarts Pods. | - Watches Services and Endpoints automatically. <br> - Updates iptables/IPVS rules automatically. <br> - Balances traffic to healthy Pods automatically. |
| **Manual Tasks**                         | - Define Pod YAML and apply. <br> - Configure kubelet parameters (optional).                                                                                         | - Define Service YAML and apply. <br> - Configure proxy mode (optional).                                                                                 |
| **Configuration File**                   | `/var/lib/kubelet/config.yaml`                                                                                                                                       | `/var/lib/kube-proxy/config.conf`                                                                                                                        |
| **Modes of Operation**                   | N/A (it’s a single function agent).                                                                                                                                  | 1. Userspace <br> 2. iptables <br> 3. IPVS                                                                                                               |
| **Service/Pod Objects Watched**          | Watches **Pod** and **Node** objects.                                                                                                                                | Watches **Service** and **Endpoint** objects.                                                                                                            |
| **Load Balancing**                       | ❌ No                                                                                                                                                                 | ✅ Yes (internal cluster load balancing)                                                                                                                  |
| **Health Check Type**                    | Liveness/Readiness probes (for containers).                                                                                                                          | Checks backend endpoint health (for routing).                                                                                                            |
| **Container Image Pulling**              | Done by container runtime (on kubelet’s instruction).                                                                                                                | Not applicable.                                                                                                                                          |
| **Reporting Responsibility**             | Reports node and pod health/status to API Server.                                                                                                                    | Does not report; just applies network rules locally.                                                                                                     |
| **Restart Behavior**                     | Restarts failed containers automatically.                                                                                                                            | Reroutes traffic if a backend Pod fails.                                                                                                                 |
| **Dependency**                           | Depends on **container runtime**.                                                                                                                                    | Depends on **Linux networking stack** (iptables/IPVS).                                                                                                   |
| **Failure Impact**                       | If kubelet fails → Pods won’t be created/monitored on that node.                                                                                                     | If kube-proxy fails → Services become unreachable or load balancing breaks.                                                                              |
| **Log Checking Command**                 | `journalctl -u kubelet -f`                                                                                                                                           | `journalctl -u kube-proxy -f`                                                                                                                            |
| **Port Used (Default)**                  | 10250 (Kubelet API for metrics and communication).                                                                                                                   | 10249 (Metrics for monitoring).                                                                                                                          |
| **Security / Auth**                      | Uses TLS certificates and kubeconfig for authentication.                                                                                                             | Uses kubeconfig to securely connect to API Server.                                                                                                       |
| **Example Manual YAML Object**           | `Pod.yaml`                                                                                                                                                           | `Service.yaml`                                                                                                                                           |
| **Example CLI Command**                  | `kubectl apply -f pod.yaml`                                                                                                                                          | `kubectl apply -f service.yaml`                                                                                                                          |
| **Real-World Analogy**                   | Like a **node caretaker** ensuring all containers are healthy and running.                                                                                           | Like a **traffic director** ensuring requests reach the right container.                                                                                 |
| **Involvement in Cluster Bootstrapping** | Registers node and ensures readiness.                                                                                                                                | Not involved in bootstrapping; becomes active after Services are created.                                                                                |
| **Involvement in Scheduling**            | No — scheduling is done by kube-scheduler.                                                                                                                           | No — only handles routing after Pods are scheduled.                                                                                                      |
| **Component Type**                       | Node agent (system-level).                                                                                                                                           | Network proxy (service-level).                                                                                                                           |
