# **Pod `spec.containers` Fields Explained**

```yaml
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      command: ["nginx"]
      args: ["-g", "daemon off;"]
      ports:
        - containerPort: 80
          name: http
          protocol: TCP
      env:
        - name: ENVIRONMENT
          value: production
      envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
      resources:
        requests:
          cpu: "200m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
          readOnly: false
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 5
      readinessProbe:
        tcpSocket:
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 3
      imagePullPolicy: IfNotPresent
      lifecycle:
        preStop:
          exec:
            command: ["nginx", "-s", "quit"]
```

---

## 1. `name`

* **Type:** String
* **Mandatory:** ✅ Yes
* **Meaning:** Name of the container (must be unique within a Pod).
* **Purpose:** Helps identify containers inside the same Pod.

**Example:**

```yaml
name: nginx-container
```

**Use case:** When you have multiple containers (e.g., main + sidecar), `name` uniquely identifies each.

---

## 2. `image`

* **Type:** String
* **Mandatory:** ✅ Yes
* **Meaning:** Specifies which container image to use.
* **Purpose:** Tells Kubelet what to pull and run.
* **Syntax:** `image: <repository>/<name>:<tag>`

**Example:**

```yaml
image: nginx:latest
```

**Use case:** Core of container definition — without it, the container won’t run.

---

## 3. `imagePullPolicy`

* **Type:** String
* **Values:** `Always`, `IfNotPresent`, `Never`
* **Default:**

  * `Always` → if tag = `latest`
  * `IfNotPresent` → otherwise

**Purpose:** Controls when to pull the image from registry.

**Examples:**

```yaml
imagePullPolicy: IfNotPresent
```

**Use cases:**

* `Always` → for CI/CD or frequently updated images
* `IfNotPresent` → stable production images
* `Never` → when you already preloaded image on nodes

---

## 4. `command`

* **Type:** Array of strings
* **Meaning:** Overrides the default **entrypoint** of the image.
* **Purpose:** Used to specify the exact process to start inside the container.

**Example:**

```yaml
command: ["nginx"]
```

**Use case:** Useful when customizing image behavior, overriding Dockerfile’s `ENTRYPOINT`.

---

## 5. `args`

* **Type:** Array of strings
* **Meaning:** Overrides the **CMD** arguments from the image.
* **Example:**

```yaml
args: ["-g", "daemon off;"]
```

**Use case:** Used to modify runtime parameters of your application.

---

## 6. `ports`

* **Type:** List
* **Meaning:** Exposes ports from container.
* **Subfields:**

  * `containerPort`: actual port number inside container
  * `protocol`: TCP/UDP/SCTP (default: TCP)
  * `name`: optional, helps identify port in Services

**Example:**

```yaml
ports:
  - name: http
    containerPort: 80
    protocol: TCP
```

**Use case:** When defining Service selectors or probes referring to container ports.

---

## 7. `env`

* **Type:** List
* **Meaning:** Environment variables for the container.
* **Purpose:** Inject configuration or secrets directly.

**Example:**

```yaml
env:
  - name: ENVIRONMENT
    value: production
```

**Use case:** To pass configuration dynamically (DB credentials, mode, etc.)

---

## 8. `envFrom`

* **Type:** List
* **Meaning:** Loads environment variables **from ConfigMaps or Secrets**.

**Example:**

```yaml
envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: app-secrets
```

**Use case:** When you have multiple environment variables managed centrally.

---

## 9. `resources`

* **Type:** Object
* **Subfields:**

  * `requests`: Minimum resources guaranteed
  * `limits`: Maximum resources allowed

**Example:**

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

**Use case:** Helps scheduler place pods on appropriate nodes; prevents resource hogging.

---

## 10. `volumeMounts`

* **Type:** List
* **Meaning:** Mounts storage volumes into containers.
* **Subfields:**

  * `name`: must match `volumes` section
  * `mountPath`: directory in container
  * `readOnly`: optional

**Example:**

```yaml
volumeMounts:
  - name: html-volume
    mountPath: /usr/share/nginx/html
```

**Use case:** Sharing files, configs, or persistent data between pods/containers.

---

## 11. `livenessProbe`

* **Purpose:** Checks **if the container is healthy**.
* **If fails → container is restarted.**
* **Probe types:** `httpGet`, `tcpSocket`, `exec`

**Example:**

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 5
```

**Use case:** Restart crashed or hung applications automatically.

---

## 12. `readinessProbe`

* **Purpose:** Checks if container is **ready to serve traffic**.
* **If fails → Pod removed from Service endpoints.**

**Example:**

```yaml
readinessProbe:
  tcpSocket:
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 3
```

**Use case:** Ensures traffic isn’t sent until app is fully initialized.

---

## 13. `startupProbe`

* **Purpose:** Checks **if app started successfully** (before liveness probe).
* **Example:**

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

**Use case:** For slow-starting apps like Java/Spring Boot.

---

## 14. `lifecycle`

* **Type:** Object
* **Meaning:** Defines hooks triggered during container lifecycle.
* **Subfields:**

  * `postStart`: run command **after** container starts
  * `preStop`: run command **before** container stops

**Example:**

```yaml
lifecycle:
  preStop:
    exec:
      command: ["nginx", "-s", "quit"]
```

**Use case:** Gracefully shut down apps or cleanup before termination.

---

## 15. `stdin`, `tty`, `stdinOnce`

* **Type:** Boolean
* **Purpose:** Used for interactive containers.
* **Example:**

```yaml
stdin: true
tty: true
```

**Use case:** For debugging or running interactive shells (`kubectl exec -it`).

---

## 🧭 **Container Fields Summary**

| Field             | Mandatory | Purpose                      | Example                 |
| ----------------- | --------- | ---------------------------- | ----------------------- |
| `name`            | ✅         | Unique container name        | `name: nginx`           |
| `image`           | ✅         | Defines which image to use   | `image: nginx:latest`   |
| `imagePullPolicy` | ❌         | Controls when to pull image  | `IfNotPresent`          |
| `command`         | ❌         | Override ENTRYPOINT          | `["nginx"]`             |
| `args`            | ❌         | Override CMD args            | `["-g", "daemon off;"]` |
| `ports`           | ❌         | Expose container ports       | `containerPort: 80`     |
| `env`             | ❌         | Define environment variables | `value: prod`           |
| `envFrom`         | ❌         | Import envs from CM/Secret   | `configMapRef`          |
| `resources`       | ❌         | Define CPU/memory limits     | `requests`, `limits`    |
| `volumeMounts`    | ❌         | Attach volumes               | `mountPath: /data`      |
| `livenessProbe`   | ❌         | Restart unhealthy container  | HTTP probe              |
| `readinessProbe`  | ❌         | Exclude unready pods         | TCP probe               |
| `startupProbe`    | ❌         | Check startup success        | Slow apps               |
| `lifecycle`       | ❌         | Hooks before/after run       | `preStop`               |
| `stdin`, `tty`    | ❌         | Enable interactive shell     | `true`                  |

---

Would you like me to continue next with **`pod.spec.volumes`** in the same structured way (types + examples + when to use)?
It’s another **very important interview topic** right after containers.
