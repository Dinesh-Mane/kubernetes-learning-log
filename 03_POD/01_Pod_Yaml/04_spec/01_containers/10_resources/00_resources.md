# **`spec.containers.resources` — Complete guide with examples, pitfalls, and best practices**

`spec.containers.resources` is the container-level resource specification in a Pod spec. It tells Kubernetes **how much CPU, memory, and other resources** a container **requests** (what scheduler uses to place pod) and **limits** (what kubelet enforces at runtime). This field is critical for scheduling, QoS, throttling, OOM behavior, autoscaling, and cluster stability.

---

## 1. Purpose — quick summary

* **requests**: what the container *needs*. The scheduler sums requests across containers and uses that to place Pods on nodes. Requests affect scheduling and QoS.
* **limits**: the maximum the container *may use*. Kubelet enforces limits (memory = hard limit → OOMKilled if exceeded; CPU = throttled via cgroups if over limit).
* **If you omit both**: container is BestEffort (lowest priority, can be evicted first).
* **If you set requests but no limits**: container is Burstable.
* **If every container has requests == limits**: Pod is Guaranteed (best protection from eviction).

---

## 2. YAML syntax (all valid forms)

Basic form (CPU in millicores, memory in Mi):

```yaml
resources:
  requests:
    cpu: "250m"        # 0.25 CPU
    memory: "128Mi"    # 128 mebibytes
  limits:
    cpu: "500m"
    memory: "256Mi"
```

You may omit `limits` or `requests`. Strings are recommended for clarity.

Other examples:

* Decimal CPU (allowed):

```yaml
requests:
  cpu: "0.5"   # 0.5 CPU == 500m
```

* Memory absolute bytes (integer) — not recommended (use Mi/Gi):

```yaml
limits:
  memory: 268435456   # 256 * 1024 * 1024 bytes
```

* Ephemeral storage:

```yaml
resources:
  requests:
    ephemeral-storage: "1Gi"
  limits:
    ephemeral-storage: "2Gi"
```

* GPU / extended resource (vendor-specific resource name):

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

* HugePages:

```yaml
resources:
  limits:
    hugepages-2Mi: "4Mi"
```

* `resourceFieldRef` (Downward API inside env/resourceFieldRef is different — not used in spec.resources)

---

## 3. How scheduler and kubelet use these values

* **Scheduler** uses `requests` to compute if a node has enough *allocatable* resources to host the pod. It sums requests of existing pods + this pod and compares to node allocatable.
* **Kubelet** enforces `limits` using cgroups:

  * **CPU**: container can use up to limit but is *throttled* if it tries to exceed. CPU usage is not hard-killed.
  * **Memory**: if container’s resident memory goes above `memory.limit` → container receives OOM and usually `OOMKilled`.
* **Eviction**: on node pressure (memory/disk), kubelet uses Pod `requests` and QoS classes to decide which pods to evict.

---

## 4. QoS classes (derived from requests/limits)

| QoS class  | Condition on containers' requests & limits                                                           | Effect                                                                                |
| ---------- | ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Guaranteed | Every container has `memory` and `cpu` requests and limits, and for each resource `request == limit` | Highest scheduling priority, protected from eviction unless node itself unrecoverable |
| Burstable  | At least one container has requests but not all equal to limits (or some have only request)          | Middle priority; may be throttled/evicted under pressure                              |
| BestEffort | No requests and no limits specified                                                                  | Lowest priority; first to be evicted on memory pressure                               |

**Example Guaranteed:**

```yaml
resources:
  requests: { cpu: "100m", memory: "256Mi" }
  limits:   { cpu: "100m", memory: "256Mi" }
```

---

## 5. Units and formats — common confusions

**CPU**

* Unit: cores.
* `1` == 1 CPU core, `0.5` == half core, `500m` == 500 milliCPU.
* Prefer `m` for clarity: `250m`, `1000m` (1000m == 1).
* Floating forms like `0.2` are accepted but `200m` is explicit.

**Memory**

* Use binary SI: `Ki`, `Mi`, `Gi`, `Ti` (kibibyte, mebibyte, gibibyte).
* `128Mi` is 134,217,728 bytes.
* Avoid `MB/GB` — they are decimal and can be confusing; K8s accepts metric suffixes but prefer Mi/Gi.

**Ephemeral storage**

* Use same suffixes: `1Gi`, `500Mi`.

**Extended resources**

* Use vendor resource name exactly: e.g., `nvidia.com/gpu: 1`.

