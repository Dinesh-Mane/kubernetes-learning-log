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

**Example:**
```yaml
apiVersion: v1
```

**When to use:**
Always mandatory. If wrong, Kubernetes will reject the YAML.

**Real use case:**
Different resources have different API versions — you must match it to the resource type.

Example groups:
* Pods → `v1`
* Deployments → `apps/v1`
> To see api virsion and kind use `kubectl api-resources` or `kubectl explain <resource> | grep apiVersion`

---

| NAME                     | SHORTNAMES | APIVERSION                   | NAMESPACED | KIND                    |
| ------------------------ | ---------- | ---------------------------- | ---------- | ----------------------- |
| pods                     | po         | v1                           | true       | Pod                     |
| services                 | svc        | v1                           | true       | Service                 |
| endpoints                | ep         | v1                           | true       | Endpoints               |
| configmaps               | cm         | v1                           | true       | ConfigMap               |
| secrets                  |            | v1                           | true       | Secret                  |
| namespaces               | ns         | v1                           | false      | Namespace               |
| nodes                    | no         | v1                           | false      | Node                    |
| persistentvolumes        | pv         | v1                           | false      | PersistentVolume        |
| persistentvolumeclaims   | pvc        | v1                           | true       | PersistentVolumeClaim   |
| serviceaccounts          | sa         | v1                           | true       | ServiceAccount          |
| replicationcontrollers   | rc         | v1                           | true       | ReplicationController   |
| deployments              | deploy     | apps/v1                      | true       | Deployment              |
| replicasets              | rs         | apps/v1                      | true       | ReplicaSet              |
| statefulsets             | sts        | apps/v1                      | true       | StatefulSet             |
| daemonsets               | ds         | apps/v1                      | true       | DaemonSet               |
| jobs                     |            | batch/v1                     | true       | Job                     |
| cronjobs                 | cj         | batch/v1                     | true       | CronJob                 |
| ingresses                | ing        | networking.k8s.io/v1         | true       | Ingress                 |
| networkpolicies          | netpol     | networking.k8s.io/v1         | true       | NetworkPolicy           |
| roles                    |            | rbac.authorization.k8s.io/v1 | true       | Role                    |
| rolebindings             |            | rbac.authorization.k8s.io/v1 | true       | RoleBinding             |
| clusterroles             |            | rbac.authorization.k8s.io/v1 | false      | ClusterRole             |
| clusterrolebindings      |            | rbac.authorization.k8s.io/v1 | false      | ClusterRoleBinding      |
| storageclasses           | sc         | storage.k8s.io/v1            | false      | StorageClass            |
| volumeattachments        |            | storage.k8s.io/v1            | false      | VolumeAttachment        |
| poddisruptionbudgets     | pdb        | policy/v1                    | true       | PodDisruptionBudget     |
| limitranges              | limits     | v1                           | true       | LimitRange              |
| resourcequotas           | quota      | v1                           | true       | ResourceQuota           |
| horizontalpodautoscalers | hpa        | autoscaling/v2               | true       | HorizontalPodAutoscaler |
| leases                   |            | coordination.k8s.io/v1       | true       | Lease                   |
| events                   | ev         | events.k8s.io/v1             | true       | Event                   |

---

### Notes:

* **NAMESPACED = true** → object exists within a namespace (e.g., Pods, Services).
* **NAMESPACED = false** → cluster-wide object (e.g., Nodes, PVs, ClusterRoles).
* To verify this list on your cluster, run:

  ```bash
  kubectl api-resources
  ```
* API versions may slightly change depending on your Kubernetes version.

---

# Here’s a **grouped and organized list of all popular Kubernetes resources** — arranged by category 

## ⚙️ **1. Core (Basic) Resources**

| NAME       | SHORTNAMES | APIVERSION       | NAMESPACED | KIND      |
| ---------- | ---------- | ---------------- | ---------- | --------- |
| pods       | po         | v1               | true       | Pod       |
| services   | svc        | v1               | true       | Service   |
| endpoints  | ep         | v1               | true       | Endpoints |
| configmaps | cm         | v1               | true       | ConfigMap |
| secrets    |            | v1               | true       | Secret    |
| namespaces | ns         | v1               | false      | Namespace |
| nodes      | no         | v1               | false      | Node      |
| events     | ev         | events.k8s.io/v1 | true       | Event     |

