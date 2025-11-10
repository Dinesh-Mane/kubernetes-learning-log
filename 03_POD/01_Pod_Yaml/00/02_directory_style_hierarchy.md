# **Pod YAML Directory-Level Structure**

```
pod.yaml
└── apiVersion: v1
└── kind: Pod
└── metadata:
    ├── name:
    ├── namespace:
    ├── labels:
    │   ├── app:
    │   └── tier:
    ├── annotations:
    │   └── key: value
    └── ownerReferences:        # links to higher-level controller
        ├── apiVersion:
        ├── kind:
        ├── name:
        └── uid:

└── spec:
    ├── containers: [ ]                  # Main section (core of Pod)
    │   └── - name:
    │       ├── image:
    │       ├── imagePullPolicy:
    │       ├── command: [ ]
    │       ├── args: [ ]
    │       ├── workingDir:
    │       ├── ports: [ ]
    │       │   └── - containerPort:
    │       │       ├── name:
    │       │       └── protocol:
    │       ├── env: [ ]
    │       │   └── - name:
    │       │       ├── value:
    │       │       └── valueFrom:
    │       │           ├── configMapKeyRef:
    │       │           │   ├── name:
    │       │           │   └── key:
    │       │           ├── secretKeyRef:
    │       │           │   ├── name:
    │       │           │   └── key:
    │       │           ├── fieldRef:
    │       │           │   └── fieldPath:
    │       │           └── resourceFieldRef:
    │       │               ├── resource:
    │       │               └── divisor:
    │       ├── envFrom: [ ]
    │       │   ├── configMapRef:
    │       │   │   └── name:
    │       │   └── secretRef:
    │       │       └── name:
    │       ├── resources:
    │       │   ├── limits:
    │       │   │   ├── cpu:
    │       │   │   └── memory:
    │       │   └── requests:
    │       │       ├── cpu:
    │       │       └── memory:
    │       ├── volumeMounts: [ ]
    │       │   └── - name:
    │       │       ├── mountPath:
    │       │       ├── subPath:
    │       │       └── readOnly:
    │       ├── volumeDevices: [ ]
    │       │   └── - name:
    │       │       └── devicePath:
    │       ├── livenessProbe:
    │       │   ├── httpGet:
    │       │   │   ├── path:
    │       │   │   ├── port:
    │       │   │   └── scheme:
    │       │   ├── exec:
    │       │   │   └── command: [ ]
    │       │   ├── tcpSocket:
    │       │   │   └── port:
    │       │   ├── initialDelaySeconds:
    │       │   ├── periodSeconds:
    │       │   ├── timeoutSeconds:
    │       │   ├── successThreshold:
    │       │   └── failureThreshold:
    │       ├── readinessProbe:         # same structure as livenessProbe
    │       ├── startupProbe:           # same structure as livenessProbe
    │       ├── lifecycle:
    │       │   ├── postStart:
    │       │   │   └── exec:
    │       │   │       └── command: [ ]
    │       │   └── preStop:
    │       │       └── exec:
    │       │           └── command: [ ]
    │       ├── securityContext:
    │       │   ├── runAsUser:
    │       │   ├── runAsGroup:
    │       │   ├── fsGroup:
    │       │   ├── allowPrivilegeEscalation:
    │       │   ├── readOnlyRootFilesystem:
    │       │   ├── privileged:
    │       │   ├── capabilities:
    │       │   │   ├── add: [ ]
    │       │   │   └── drop: [ ]
    │       │   └── seLinuxOptions:
    │       │       ├── level:
    │       │       ├── role:
    │       │       └── type:
    │       ├── terminationMessagePath:
    │       ├── terminationMessagePolicy:
    │       ├── stdin:
    │       ├── stdinOnce:
    │       └── tty:

    ├── initContainers: [ ]              # same structure as containers
    ├── ephemeralContainers: [ ]         # used for debugging pods

    ├── restartPolicy:                   # Always / OnFailure / Never
    ├── terminationGracePeriodSeconds:
    ├── activeDeadlineSeconds:
    ├── dnsPolicy:                       # ClusterFirst / Default / None
    ├── dnsConfig:
    │   ├── nameservers:
    │   ├── searches:
    │   └── options:
    ├── hostNetwork:
    ├── hostPID:
    ├── hostIPC:
    ├── nodeSelector:
    ├── nodeName:
    ├── serviceAccountName:
    ├── automountServiceAccountToken:
    ├── imagePullSecrets:
    │   └── - name:
    ├── affinity:
    │   ├── nodeAffinity:
    │   ├── podAffinity:
    │   └── podAntiAffinity:
    ├── tolerations: [ ]
    │   └── - key:
    │       ├── operator:
    │       ├── value:
    │       └── effect:
    ├── schedulerName:
    ├── priorityClassName:
    ├── securityContext:                 # Pod-level security context
    │   ├── fsGroup:
    │   ├── runAsUser:
    │   └── seLinuxOptions:
    ├── topologySpreadConstraints:
    │   └── - maxSkew:
    │       ├── topologyKey:
    │       ├── whenUnsatisfiable:
    │       └── labelSelector:
    ├── volumes: [ ]
    │   └── - name:
    │       ├── emptyDir:
    │       │   └── medium:
    │       ├── hostPath:
    │       │   ├── path:
    │       │   └── type:
    │       ├── configMap:
    │       │   ├── name:
    │       │   └── items:
    │       ├── secret:
    │       │   ├── secretName:
    │       │   └── items:
    │       ├── persistentVolumeClaim:
    │       │   └── claimName:
    │       └── projected:
    │           └── sources: [ ]
    ├── hostAliases:
    │   └── - ip:
    │       └── hostnames: [ ]
    ├── enableServiceLinks:
    ├── preemptionPolicy:
    ├── readinessGates:
    │   └── - conditionType:
    ├── runtimeClassName:
    ├── os:
    │   └── name: linux
    └── overhead:                        # Pod-level resource overhead

```

