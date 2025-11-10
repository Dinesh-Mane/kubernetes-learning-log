# **`spec.containers.ports`**

## **Meaning**

* `spec.containers.ports` specifies the **network ports** that a container **listens on**.
* It tells Kubernetes **which ports** are **exposed inside the container** and optionally helps other components (like Services or Probes) know how to connect.
* It does **not actually open the port** — it’s **metadata** used by Kubernetes networking and tools.

> Think of it as:
> “Hey Kubernetes, my container is listening on port 8080 — please keep track of that.”

---

## **All types of Syntax**

```yaml
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - containerPort: 80
```

---

## **Explanation**

* `ports:` → list of ports exposed by the container.
* `containerPort:` → actual port number **inside the container** on which the app listens (e.g., nginx listens on 80).
* You can declare **multiple ports** per container if your app serves multiple purposes (HTTP, metrics, etc.).
* Kubernetes uses these port definitions for:

  * **Probes** (readiness/liveness checks)
  * **Service auto-discovery**
  * **Documentation and clarity**

---

## **Common Fields in `ports`**

| Field             | Type    | Required                   | Description                                                                      |
| ----------------- | ------- | -------------------------- | -------------------------------------------------------------------------------- |
| **containerPort** | Integer | ✅ Yes                      | The port that the container process listens on.                                  |
| **name**          | String  | ❌ Optional                 | A unique name (DNS_LABEL format) for the port — used in Services or Probes.      |
| **protocol**      | String  | ❌ Optional (default = TCP) | Can be `TCP` or `UDP` (rarely `SCTP`).                                           |
| **hostPort**      | Integer | ❌ Optional                 | Port number on the **host node** to expose the container’s port.                 |
| **hostIP**        | String  | ❌ Optional                 | Specific IP address of the host to bind to. (Default = all interfaces `0.0.0.0`) |

---

## **Example 1 — Basic HTTP Container**

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

✅ This tells Kubernetes:

> "The container named nginx listens on TCP port 80."

---

## **Example 2 — Multiple Ports**

```yaml
spec:
  containers:
    - name: webapp
      image: myapp:v1
      ports:
        - name: http
          containerPort: 80
        - name: metrics
          containerPort: 9090
```

✅ This container serves:

* HTTP traffic on port `80`
* Prometheus metrics on port `9090`

> These names (`http`, `metrics`) help you reference ports by name in **Services** or **Probes**.

---

## **Example 3 — With `hostPort` and `hostIP`**

```yaml
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
          hostPort: 8080
          hostIP: 127.0.0.1
```

✅ This binds:

* Container port **80** → Host port **8080**
* Only accessible on **localhost (127.0.0.1)** of the node

⚠️ Be careful — using `hostPort` restricts scheduling because Kubernetes must find a node with that port free.

---

## **Example 4 — UDP Protocol**

```yaml
spec:
  containers:
    - name: dns-server
      image: bind9
      ports:
        - name: dns
          containerPort: 53
          protocol: UDP
```

✅ Used for DNS, games, or any UDP-based communication.

---

## **Example 5 — Named Port with Probes**

```yaml
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: http
          containerPort: 8080
      livenessProbe:
        httpGet:
          path: /health
          port: http  # 👈 uses named port instead of number
```

✅ The probe automatically maps the name `http` → port `8080`.

---

## **Example 6 — Combined with Service**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: web
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: http
          containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
    - name: http
      port: 8080
      targetPort: http  # maps to named container port (80)
```

✅ The Service forwards traffic from port **8080 (service)** → **80 (container)** automatically using the name `http`.

---

## **Validation Rules**

| Rule                            | Description                                       |
| ------------------------------- | ------------------------------------------------- |
| ✅ **Required:** `containerPort` | Must be specified; cannot be zero.                |
| 🔢 **Range:** 1–65535           | Valid port numbers only.                          |
| ⚙️ **Type:** Integer            | Must not be quoted.                               |
| 🆔 **Unique Names:**            | If multiple ports defined, `name` must be unique. |
| 🧱 **Immutable:**               | Cannot be changed after Pod creation.             |
| 🧩 **Protocol Default:**        | If not specified, defaults to `TCP`.              |

---

## ⚠️ **Common Mistakes**

| Mistake                        | Result                                                            |
| ------------------------------ | ----------------------------------------------------------------- |
| `containerPort: "80"` (quoted) | Treated as string, may fail validation.                           |
| Using same port name twice     | Validation error — names must be unique.                          |
| Using `hostPort` carelessly    | Node scheduling issues — only one Pod per node can use that port. |
| Omitting `ports:` block        | Works fine, but not auto-discoverable by Services.                |

---

## ✅ **Best Practices**

1. **Always specify port names** if your app has multiple exposed ports.
   → Makes referencing easier in probes & services.

2. **Avoid hostPort** unless absolutely necessary.
   → It reduces scheduling flexibility.

3. **Use named ports** (`http`, `metrics`, etc.) for clean YAML references.

4. **Document purpose** of each port using descriptive names.

5. **Don’t quote port numbers** — must be integers.

---

## **Internal Representation Example**

After Pod creation:

```bash
kubectl get pod web-pod -o yaml
```

You’ll see:

```yaml
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: http
          containerPort: 80
          protocol: TCP
```

---

## 🧾 **Summary Table**

| Field             | Type   | Mandatory | Default   | Description              | Example      |
| ----------------- | ------ | --------- | --------- | ------------------------ | ------------ |
| **containerPort** | int    | ✅         | —         | Port inside container    | `80`         |
| **name**          | string | ❌         | —         | Human-readable port name | `http`       |
| **protocol**      | string | ❌         | `TCP`     | Communication protocol   | `UDP`, `TCP` |
| **hostPort**      | int    | ❌         | —         | Port on host node        | `8080`       |
| **hostIP**        | string | ❌         | `0.0.0.0` | Host IP to bind          | `127.0.0.1`  |

---

## **Quick Recap Summary**

* `spec.containers.ports` describes which ports the container listens on.
* Each port entry may include:

  * `containerPort` (required)
  * `name`, `protocol`, `hostPort`, `hostIP` (optional)
* Used by:

  * **Services**, **Probes**, and **Network Policies**.
* Does **not actually open** the port — the app inside the container must handle that.
* Default protocol = **TCP**.
* Avoid `hostPort` unless necessary (e.g., debugging or specific network bindings).
* Use **named ports** for cleaner YAML and easier references.

---
