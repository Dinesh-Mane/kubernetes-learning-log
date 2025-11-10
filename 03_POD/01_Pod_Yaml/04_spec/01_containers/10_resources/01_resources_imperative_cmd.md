# ⚙️ **`spec.containers.resources` — Imperative Command Edition**

## 🧠 **Quick Recap**

This field controls how much **CPU** and **Memory (RAM)** your container requests and limits.

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

Now let’s see how to set these from **imperative kubectl commands** 👇

---

## 🧩 **1️⃣ Basic Pod Creation With Resource Limits**

### ✅ Command:

```bash
kubectl run mypod --image=nginx \
  --requests='cpu=100m,memory=128Mi' \
  --limits='cpu=500m,memory=512Mi'
```

### ✅ Equivalent YAML:

```yaml
spec:
  containers:
  - name: mypod
    image: nginx
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
```

📌 **Explanation:**

* `--requests` sets **minimum guaranteed** resources.
* `--limits` sets **maximum allowed** resources.

🧠 Mnemonic:
**Requests = Reserve**, **Limits = Lock upper bound**.

---

## 🧩 **2️⃣ Only Setting Requests (Optional)**

```bash
kubectl run busybox --image=busybox \
  --requests='cpu=50m,memory=64Mi' -- sleep 100
```

✅ This sets only requests; limits remain unset.

---

## 🧩 **3️⃣ Only Setting Limits (Optional)**

```bash
kubectl run webapp --image=nginx \
  --limits='cpu=1,memory=1Gi'
```

✅ Sets only limits (no requests).
Kubernetes will schedule the Pod **without guaranteed resources**.

---

## 🧩 **4️⃣ Using `kubectl set resources` on an Existing Deployment**

If your Deployment already exists:

```bash
kubectl set resources deployment my-deploy \
  --containers=my-container \
  --limits=cpu=800m,memory=512Mi \
  --requests=cpu=200m,memory=128Mi
```

✅ Updates the container’s resource settings **without recreating** the deployment manually.

🧾 Verify:

```bash
kubectl get deployment my-deploy -o yaml | grep -A5 resources
```

---

## 🧩 **5️⃣ Using Multiple Containers in One Pod**

You can specify per-container limits with separate `--overrides` YAML (JSON patch):

```bash
kubectl run multi --image=nginx --overrides='
{
  "spec": {
    "containers": [
      {
        "name": "frontend",
        "image": "nginx",
        "resources": {
          "limits": {"cpu": "500m", "memory": "512Mi"}
        }
      },
      {
        "name": "sidecar",
        "image": "busybox",
        "args": ["sleep","3600"],
        "resources": {
          "limits": {"cpu": "100m", "memory": "128Mi"}
        }
      }
    ]
  }
}'
```

✅ Creates a Pod with two containers, each having distinct resource limits.

---

## 🧩 **6️⃣ Editing Resources After Creation**

You can’t directly edit resources of a running Pod.
If you try:

```bash
kubectl edit pod mypod
```

and change:

```yaml
resources:
  limits:
    cpu: "1"
```

You’ll get ❌ an error:

> “field is immutable”.

🩹 **Fix:** Delete and recreate, or use **Deployment** and then `kubectl set resources`.

---

## 🧩 **7️⃣ Units You Can Use**

| Resource | Unit Examples    | Meaning                         |
| -------- | ---------------- | ------------------------------- |
| CPU      | `m`, none        | `100m = 0.1 core`, `1 = 1 core` |
| Memory   | `Ki`, `Mi`, `Gi` | `128Mi = 134MB`, `1Gi = 1.07GB` |

🧠 Tip: Always use **Mi/Gi** for memory and **m** for CPU to avoid confusion.

---

## 🧩 **8️⃣ Common Mistakes (and Fixes)**

| Mistake                             | What Happens                             | Fix                                      |
| ----------------------------------- | ---------------------------------------- | ---------------------------------------- |
| `--limits='cpu=500,memory=512Mi'`   | ❌ CPU "500" means **500 cores**, not 0.5 | Use `500m`                               |
| `--requests='cpu=0.5,memory=512Mi'` | ❌ Invalid unit                           | Use `500m` instead of decimal            |
| `--limits='cpu=1'` (no memory)      | ⚠️ Works, but Pod can consume all memory | Always set both CPU + memory             |
| Missing quotes                      | YAML parsing errors                      | Wrap values in `' '` if comma-separated  |
| Editing Pod resources manually      | ❌ Immutable field                        | Use Deployment + `kubectl set resources` |

---

## 🧩 **9️⃣ Example — Apply via Imperative Command then Verify**

```bash
kubectl run app --image=nginx \
  --requests='cpu=250m,memory=256Mi' \
  --limits='cpu=500m,memory=512Mi'
```

Then check:

```bash
kubectl get pod app -o yaml | grep -A5 resources
```

Output:

```yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
```

---

## 🧩 **🔟 Example — With Deployment**

```bash
kubectl create deployment mydeploy --image=nginx \
  --dry-run=client -o yaml > dep.yaml
```

Edit `dep.yaml` and add:

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "512Mi"
```

Apply it:

```bash
kubectl apply -f dep.yaml
```

Then check:

```bash
kubectl describe pod | grep -A4 Limits
```

---

## ⚠️ **11️⃣ Validation Rules**

| Field        | Mandatory              | Description                      |
| ------------ | ---------------------- | -------------------------------- |
| `requests`   | ❌ Optional             | Scheduler uses it for placement  |
| `limits`     | ❌ Optional             | Kubelet enforces hard caps       |
| `cpu/memory` | ✅ Required inside both | Unit-sensitive                   |
| Type         | String                 | Must be string (even if numeric) |
| Immutable    | ✅                      | Once Pod runs, can’t change      |

---

## 💡 **12️⃣ Advanced: CPU Burst Problem**

If you only specify **requests**, your Pod can use more CPU temporarily (burstable class).
If you set both requests=limits, Pod becomes **Guaranteed class**.

| QoS Class      | Definition         |
| -------------- | ------------------ |
| **Guaranteed** | requests == limits |
| **Burstable**  | requests < limits  |
| **BestEffort** | No requests/limits |

🧠 **Mnemonic:**

* *Guaranteed* = VIP pod (reserved seat 🎟️)
* *Burstable* = normal pod (standby seat)
* *BestEffort* = waitlist pod 😅

---

## ✅ **13️⃣ Quick Recap Summary**

| Command                           | Purpose                       |
| --------------------------------- | ----------------------------- |
| `--requests='cpu=...,memory=...'` | Minimum resources             |
| `--limits='cpu=...,memory=...'`   | Maximum resources             |
| `kubectl set resources`           | Update running workloads      |
| `kubectl get pod -o yaml`         | Verify resource section       |
| Units: `m`, `Mi`, `Gi`            | For CPU & memory              |
| Immutable                         | Can’t edit on running pod     |
| Use quotes `' '`                  | To avoid shell parsing issues |

---

## 🧾 **Final Example for Memory**

```bash
kubectl run ramtest --image=busybox \
  --limits='memory=1Gi' \
  --requests='memory=512Mi' -- sleep 3600
```

✅ Final YAML:

```yaml
resources:
  limits:
    memory: "1Gi"
  requests:
    memory: "512Mi"
```

---