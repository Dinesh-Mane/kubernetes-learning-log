
## 2. `kind`

* **Meaning:** Tells Kubernetes what type of object to create (Pod, Deployment, Service, etc.).
* **Type:** Mandatory
* **Example:**

  ```yaml
  kind: Pod
  ```
* **When to use:** Always; defines what you are creating.
  Kubernetes will reject a YAML without a `kind`.

To see api virsion and kind use `kubectl api-resources` or `kubectl explain <resource> | grep kind`

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
