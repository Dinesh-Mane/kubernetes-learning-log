![](../images/01.png)
# ETCD in Kubernetes
- ETCD is a distributed reliable key-value store used by kubernetes to store all data used to manage the cluster
- etcd is a distributed key-value store that maintains configuration data, state information, and metadata for your Kubernetes cluster.
- It stores information regarding every object—nodes, pods, configurations, secrets, accounts, roles, and role bindings—is stored within etcd. 
- Every information you see when you run the **`kubectl get`** command is from the **`ETCD Server`**.
-  ETCD is responsible for implementing locks within the cluster to ensure there are no conflicts between the Masters. 
- Any changes you make to the cluster—whether adding nodes, deploying pods, or configuring ReplicaSets—are first recorded in etcd. Only after etcd is updated are these changes considered to be complete.
- By default, etcd listens on port **`2379`**, allowing you to use the etcdctl client tool to store and retrieve key-value pairs.
- kube-apiserver is the only component that interacts directly to the etcd datastore.

---

# Deployment Methods
Depending on your Kubernetes setup, you can deploy etcd in two primary ways: manually from scratch or automatically with kubeadm. Each method has its use cases, with manual setups providing a deeper understanding of etcd configurations and kubeadm streamlining the deployment process.

## 1. Manually from scratch
- If you setup your cluster from scratch then you deploy **`ETCD`** by downloading ETCD Binaries yourself
- Installing Binaries and Configuring ETCD as a service in your master node yourself.  

```bash
$ wget -q --https-only "https://github.com/etcd-io/etcd/releases/download/v3.3.11/etcd-v3.3.11-linux-amd64.tar.gz"
  ```
```bash
wget -q --https-only \
"https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"


# Example etcd service configuration
ExecStart=/usr/local/bin/etcd \
  --name ${ETCD_NAME} \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://${INTERNAL_IP}:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster controller-0=https://${CONTROLLER0_IP}:2380,controller-1=https://${CONTROLLER1_IP}:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
```

> **TLS Certificates :**  
> For detailed information on configuring TLS certificates, refer to the Kubernetes documentation on TLS Configuration. Adjust certificate parameters according to your security requirements.


### High Availability Considerations
In a production Kubernetes environment, high availability (HA) is paramount. By running multiple master nodes with corresponding etcd instances, you ensure that your cluster remains resilient even if one node fails.

To enable HA, each etcd instance must know about its peers. This is achieved by configuring the `--initial-cluster` parameter with the details of each member in the cluster. For example:

```bash
ExecStart=/usr/local/bin/etcd \
  --name ${ETCD_NAME} \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls=https://${INTERNAL_IP}:2380 \
  --listen-peer-urls=https://${INTERNAL_IP}:2380 \
  --advertise-client-urls=https://${INTERNAL_IP}:2379 \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster controller-0=https://${CONTROLLER0_IP}:2380,controller-1=https://${CONTROLLER1_IP}:2380 \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
```

> **Alternate Certificate Options :**  
> In some deployments, you may use separate certificate files for peer communications (e.g., `/etc/etcd/peer.pem` and `/etc/etcd/peer-key.pem`). Always tailor these settings to match your desired security posture.


## 2. Automatically with kubeadm
If you setup your cluster using **`kubeadm`** then kubeadm will deploy etcd server for you as a pod in **`kube-system`** namespace.
```bash
$ kubectl get pods -n kube-system
```

Example output:
```bash
NAMESPACE     NAME                                 READY   STATUS      RESTARTS   AGE
kube-system   coredns-78fcdf6894-prwl              1/1     Running     0          1h
kube-system   coredns-78fcdf6894-vqd9w             1/1     Running     0          1h
kube-system   etcd-master                          1/1     Running     0          1h
kube-system   kube-apiserver-master                1/1     Running     0          1h
kube-system   kube-controller-manager-master       1/1     Running     0          1h
kube-system   kube-proxy-f6k26                     1/1     Running     0          1h
kube-system   kube-proxy-hnzw                      1/1     Running     0          1h
kube-system   kube-scheduler-master                1/1     Running     0          1h
kube-system   weave-net-924k8                      2/2     Running     1          1h
kube-system   weave-net-hzfcz                      2/2     Running     1          1h
```
To examine the keys stored in etcd (organized under the registry directory), use the following command:
```bash
kubectl exec etcd-master -n kube-system -- etcdctl get / --prefix --keys-only
```
Sample output:
```bash
/registry/apiregistration.k8s.io/apiservices/v1
/registry/apiregistration.k8s.io/apiservices/v1.apps
/registry/apiregistration.k8s.io/apiservices/v1.authentication.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.authorization.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.autoscaling
/registry/apiregistration.k8s.io/apiservices/v1.batch
/registry/apiregistration.k8s.io/apiservices/v1.networking.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.rbac.authorization.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1beta1.admissionregistration.k8s.io
```
The etcd root directory, organized as the registry, contains subdirectories for various Kubernetes components such as nodes, pods, ReplicaSets, and Deployments.


### ETCD in HA Environment
- In a high availability environment, you will have multiple master nodes in your cluster that will have multiple ETCD Instances spread across the master nodes.
- Make sure etcd instances know each other by setting the right parameter in the **`etcd.service`** configuration. The **`--initial-cluster`** option where you need to specify the different instances of the etcd service.

---

## Explore ETCD
To list all keys stored by kubernetes, run the below command
```bash
$ kubectl exec etcd-master -n kube-system -- sh -c "ETCDCTL_API=3 etcdctl --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --cacert=/etc/kubernetes/pki/etcd/ca.crt get / --prefix --keys-only"
```
- Kubernetes Stores data in a specific directory structure, the root directory is the **`registry`** and under that you have various kubernetes constructs such as **`minions`**, **`nodes`**, **`pods`**, **`replicasets`**, **`deployments`**, **`roles`**, **`secrets`** and **`Others`**.