---

## 🧱 **2. Workload Resources (Apps & Controllers)**

| NAME                   | SHORTNAMES | APIVERSION | NAMESPACED | KIND                  |
| ---------------------- | ---------- | ---------- | ---------- | --------------------- |
| deployments            | deploy     | apps/v1    | true       | Deployment            |
| replicasets            | rs         | apps/v1    | true       | ReplicaSet            |
| statefulsets           | sts        | apps/v1    | true       | StatefulSet           |
| daemonsets             | ds         | apps/v1    | true       | DaemonSet             |
| replicationcontrollers | rc         | v1         | true       | ReplicationController |
| jobs                   |            | batch/v1   | true       | Job                   |
| cronjobs               | cj         | batch/v1   | true       | CronJob               |

---

## 🌐 **3. Networking Resources**

| NAME            | SHORTNAMES | APIVERSION           | NAMESPACED | KIND          |
| --------------- | ---------- | -------------------- | ---------- | ------------- |
| ingresses       | ing        | networking.k8s.io/v1 | true       | Ingress       |
| networkpolicies | netpol     | networking.k8s.io/v1 | true       | NetworkPolicy |
| services        | svc        | v1                   | true       | Service       |
| endpoints       | ep         | v1                   | true       | Endpoints     |

---

## 💾 **4. Storage Resources**

| NAME                   | SHORTNAMES | APIVERSION        | NAMESPACED | KIND                  |
| ---------------------- | ---------- | ----------------- | ---------- | --------------------- |
| persistentvolumes      | pv         | v1                | false      | PersistentVolume      |
| persistentvolumeclaims | pvc        | v1                | true       | PersistentVolumeClaim |
| storageclasses         | sc         | storage.k8s.io/v1 | false      | StorageClass          |
| volumeattachments      |            | storage.k8s.io/v1 | false      | VolumeAttachment      |

---

## 🔐 **5. RBAC (Role-Based Access Control)**

| NAME                | SHORTNAMES | APIVERSION                   | NAMESPACED | KIND               |
| ------------------- | ---------- | ---------------------------- | ---------- | ------------------ |
| roles               |            | rbac.authorization.k8s.io/v1 | true       | Role               |
| rolebindings        |            | rbac.authorization.k8s.io/v1 | true       | RoleBinding        |
| clusterroles        |            | rbac.authorization.k8s.io/v1 | false      | ClusterRole        |
| clusterrolebindings |            | rbac.authorization.k8s.io/v1 | false      | ClusterRoleBinding |
| serviceaccounts     | sa         | v1                           | true       | ServiceAccount     |

---

## ⚖️ **6. Resource Management**

| NAME                     | SHORTNAMES | APIVERSION     | NAMESPACED | KIND                    |
| ------------------------ | ---------- | -------------- | ---------- | ----------------------- |
| limitranges              | limits     | v1             | true       | LimitRange              |
| resourcequotas           | quota      | v1             | true       | ResourceQuota           |
| horizontalpodautoscalers | hpa        | autoscaling/v2 | true       | HorizontalPodAutoscaler |
| poddisruptionbudgets     | pdb        | policy/v1      | true       | PodDisruptionBudget     |

---

## 🧭 **7. Coordination & Scheduling**

| NAME           | SHORTNAMES | APIVERSION             | NAMESPACED | KIND          |
| -------------- | ---------- | ---------------------- | ---------- | ------------- |
| leases         |            | coordination.k8s.io/v1 | true       | Lease         |
| endpointslices |            | discovery.k8s.io/v1    | true       | EndpointSlice |

---

## 🧩 **8. Cluster-Level Resources**

| NAME                | SHORTNAMES | APIVERSION                   | NAMESPACED | KIND               |
| ------------------- | ---------- | ---------------------------- | ---------- | ------------------ |
| nodes               | no         | v1                           | false      | Node               |
| namespaces          | ns         | v1                           | false      | Namespace          |
| persistentvolumes   | pv         | v1                           | false      | PersistentVolume   |
| storageclasses      | sc         | storage.k8s.io/v1            | false      | StorageClass       |
| clusterroles        |            | rbac.authorization.k8s.io/v1 | false      | ClusterRole        |
| clusterrolebindings |            | rbac.authorization.k8s.io/v1 | false      | ClusterRoleBinding |

---