**Common mistakes**

* Quoting integers: `containerPort: "80"` vs resources — always use strings for resources like `"256Mi"` or unquoted with suffix is acceptable, but avoid plain numbers for memory without units.
* Using `MB` vs `Mi`: `512MB` != `512Mi`.
* Missing units for memory will be interpreted as bytes (dangerous).

---

## 6. Examples: everyday cases

### 6.1 Webapp (typical)

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

Effect: scheduler reserves 200m, kubelet allows bursts up to 500m but throttle if sustained; memory above 512Mi → OOMKilled.

### 6.2 Sidecar logger with minimal resources

```yaml
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
```

No limits: Burstable; may be throttled if node overloaded.

### 6.3 Batch job requiring lots of CPU (and GPU)

```yaml
resources:
  requests:
    cpu: "4"
    memory: "16Gi"
  limits:
    cpu: "8"
    memory: "32Gi"
  limits:
    nvidia.com/gpu: 1
```

Note: scheduling will require node with 4 CPUs free and a GPU.

### 6.4 Pod with hugepages

```yaml
resources:
  limits:
    hugepages-2Mi: "64Mi"
```

### 6.5 Ephemeral-storage request/limit

```yaml
resources:
  requests:
    ephemeral-storage: "1Gi"
  limits:
    ephemeral-storage: "5Gi"
```

---

## 7. Pod-level behavior and initContainers

* **Requests are additive** across containers. Scheduler sums all containers' requests to decide placement.
* **Init containers**: scheduling uses the **max** of init containers' requests for each resource when starting; their resource usage is sequential but they can temporarily require more than regular containers.
* **Limits across containers** are per-container, not shared. The pod cannot borrow another container’s limit.

---

## 8. LimitRange and defaults in namespaces

* A `LimitRange` can enforce default requests/limits in a namespace. If a Pod omits requests or limits, LimitRange can populate them automatically.
* Also can set min/max allowed values. If you forget to set resources, cluster admins may apply defaults or reject creation.

---

## 9. Autoscaling and resources

* **Horizontal Pod Autoscaler (HPA)** (v1) uses **CPU utilization** target computed against `requests.cpu`. If you set requests incorrectly (too high/low), HPA scaling will be misinformed.
* HPA v2 can use custom metrics; but requests remain important as baseline.
* **Vertical Pod Autoscaler (VPA)** proposes new requests/limits based on historical usage — but VPA may evict/recreate pods to apply new values.

---

## 10. Node allocation and overcommit

* Nodes expose `capacity` and `allocatable`. Kubernetes allows overcommit: sum of requests may exceed actual capacity (but the scheduler uses requests vs allocatable for placement). Overcommit makes cluster utilization efficient but increases risk of eviction under pressure.

---

## 11. Troubles you may face & how to detect/fix

### Symptoms and causes

* **Pod stuck Pending**: likely requests too large for any node. `kubectl describe pod` shows scheduling events (`0/3 nodes are available: 3 Insufficient cpu`).

  * Fix: reduce requests or add capacity/labels/taints or use nodeSelector/affinity.

* **Container OOMKilled**: memory > memory.limit. `kubectl describe pod` or `kubectl get pod -o yaml` shows `state: terminated reason: OOMKilled`.

  * Fix: increase memory limit, optimize app memory, or remove memory leak.

* **CPU throttling (slow performance)**: container hits cpu.limit and gets throttled (check `kubectl describe pod` events or metrics).

  * Fix: increase cpu.limit or adjust requests/limits to avoid throttling for critical workloads.

* **Pod evicted under pressure**: check node events: `kubectl describe node <node>` shows eviction events (MemoryPressure/DiskPressure). QoS matters — BestEffort evicted first.

  * Fix: raise requests to make pod Guaranteed/Burstable appropriately, tune eviction thresholds, add node capacity.

* **HPA not scaling as expected**: HPA uses CPU usage vs requests. If requests are too low, HPA will scale aggressively; if too high, HPA won’t scale.

  * Fix: set reasonable `requests.cpu` based on real usage.

* **Scheduler errors with GPU**: GPU resources are extended resources; node must advertise `nvidia.com/gpu`. `kubectl describe nodes` can show allocatable GPUs.

### How to investigate