---

## **Understanding the Levels**

| Level                    | Description                              | Example                       |
| ------------------------ | ---------------------------------------- | ----------------------------- |
| **Root**                 | `apiVersion`, `kind`, `metadata`, `spec` | `apiVersion: v1`              |
| **spec.containers**      | Defines container runtime config         | `image`, `ports`, `env`, etc. |
| **spec.volumes**         | Defines storage available to containers  | `emptyDir`, `configMap`, etc. |
| **spec.securityContext** | Pod-wide permissions                     | `fsGroup`, `runAsUser`        |
| **metadata**             | Identity + labels for selectors          | Used by Deployments, Services |

---

## ✅ **Pro Tip (for Mastery)**

To become expert-level:

1. **Understand relationships** —
   `spec.containers.volumeMounts.name` ↔ `spec.volumes.name`
   `spec.containers.envFrom.configMapRef.name` ↔ `ConfigMap.metadata.name`

2. **Learn inheritance rules** —
   Some fields (like `securityContext`) exist both at Pod-level and container-level; container overrides Pod.

3. **Group learning path:**

   * Phase 1 → Metadata
   * Phase 2 → Containers
   * Phase 3 → Volumes
   * Phase 4 → Scheduling & Affinity
   * Phase 5 → Security, Lifecycle, and Probes

---

# **Fully Annotated Pod YAML Structure**

