# **Pod YAML**

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

Now let’s **break this down field by field** 👇

---

## 🧾 **1. apiVersion**

* **Meaning:** Defines which API version of Kubernetes this object uses.
* **Type:** Mandatory
* **Example:**

  ```yaml
  apiVersion: v1
  ```
* **When to use:** Always required.
  Each resource type (Pod, Deployment, Service, etc.) belongs to a specific API group and version.
* **Tip:** You can check valid versions with:

  ```bash
  kubectl api-resources
  ```

---

## 🧾 **2. kind**

* **Meaning:** Tells Kubernetes what type of object to create (Pod, Deployment, Service, etc.).
* **Type:** Mandatory
* **Example:**

  ```yaml
  kind: Pod
  ```
* **When to use:** Always; defines what you are creating.
  Kubernetes will reject a YAML without a `kind`.

---

## 🧾 **3. metadata**

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

## 🧾 **4. spec**

* **Meaning:** Describes the **desired state** of the Pod.
* **Type:** Mandatory
* **Example:**

  ```yaml
  spec:
    containers:
      - name: nginx-container
        image: nginx
  ```
* **When to use:** Always — it defines what to run and how.
  The `spec` block is where all behavior and configuration reside.

---

## 🧩 Inside `spec:` — the heart of the Pod

Let’s go deeper into the `spec` section 👇

---

### 🔹 4.1 **containers**

* **Meaning:** List of container definitions (a Pod can have multiple containers).
* **Type:** Mandatory
* **Example:**

  ```yaml
  containers:
    - name: nginx-container
      image: nginx:latest
  ```
* **When to use:** Always — every Pod must have at least one container.

---

### 🔹 4.2 **name**

* **Meaning:** Name of the container inside the Pod.
* **Type:** Mandatory
* **Example:**

  ```yaml
  name: nginx-container
  ```
* **When to use:** Always. Used by logs, exec commands, etc.

---

### 🔹 4.3 **image**

* **Meaning:** Specifies which container image to use.
* **Type:** Mandatory
* **Example:**

  ```yaml
  image: nginx:latest
  ```
* **When to use:** Always. You can pull from Docker Hub or private registries.

---

### 🔹 4.4 **ports**

* **Meaning:** Defines container ports to expose for communication.
* **Type:** Optional
* **Example:**

  ```yaml
  ports:
    - containerPort: 80
  ```
* **When to use:** When your app listens on a specific port.

---

### 🔹 4.5 **env**

* **Meaning:** Environment variables for containers.
* **Type:** Optional
* **Example:**

  ```yaml
  env:
    - name: ENVIRONMENT
      value: production
  ```
* **When to use:** When app needs configuration like credentials or mode.

---

### 🔹 4.6 **resources**

* **Meaning:** Specifies resource **requests and limits** for CPU/memory.
* **Type:** Optional but highly recommended.
* **Example:**

  ```yaml
  resources:
    requests:
      cpu: "200m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "256Mi"
  ```
* **When to use:** Always for production workloads to ensure fair scheduling.

---

### 🔹 4.7 **volumeMounts**

* **Meaning:** Mounts a volume inside the container.
* **Type:** Optional
* **Example:**

  ```yaml
  volumeMounts:
    - name: html-volume
      mountPath: /usr/share/nginx/html
  ```
* **When to use:** When container needs persistent or shared data.

---

### 🔹 4.8 **volumes**

* **Meaning:** Defines volumes that containers can mount.
* **Type:** Optional
* **Example:**

  ```yaml
  volumes:
    - name: html-volume
      emptyDir: {}
  ```
* **When to use:** Whenever you use `volumeMounts` in any container.

---

### 🔹 4.9 **nodeSelector**

* **Meaning:** Schedules Pod on specific nodes matching label criteria.
* **Type:** Optional
* **Example:**

  ```yaml
  nodeSelector:
    disktype: ssd
  ```
* **When to use:** To control on which node Pod should run (e.g., SSD nodes only).

---

### 🔹 4.10 **restartPolicy**

* **Meaning:** Defines how containers restart after failure.
* **Options:** `Always` (default), `OnFailure`, `Never`
* **Type:** Optional
* **Example:**

  ```yaml
  restartPolicy: Always
  ```
* **When to use:**

  * `Always` → standard Pods
  * `OnFailure` → batch jobs
  * `Never` → test/debug Pods

---

### 🔹 4.11 **imagePullPolicy**

* **Meaning:** Controls when Kubernetes pulls the image.
* **Options:** `Always`, `IfNotPresent`, `Never`
* **Type:** Optional
* **Example:**

  ```yaml
  imagePullPolicy: IfNotPresent
  ```
* **When to use:**
  Use `Always` for frequently updated images (like `latest` tag).

---

### 🔹 4.12 **command** & **args**

* **Meaning:** Override container’s default entrypoint and arguments.
* **Type:** Optional
* **Example:**

  ```yaml
  command: ["sleep"]
  args: ["3600"]
  ```
* **When to use:** When you need custom behavior for container.

---

### 🔹 4.13 **livenessProbe / readinessProbe**

* **Meaning:** Health checks for containers.
* **Type:** Optional but recommended.
* **Example:**

  ```yaml
  livenessProbe:
    httpGet:
      path: /
      port: 80
    initialDelaySeconds: 5
    periodSeconds: 10
  ```
* **When to use:** For apps that must restart automatically if unresponsive.

---

### 🔹 4.14 **affinity / tolerations**

* **Meaning:** Advanced scheduling rules for nodes and Pods.
* **Type:** Optional
* **Example:**

  ```yaml
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
  ```
* **When to use:** For fine-grained Pod placement control.

---

# 🧭 **Summary Table**

| Field             | Mandatory | Purpose               |
| ----------------- | --------- | --------------------- |
| `apiVersion`      | ✅         | Defines API version   |
| `kind`            | ✅         | Object type (Pod)     |
| `metadata.name`   | ✅         | Pod identifier        |
| `metadata.labels` | ❌         | Tagging/filtering     |
| `spec.containers` | ✅         | Container definitions |
| `image`           | ✅         | Container image       |
| `ports`           | ❌         | Expose ports          |
| `env`             | ❌         | Set environment vars  |
| `resources`       | ⚠️        | Resource management   |
| `volumes`         | ❌         | Shared storage        |
| `nodeSelector`    | ❌         | Node placement        |
| `restartPolicy`   | ⚠️        | Restart behavior      |
| `livenessProbe`   | ❌         | Health monitoring     |

---

Would you like me to make a **visual diagram (flow style)** showing how each YAML field fits together (like which fields belong under metadata, which under spec, etc.)?
It helps remember structure easily during exams or interviews.