* `kubectl describe pod <pod>` — events, reason for OOM or eviction
* `kubectl get pod <pod> -o yaml` — see spec.resources
* `kubectl top pod <pod>` (requires metrics-server) — current CPU/memory usage
* `kubectl top node` — node utilization
* `kubectl describe node <node>` — allocatable, kubelet eviction events
* `journalctl -u kubelet` on node (operator) — eviction logs
* Check cgroup metrics on node for precise CPU throttling counts (advanced)

---

## 12. Common mistakes (and how to avoid)

1. **No resource requests/limits**

   * Problem: BestEffort pod, likely to be evicted.
   * Fix: Always set reasonable requests. Use LimitRange to enforce defaults.

2. **Using `latest` or not profiling before setting values**

   * Problem: Guesswork leads to over/under provisioning.
   * Fix: Measure via `kubectl top` and APM, then set requests to typical baseline and limits to safe max.

3. **Setting requests == limits too high for Guaranteed**

   * Problem: Many Guaranteed pods reduce packability and waste resources.
   * Fix: Only use Guaranteed when necessary.

4. **Using MB instead of Mi**

   * Problem: Unit mismatch may allocate wrong bytes.
   * Fix: Prefer `Mi`/`Gi`.

5. **Setting only limits but not requests**

   * Problem: Scheduler may place too many pods (requests default to 0), leading to overcommit and runtime contention.
   * Fix: Set both.

6. **Quoting numeric values incorrectly**

   * Problem: e.g., `cpu: "100"` might be interpreted as 100 cores if missing `m`.
   * Fix: Use `100m` for 0.1 CPU or `0.1` for decimal.

7. **Assuming CPU works like memory**

   * Problem: CPU is throttled, not killed. Apps expecting memory semantics will misbehave.
   * Fix: design for CPU throttling (e.g., concurrency limits, backpressure).

8. **Using host-level tools to measure resources incorrectly**

   * Problem: cgroup vs container-side measurements differ.
   * Fix: use container metrics (`kubectl top` or metrics exporter).

---

## 13. Advanced: ephemeral-storage, hugepages, extended resources

* **ephemeral-storage** is tracked and used by kubelet for eviction decisions. If a container writes too much ephemeral data beyond limit → container may be throttled/evicted.
* **hugepages** require node support and must be requested in `limits` (no requests).
* **Extended resources** (GPUs, FPGAs) are scheduled only if node advertises them and must be specified in `limits` (not requests for some vendors). Example: `limits: { "nvidia.com/gpu": 1 }`.

---

## 14. Commands to set or update resources

* Imperative (deployment):

```bash
kubectl set resources deployment/my-deploy --requests=cpu=200m,memory=256Mi --limits=cpu=500m,memory=512Mi
```

* To view pod allocated resources:

```bash
kubectl describe pod <pod-name>
kubectl top pod <pod-name>     # needs metrics-server
kubectl top node
```

---

## 15. Best practices checklist

1. **Always set `requests`** for cpu and memory for production workloads.
2. **Set `limits`** to avoid runaway containers (especially memory).
3. **Use `requests` that reflect baseline usage**, and `limits` that reflect safe maximum.
4. **Prefer millicores for CPU** (e.g., 100m, 250m).
5. **Use Mi/Gi for memory** (e.g., 128Mi, 2Gi).
6. **Avoid large Guaranteed footprints** unless strictly needed.
7. **Use LimitRanges** in namespaces to enforce sensible defaults.
8. **Monitor** (metrics-server, Prometheus) and iterate values.
9. **Test behavior under load** to detect throttling or OOMs.
10. **Document** why you chose particular requests/limits for future maintainers.

---

## 16. Quick reference examples

### Lightweight web container

```yaml
resources:
  requests: { cpu: "100m", memory: "128Mi" }
  limits:   { cpu: "500m", memory: "256Mi" }
```

### Worker (memory-hungry)

```yaml
resources:
  requests: { cpu: "500m", memory: "2Gi" }
  limits:   { cpu: "2",    memory: "8Gi" }
```

### GPU job

```yaml
resources:
  requests: { cpu: "4", memory: "16Gi" }
  limits:
    cpu: "8"
    memory: "32Gi"
    nvidia.com/gpu: 1
```

### Pod with ephemeral storage request

```yaml
resources:
  requests:
    ephemeral-storage: "1Gi"
```

---

## 17. Summary (do not forget)

* `requests` affect scheduling and QoS.
* `limits` enforce runtime constraints (memory = hard, cpu = throttled).
* Properly-specified resources prevent noisy neighbors, reduce evictions, and enable autoscaling to work correctly.
* Monitor, iterate, and use namespace policies (LimitRange, ResourceQuota) to enforce sane defaults.