```
pod.yaml
├── apiVersion: v1                          # API group/version defining object schema
├── kind: Pod                               # Tells Kubernetes what resource this is
├── metadata:                               # Identification & organization metadata
│   ├── name:                               # Unique pod name within namespace
│   ├── namespace:                          # Namespace pod belongs to
│   ├── labels:                             # Key-value pairs for selectors (used by services, deployments)
│   ├── annotations:                        # Non-identifying metadata (used by tools/controllers)
│   └── ownerReferences:                    # Tracks parent object (like ReplicaSet, Deployment)
│       ├── apiVersion:
│       ├── kind:
│       ├── name:
│       └── uid:
│
└── spec:                                   # Desired behavior and configuration of the Pod
    ├── containers: [ ]                     # Main app containers
    │   └── - name:                         # Unique container name within Pod
    │       ├── image:                      # Container image to run
    │       ├── imagePullPolicy:            # Image pull strategy (Always / IfNotPresent / Never)
    │       ├── command: [ ]                # Overrides ENTRYPOINT in image
    │       ├── args: [ ]                   # Overrides CMD arguments in image
    │       ├── workingDir:                 # Default working directory for commands
    │       ├── ports: [ ]                  # List of ports exposed by container
    │       │   └── - containerPort:        # Port number container listens on
    │       │       ├── name:               # Optional port name (used by probes/services)
    │       │       └── protocol:           # Protocol (TCP/UDP/SCTP)
    │       ├── env: [ ]                    # Environment variables (key/value or from refs)
    │       │   └── - name:                 # Name of environment variable
    │       │       ├── value:              # Literal value
    │       │       └── valueFrom:          # Reference value from ConfigMap, Secret, or Pod field
    │       │           ├── configMapKeyRef:# Reads single key from ConfigMap
    │       │           ├── secretKeyRef:   # Reads single key from Secret
    │       │           ├── fieldRef:       # Reads Pod field (e.g., metadata.name)
    │       │           └── resourceFieldRef:# Reads resource limits/requests
    │       ├── envFrom: [ ]                # Import all keys from ConfigMap or Secret
    │       │   ├── configMapRef:           # Import all from ConfigMap
    │       │   └── secretRef:              # Import all from Secret
    │       ├── resources:                  # Resource requirements for container
    │       │   ├── limits:                 # Max CPU/memory the container can use
    │       │   └── requests:               # Minimum guaranteed CPU/memory reservation
    │       ├── volumeMounts: [ ]           # Mount defined volumes into container
    │       │   └── - name:                 # Must match spec.volumes.name
    │       │       ├── mountPath:          # Directory path inside container
    │       │       ├── subPath:            # Mount only specific file/dir within volume
    │       │       └── readOnly:           # Whether volume is writable
    │       ├── volumeDevices: [ ]          # Attach block devices (raw mode)
    │       │   └── - name:                 # Volume name
    │       │       └── devicePath:         # Path where block device appears
    │       ├── livenessProbe:              # Checks container health during runtime
    │       │   ├── httpGet:                # Probe over HTTP
    │       │   ├── exec:                   # Probe runs a command
    │       │   ├── tcpSocket:              # Probe over TCP
    │       │   ├── initialDelaySeconds:    # Wait before starting probes
    │       │   ├── periodSeconds:          # How often to probe
    │       │   ├── timeoutSeconds:         # Probe timeout duration
    │       │   ├── successThreshold:       # Success count needed to mark healthy
    │       │   └── failureThreshold:       # Fail count before restart
    │       ├── readinessProbe:             # Checks if container is ready for traffic
    │       ├── startupProbe:               # Checks app startup success (before other probes)
    │       ├── lifecycle:                  # Defines lifecycle hooks for container
    │       │   ├── postStart:              # Runs after container starts
    │       │   └── preStop:                # Runs before container stops
    │       ├── securityContext:            # Container-level security rules
    │       │   ├── runAsUser:              # UID container runs as
    │       │   ├── runAsGroup:             # GID container runs as
    │       │   ├── allowPrivilegeEscalation:# Block privilege escalation
    │       │   ├── readOnlyRootFilesystem: # Make root FS read-only
    │       │   ├── privileged:             # Run container with full privileges
    │       │   ├── capabilities:           # Add/remove Linux capabilities
    │       │   └── seLinuxOptions:         # SELinux labels
    │       ├── terminationMessagePath:     # Where exit message/log is stored
    │       ├── terminationMessagePolicy:   # Controls message source (File or FallbackToLogs)
    │       ├── stdin:                      # Keep STDIN open for interaction
    │       ├── stdinOnce:                  # Close STDIN once session ends
    │       └── tty:                        # Allocate a terminal for interactive shell
    │
    ├── initContainers: [ ]                 # Containers that run before main ones
    ├── ephemeralContainers: [ ]            # Temporary containers for debugging
    │
    ├── restartPolicy: Always               # Pod restart behavior (Always/OnFailure/Never)
    ├── terminationGracePeriodSeconds:      # Time to wait before killing container
    ├── activeDeadlineSeconds:              # Pod max runtime limit
    ├── dnsPolicy: ClusterFirst             # DNS lookup policy
    ├── dnsConfig:                          # Custom DNS options
    │   ├── nameservers: [ ]                # Override DNS servers
    │   ├── searches: [ ]                   # Search domains
    │   └── options: [ ]                    # DNS resolver options
    ├── hostNetwork: false                  # Share host’s network namespace
    ├── hostPID: false                      # Share host’s process namespace
    ├── hostIPC: false                      # Share host’s IPC namespace
    ├── nodeSelector:                       # Schedule Pod on specific node labels
    ├── nodeName:                           # Force schedule on a specific node
    ├── serviceAccountName:                 # Use a specific ServiceAccount
    ├── automountServiceAccountToken: true  # Auto-mount service account token
    ├── imagePullSecrets:                   # Secret refs for private registries
    │   └── - name:
    ├── affinity:                           # Rules for node/pod placement
    │   ├── nodeAffinity:                   # Match Pods to specific nodes
    │   ├── podAffinity:                    # Co-locate with specific Pods
    │   └── podAntiAffinity:                # Avoid specific Pods
    ├── tolerations: [ ]                    # Allow scheduling on tainted nodes
    │   └── - key:
    │       ├── operator:
    │       ├── value:
    │       └── effect:
    ├── schedulerName:                      # Custom scheduler name
    ├── priorityClassName:                  # Assign scheduling priority
    ├── securityContext:                    # Pod-level security settings
    │   ├── fsGroup:                        # Filesystem group ownership
    │   ├── runAsUser:                      # Default user ID for containers
    │   └── seLinuxOptions:                 # SELinux context
    ├── topologySpreadConstraints:          # Spread Pods across topology (zones, nodes)
    │   └── - maxSkew:
    │       ├── topologyKey:
    │       ├── whenUnsatisfiable:
    │       └── labelSelector:
    ├── volumes: [ ]                        # Define all storage volumes available to containers
    │   └── - name:
    │       ├── emptyDir:                   # Temporary storage
    │       ├── hostPath:                   # Mount host file or directory
    │       ├── configMap:                  # Mount ConfigMap data as files
    │       ├── secret:                     # Mount Secret data as files
    │       ├── persistentVolumeClaim:      # Attach PVC for persistent storage
    │       └── projected:                  # Combine multiple volume sources
    ├── hostAliases:                        # Add entries to /etc/hosts
    │   └── - ip:
    │       └── hostnames: [ ]
    ├── enableServiceLinks: true            # Inject service env vars automatically
    ├── preemptionPolicy:                   # Allow/prevent preemption
    ├── readinessGates:                     # Custom Pod readiness conditions
    │   └── - conditionType:
    ├── runtimeClassName:                   # Select container runtime (e.g., gVisor)
    ├── os:                                 # Specify OS type (linux/windows)
    │   └── name: linux
    └── overhead:                           # Resource overhead assigned to Pod
```

---

## **How to Use This Structure**

| Purpose              | How to Use                                                                            |
| -------------------- | ------------------------------------------------------------------------------------- |
| **Learning Roadmap** | Start top-down: metadata → spec → containers → probes → security → scheduling         |
| **Interview Prep**   | Remember *why each field exists* and *how it connects* (e.g., volumeMounts ↔ volumes) |
| **Debugging**        | Quickly find where each config belongs (container-level vs pod-level)                 |
| **Real Projects**    | Use as a checklist when writing production-grade manifests                            |


