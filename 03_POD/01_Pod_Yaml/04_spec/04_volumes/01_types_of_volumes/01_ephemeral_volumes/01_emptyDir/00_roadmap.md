# FIRST — Core Concepts (Before Learning Any Volume Type)

Before even touching `emptyDir`, PVC, configMap, or anything else, you **must** understand these core ideas.
If you skip these, everything else in Kubernetes storage will feel confusing.

# (A) What is a Volume?

### 1. Simple Explanation

A **volume is just a folder that Kubernetes attaches to your Pod**, and your container(s) can read/write files inside that folder.

Think of it like:
Your container = a small temporary machine
Volume = an extra folder that comes from outside the container

### 2. Why do we need volumes?

Containers **delete everything** inside their filesystem when they restart.
Example:

* Your container writes logs to `/var/log/app.log`
* Container restarts → file is gone
  Because container’s root filesystem is **ephemeral**.

To avoid this loss, Kubernetes allows attaching a **volume**, which survives container restarts (but not always Pod restart depending on type).

### 3. Correct way to think about volume lifecycle

Volumes can be:

* Temporary (live only when Pod lives)
* Persistent (survive Pod deletion)
* Mounted from ConfigMap/Secret (read-only file based)
* Mounted from Host machine (hostPath)
* Mounted from external storage (PVC, CSI)

So each volume type has its own **lifecycle**, based on what storage backend it uses.

### 4. Important Truth

A *Kubernetes Volume = a directory mounted into a Pod*
A *Container’s filesystem = gets wiped on restart*

---

# (B) Relationship between volumes and volumeMounts

A mistake beginners make:
They think that defining `volumes:` is enough. No.

Kubernetes storage needs **two parts**:

### 1. `pod.spec.volumes`

This tells Kubernetes:

* What storage are we using?
* Where does it come from?
* What type of storage is it?

Example:

```yaml
volumes:
  - name: data-store
    emptyDir: {}
```

This only **defines the storage**.
It does NOT put anything inside the container yet.

### 2. `spec.containers.volumeMounts`

This tells Kubernetes:

* Where to put that storage **inside the container**

Example:

```yaml
volumeMounts:
  - name: data-store
    mountPath: /opt/app/data
```

This only mounts an **existing** volume.

### Combined Meaning

* `volumes` = declaring the storage
* `volumeMounts` = using the storage

If any container misses the `volumeMounts`, it cannot access the volume even if it exists.

### Example (complete)

```yaml
volumes:
  - name: mydata
    emptyDir: {}

containers:
  - name: app
    image: nginx
    volumeMounts:
      - name: mydata
        mountPath: /app/data
```

---

# (C) Volume Lifetime

A very important topic for DevOps engineers.

You must know what survives and what gets deleted.

Below is the truth table:

| Volume Type              | Survives Container Restart? | Survives Pod Restart?          | Survives Node Restart? | Survives Cluster Restart? |
| ------------------------ | --------------------------- | ------------------------------ | ---------------------- | ------------------------- |
| emptyDir                 | Yes                         | No                             | No                     | No                        |
| hostPath                 | Yes                         | Yes (if Pod goes to same node) | Yes                    | No guarantee              |
| pvc (persistent storage) | Yes                         | Yes                            | Yes                    | Yes                       |
| configMap/secret         | Yes                         | Yes                            | Yes                    | Yes                       |
| downwardAPI              | Yes                         | Yes                            | Yes                    | Yes                       |

### Explanation:

* Pod restart = new filesystem → emptyDir lost
* Node restart = temp disk cleared → emptyDir lost
* PVC is backed by cloud disk like EBS → survives everything
* hostPath depends on node — if node is down, data is gone

This lifecycle table is extremely important during interviews and real deployments.

---

# (D) ReadOnly vs ReadWrite

Some volumes **cannot** be written to
Some can.

### Read-only volumes:

* configMap
* secret
* projected volumes
* downwardAPI

You cannot modify files inside them (and shouldn’t).

### Read-write volumes:

* emptyDir
* hostPath
* PVC (persistent)
* CSI ephemeral

We usually use them for logs, temporary storage, caching, and persistent data.

---

# 1. emptyDir — Complete Deep Explanation

Now let’s get into the first real volume type.

---

# What is emptyDir?

`emptyDir` is a **temporary directory** created on the node when the Pod starts.
It is **empty** at the beginning (hence the name).
It exists as long as the **Pod exists**.

### The simplest mental model:

* Pod starts → empty folder created
* Container writes files inside it
* Pod dies → folder deleted forever

---

# Why do we use emptyDir?

### Used for:

1. Temporary data
2. Cache
3. Shared data between multiple containers in the same Pod
4. Storing large in-memory files (if using medium: Memory)

---

# How emptyDir works internally (detailed)

1. You define an emptyDir volume.
2. Kubelet creates a directory on the node:

   * For normal mode → on node disk
   * For `medium: Memory` → RAM-backed tmpfs
3. The directory is mounted into the container.
4. All containers that mount the same volume share the same folder.
5. When the Pod is deleted, that directory is fully deleted.

---

# emptyDir Types

There are two types of emptyDir:

## 1. Default emptyDir (disk-backed)

```yaml
emptyDir: {}
```

This uses the node’s hard disk (or SSD).

Good for:

* temporary files
* log buffering
* application caches
* file processing during jobs

## 2. Memory-backed emptyDir

```yaml
emptyDir:
  medium: Memory
```

This stores data in RAM.

Useful for:

* extremely fast access
* encryption key storage (more secure)
* streaming temporary data

But:

* RAM is expensive
* If node OOM occurs → kernel may evict data

---

# Real Example: Simple emptyDir

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo hello > /cache/hello.txt && sleep 3600"]
      volumeMounts:
        - name: cache
          mountPath: /cache
  volumes:
    - name: cache
      emptyDir: {}
```

### What happens?

1. Pod starts
2. Kubernetes creates a directory, e.g.:
   `/var/lib/kubelet/pods/<pod-id>/volumes/kubernetes.io~empty-dir/cache`
3. Container writes file `/cache/hello.txt`
4. If container crashes and restarts → file still exists
5. If Pod is deleted → file is gone forever

---

# Real Example: Sharing data between containers

This is VERY important.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "echo data > /shared/file.txt && sleep 3600"]
      volumeMounts:
        - name: work
          mountPath: /shared

    - name: reader
      image: busybox
      command: ["sh", "-c", "cat /shared/file.txt && sleep 3600"]
      volumeMounts:
        - name: work
          mountPath: /shared

  volumes:
    - name: work
      emptyDir: {}
```

### Use case:

initContainer writes config → main container reads it
or
sidecar container processes logs written by app

---

# Example: medium: Memory

```yaml
volumes:
  - name: fast-cache
    emptyDir:
      medium: Memory
```

Container:

```yaml
volumeMounts:
  - name: fast-cache
    mountPath: /tmp/cache
```

This creates an in-memory filesystem like `tmpfs`.

### Use cases:

* high-speed caching
* session storage
* encryption key handling (auto destroyed on pod death)

---

# Common Mistakes with emptyDir

## 1. Using emptyDir for persistent data

Example: storing DB data
When Pod dies → all data lost
This is a major production mistake.

## 2. Using medium: Memory without planning RAM usage

If your container writes 4GB into a memory emptyDir but the node only has 2GB free → node gets OOM-killed.

## 3. Assuming emptyDir survives Pod rescheduling

If Pod moves to another node, emptyDir is recreated fresh.

## 4. Expecting emptyDir to behave like PVC

PVC survives Pod deletion — emptyDir does NOT.

---