---

# **`spec.containers.resources` in Imperative Commands**

## **Meaning**

The field **`spec.containers.resources`** in a Pod defines **how much CPU and memory** your container **requests** and the **maximum it can use**.

When you create a Pod using an **imperative command** like:

```bash
kubectl run mypod --image=nginx
```

→ By default, **no resource limits or requests** are set.
Your container can consume **unlimited CPU and memory**, which is risky in shared clusters.

That’s where `--requests` and `--limits` flags come in.

---

## **Core Concept Recap**

| Term         | Meaning                                     | Purpose                                         |
| ------------ | ------------------------------------------- | ----------------------------------------------- |
| **requests** | Minimum guaranteed CPU/memory for container | Scheduler uses this to decide where Pod can fit |
| **limits**   | Maximum CPU/memory container can use        | Kernel enforces this using cgroups              |

Think of it like this:

> 🧠 **Requests** = "I need at least this much to start and run properly."
> 🚫 **Limits** = "Don’t allow me to cross this much usage."

---

## **Imperative Command Syntax Forms**

Kubernetes supports **two primary forms** for specifying resources imperatively.

---

### **1. Inline Resource Definition with `--requests` and `--limits`**

**Syntax:**

```bash
kubectl run <pod-name> \
  --image=<image-name> \
  --requests='cpu=<value>,memory=<value>' \
  --limits='cpu=<value>,memory=<value>'
```

---

### **2. Through `kubectl set resources` (after Pod/Deployment creation)**

If a Pod or Deployment is **already created**, you can modify it:

```bash
kubectl set resources pod <pod-name> \
  --limits=cpu=500m,memory=256Mi \
  --requests=cpu=200m,memory=128Mi
```

✅ This is handy when you can’t modify YAML directly.

---

## **Examples (All Major Scenarios)**

---

### **Example 1 — Basic Pod with CPU/Memory Requests & Limits**

```bash
kubectl run nginx-pod \
  --image=nginx \
  --requests='cpu=200m,memory=128Mi' \
  --limits='cpu=500m,memory=256Mi'
```

✅ Creates a Pod with:

* Guaranteed **200 millicores CPU** and **128Mi memory**
* Hard limit: **500 millicores CPU**, **256Mi memory**

Check with:

```bash
kubectl get pod nginx-pod -o yaml
```

You’ll see:

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

---

### **Example 2 — CPU Only**

```bash
kubectl run cpu-pod \
  --image=busybox \
  --requests='cpu=250m' \
  --limits='cpu=1000m' \
  --command -- sh -c "sleep 3600"
```

✅ This container:

* Needs **0.25 CPU** minimum
* Can burst up to **1 full CPU core**

---

### **Example 3 — Memory Only**

```bash
kubectl run mem-pod \
  --image=alpine \
  --requests='memory=64Mi' \
  --limits='memory=128Mi' \
  --command -- sh -c "sleep 3600"
```

✅ Good for testing memory-based scheduling.

---

### **Example 4 — Setting Resources After Pod Creation**

If you forgot to add them while running:

```bash
kubectl set resources pod mem-pod \
  --requests=cpu=100m,memory=64Mi \
  --limits=cpu=500m,memory=256Mi
```

Kubernetes patches the live resource definition.

---

### **Example 5 — Deployment or ReplicaSet**

```bash
kubectl create deployment webapp --image=nginx \
  --replicas=2 \
  --dry-run=client -o yaml > webapp.yaml
```

Then apply imperatively:

```bash
kubectl set resources deployment webapp \
  --limits=cpu=500m,memory=256Mi \
  --requests=cpu=200m,memory=128Mi
```

✅ Automatically updates the resource section for **all replicas**.

---

## **How to Verify Resources**

1. **Describe the Pod:**

   ```bash
   kubectl describe pod nginx-pod
   ```

   Look for:

   ```
   Limits:
     cpu: 500m
     memory: 256Mi
   Requests:
     cpu: 200m
     memory: 128Mi
   ```

2. **Get YAML View:**

   ```bash
   kubectl get pod nginx-pod -o yaml
   ```

3. **Using Metrics Server (Optional):**

   ```bash
   kubectl top pod nginx-pod
   ```

   → Shows real-time CPU/Memory usage vs limit.

---

## **Units Explanation**

