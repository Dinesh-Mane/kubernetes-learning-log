# Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  namespace: dev
  labels:
    app: web
    tier: frontend
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
      env:
        - name: ENVIRONMENT
          value: production
      resources:
        limits:
          cpu: "500m"
          memory: "256Mi"
        requests:
          cpu: "200m"
          memory: "128Mi"
      volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
  volumes:
    - name: html-volume
      emptyDir: {}
  nodeSelector:
    disktype: ssd
  restartPolicy: Always
```

# **Top-Level Fields in a Pod YAML**

---

## 1️ `apiVersion`  *(string, mandatory)*

**Meaning:**
Defines which API version of Kubernetes this object uses.
Each Kubernetes object (Pod, Deployment, etc.) is defined in a particular API group and version.

**Purpose:**
Helps the API Server understand how to interpret the object.
Example groups:

* Pods → `v1`
* Deployments → `apps/v1`
> To see api virsion and kind use `kubectl api-resources`

**Example:**

```yaml
apiVersion: v1
```

**When to use:**
Always mandatory. If wrong, Kubernetes will reject the YAML.

**Real use case:**
Different resources have different API versions — you must match it to the resource type.

---

## 2. `kind`

* **Meaning:** Tells Kubernetes what type of object to create (Pod, Deployment, Service, etc.).
* **Type:** Mandatory
* **Example:**

  ```yaml
  kind: Pod
  ```
* **When to use:** Always; defines what you are creating.
  Kubernetes will reject a YAML without a `kind`.

---

## `3. metadata`

* **Meaning:** Contains **identification and organizational information** (name, labels, namespace, annotations).
* **Type:** Mandatory
* **Subfields:**

  * **name:** Unique name of the Pod within a namespace (required).
  * **namespace:** Logical group (optional, defaults to `default`).
  * **labels:** Key-value metadata for grouping/filtering (optional).
  * **annotations:** For attaching non-identifying info (optional).

* **Example:**

  ```yaml
  metadata:
    name: mypod
    namespace: dev
    labels:
      app: web
      tier: frontend
  ```

* **When to use:**
  Always include `name`; use `labels` and `annotations` for organization and automation.

---

## 4️ `spec`  *(PodSpec, mandatory)*

**Meaning:**
Defines the **desired state or configuration** of the Pod.

**Purpose:**
Describes what containers to run, what volumes to attach, restart policies, etc.
It’s the "instruction manual" for how Kubernetes should create and run your Pod.

**Example:**

```yaml
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
      env:
        - name: ENVIRONMENT
          value: production
      resources:
        limits:
          cpu: "500m"
          memory: "256Mi"
        requests:
          cpu: "200m"
          memory: "128Mi"
      volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
  volumes:
    - name: html-volume
      emptyDir: {}
  nodeSelector:
    disktype: ssd
  restartPolicy: Always
```

**When to use:**
Always mandatory — without `spec`, Kubernetes doesn’t know what to run.

**Real use case:**
All runtime behavior — container image, ports, volumes, environment variables — lives inside `spec`.

---

## 5️ `status`  *(PodStatus, read-only)*

**Meaning:**
Shows the **current observed state** of the Pod.
It’s automatically maintained by the Kubernetes control plane.

**Purpose:**
Used to check if the Pod is running, pending, succeeded, or failed.

**Example (auto-populated):**

```yaml
status:
  phase: Running
  hostIP: 10.0.0.5
  podIP: 10.244.0.12
  startTime: "2025-11-08T09:15:22Z"
```

**When to use:**
Read-only — you don’t define it. Kubernetes fills it automatically after creation.

**Real use case:**
View Pod’s live condition using:

```bash
kubectl get pod mypod -o yaml
```

or

```bash
kubectl describe pod mypod
```

---

# 🧭 **Summary Table**

| Field        | Type       | Mandatory | Purpose                               | Editable    |
| ------------ | ---------- | --------- | ------------------------------------- | ----------- |
| `apiVersion` | string     | ✅         | Defines API group/version             | Yes         |
| `kind`       | string     | ✅         | Specifies object type                 | Yes         |
| `metadata`   | ObjectMeta | ✅         | Identifies and organizes object       | Yes         |
| `spec`       | PodSpec    | ✅         | Describes desired behavior            | Yes         |
| `status`     | PodStatus  | ❌         | Shows actual state (Running, Pending) | ❌ Read-only |

---