| Resource | Example Value | Meaning             |
| -------- | ------------- | ------------------- |
| CPU      | `500m`        | 0.5 core            |
| CPU      | `1`           | 1 full CPU core     |
| Memory   | `128Mi`       | 128 mebibytes       |
| Memory   | `512Mi`       | 512 mebibytes       |
| Memory   | `1Gi`         | 1 gibibyte (1024Mi) |

> Mnemonic tip 🧠:
> `m` → millicore (1/1000 CPU), `Mi` → Mebibyte, `Gi` → Gibibyte

---

## **Common Mistakes & Problems**

| Mistake                         | Example                                        | Effect                        |
| ------------------------------- | ---------------------------------------------- | ----------------------------- |
| ❌ Quotes around whole argument  | `'--limits="cpu=500m,memory=256Mi"'`           | Fails — incorrect flag format |
| ❌ Forgetting commas             | `--limits=cpu=500m memory=256Mi`               | Invalid syntax                |
| ❌ Wrong units                   | `--limits=cpu=0.5,memory=256MB`                | `MB` not allowed — use `Mi`   |
| ❌ Swapped fields                | `--requests=cpu=1Gi,memory=500m`               | Wrong unit type causes error  |
| ❌ Too high limit for small node | Node can’t schedule — "Insufficient resources" |                               |
| ⚠️ No requests set              | Pod may land anywhere — even overloaded nodes  |                               |
| ⚠️ Requests > node capacity     | Scheduler fails — Pending Pod forever          |                               |
| ⚠️ Limits < requests            | Invalid, rejected by API                       |                               |

---

## **Practical Tip — Testing Resource Enforcement**

You can simulate hitting memory limits:

```bash
kubectl run testlimit \
  --image=busybox \
  --limits='memory=64Mi' \
  --command -- sh -c "dd if=/dev/zero of=/dev/null bs=1M count=100"
```

→ You’ll get **OOMKilled (Out Of Memory)** because it exceeded 64Mi.

Check with:

```bash
kubectl get pod testlimit
kubectl describe pod testlimit
```

---

## **Default Behavior (When Not Specified)**

If you don’t define `resources`, behavior depends on:

* **Namespace LimitRange** or **ResourceQuota**
* If not configured, container can consume any amount (unbounded)

---

## **Dry Run to Verify YAML**

You can see how your imperative command converts to YAML:

```bash
kubectl run testpod \
  --image=nginx \
  --limits='cpu=400m,memory=256Mi' \
  --requests='cpu=200m,memory=128Mi' \
  --dry-run=client -o yaml
```

Output:

```yaml
resources:
  limits:
    cpu: 400m
    memory: 256Mi
  requests:
    cpu: 200m
    memory: 128Mi
```

---

## **When to Use What**

| Type              | Use Case              | Notes                                    |
| ----------------- | --------------------- | ---------------------------------------- |
| `--requests` only | Lightweight workloads | Ensures scheduling fairness              |
| `--limits` only   | Strictly cap usage    | Risk of scheduling issues if no requests |
| Both              | Recommended           | Balanced scheduling + resource safety    |

---

## **Real-world Example — Java App**

```bash
kubectl run java-api \
  --image=openjdk:17-jdk \
  --requests='cpu=250m,memory=512Mi' \
  --limits='cpu=1000m,memory=1Gi' \
  --command -- sh -c "java -jar myapp.jar"
```

✅ Ensures your JVM won’t overconsume node memory and cause eviction.

---

## **Summary Table**

| Flag                    | Example                                                     | Purpose                      |
| ----------------------- | ----------------------------------------------------------- | ---------------------------- |
| `--requests`            | `--requests=cpu=200m,memory=128Mi`                          | Minimum guaranteed resources |
| `--limits`              | `--limits=cpu=500m,memory=256Mi`                            | Maximum allowed usage        |
| `kubectl set resources` | `kubectl set resources pod web --limits=cpu=1,memory=512Mi` | Modify existing resources    |
| `kubectl top pod`       |                                                             | Check live usage             |
| `--dry-run -o yaml`     |                                                             | Verify YAML before applying  |

---

## **Best Practices**

1. **Always specify both requests & limits** for predictable performance.
2. **Avoid overcommitting** memory — causes OOMKilled.
3. Use **`kubectl top`** to tune real-world workloads.
4. Keep **ratios** close (e.g., 200m→500m) for fairness.
5. **Test under load** — see when your container throttles.
6. Avoid `Gi`/`Mi` mix-ups.
7. If using **HPA (Horizontal Pod Autoscaler)**, proper requests are essential — HPA uses them for scaling metrics.

---

