# **Complete Learning Roadmap for `pod.spec.containers.securityContext`**
## **1. Core Concept and Purpose**

* Understand **what `securityContext` is** and why it exists.
* Difference between:

  * `pod.spec.securityContext` (applies to all containers)
  * `pod.spec.containers[*].securityContext` (per-container level)
* How Kubernetes uses `securityContext` to configure Linux namespaces, UID/GID mappings, and kernel security features.

---

## **2. Field-Level Overview**

Learn every field that can appear under `securityContext`:

| Field                      | Type                     | Purpose                                              |
| -------------------------- | ------------------------ | ---------------------------------------------------- |
| `runAsUser`                | int64                    | Run processes as a specific user ID                  |
| `runAsGroup`               | int64                    | Run processes with a specific group ID               |
| `runAsNonRoot`             | bool                     | Enforce non-root execution                           |
| `privileged`               | bool                     | Run container in privileged mode (host-level access) |
| `allowPrivilegeEscalation` | bool                     | Control privilege escalation (e.g., sudo, setuid)    |
| `capabilities`             | object                   | Add/drop Linux capabilities                          |
| `readOnlyRootFilesystem`   | bool                     | Make root filesystem read-only                       |
| `procMount`                | string                   | Control `/proc` visibility (`Default`, `Unmasked`)   |
| `seLinuxOptions`           | object                   | Apply SELinux labels                                 |
| `seccompProfile`           | object                   | Restrict syscalls with seccomp                       |
| `appArmorProfile`          | string (via annotations) | Restrict syscalls with AppArmor                      |
| `windowsOptions`           | object                   | Security fields for Windows containers               |

---

## **3. Understanding Linux Privileges**

* Concept of **root user inside containers** and why it’s dangerous.
* Difference between **root inside container** vs **root on host**.
* Linux **UID, GID, and supplementary groups**.
* Concept of **privilege escalation** and **capabilities**.

---

## **4. Privileged Containers**

* What happens when `privileged: true`.
* Access to host devices (`/dev`), kernel modules, etc.
* Risks of privileged containers — container breakout scenarios.
* When (if ever) privileged mode is justified (e.g., CNI, CSI drivers).

---

## **5. Linux Capabilities Deep Dive**

* Learn the concept of Linux capabilities (fine-grained privileges split from root).
* Key capabilities like `NET_ADMIN`, `SYS_ADMIN`, `CAP_SYS_PTRACE`, etc.
* Fields:

  ```yaml
  capabilities:
    add: ["NET_ADMIN", "SYS_TIME"]
    drop: ["ALL"]
  ```
* Best practice: Drop all capabilities, add only needed ones.

---

## **6. User and Group Management**

* Meaning of:

  ```yaml
  runAsUser: 1000
  runAsGroup: 3000
  ```
* Difference between `runAsNonRoot: true` vs setting `runAsUser` manually.
* How Kubernetes validates UID/GID combinations.
* How group ownership affects file writes in mounted volumes.

---

## **7. Filesystem Protections**

* `readOnlyRootFilesystem: true` — benefits and limitations.
* Mount writable paths separately (like `/tmp` or `/var/log`).
* How this mitigates container tampering and malware persistence.

---

## **8. Privilege Escalation Controls**

* `allowPrivilegeEscalation: false` — how it prevents gaining extra privileges via `setuid` binaries.
* Combined effect with `capabilities.drop: ["ALL"]` and `runAsNonRoot: true`.

---

## **9. Seccomp Profiles**

* What is seccomp and how it limits system calls.
* Types of profiles: `Unconfined`, `RuntimeDefault`, `Localhost`.
* Applying via:

  ```yaml
  seccompProfile:
    type: RuntimeDefault
  ```
* How to view blocked syscalls via audit logs.

---

## **10. SELinux Integration**

* Understand **SELinux contexts** (`user`, `role`, `type`, `level`).
* Example:

  ```yaml
  seLinuxOptions:
    level: "s0:c123,c456"
  ```
* How SELinux prevents cross-container interference.
* When you might see “permission denied” errors due to SELinux.

---

## **11. AppArmor Profiles**

* AppArmor vs seccomp vs SELinux.
* Applying profile via Pod annotation:

  ```yaml
  container.apparmor.security.beta.kubernetes.io/nginx: localhost/nginx-profile
  ```
* Restricting file or syscall access without root privileges.

---

## **12. procMount Control**

* `procMount: Default` vs `procMount: Unmasked`.
* How `/proc` controls system and process visibility.
* Why “unmasked” is dangerous in most production cases.

---

## **13. Windows SecurityContext Fields**

* Fields like `runAsUserName`, `gmsaCredentialSpecName`, `hostProcess`.
* Security differences between Windows and Linux containers.

---

## **14. Inheritance & Precedence**

* How Pod-level and Container-level `securityContext` interact.
* Rules of override — container-level always overrides Pod-level.
* Why defining both levels carefully avoids inconsistent behavior.

---

## **15. Network and Kernel Security Impacts**

* Relation of `NET_ADMIN` capability to modifying IP tables.
* Using `SYS_PTRACE` for debugging and why it’s dangerous.
* Limiting access to `/sys` and `/proc` to prevent host tampering.

---

## **16. DevSecOps: Policy Enforcement and Auditing**

* Writing **OPA Gatekeeper** or **Kyverno** policies to enforce:

  * No privileged containers.
  * Must drop all capabilities.
  * `runAsNonRoot` must be true.
* Example Kyverno policy snippet:

  ```yaml
  validationFailureAction: enforce
  match:
    resources:
      kinds: ["Pod"]
  validate:
    message: "Privileged containers not allowed!"
    pattern:
      spec:
        containers:
          - (securityContext):
              (privileged): "false"
  ```
* Automate scanning of Pod manifests in CI/CD.

---

## **17. Auditing & Monitoring**

* Audit logs to detect creation of risky Pods.
* Integration with Falco or Kubearmor for runtime security monitoring.
* Detect capability misuse in real time.

---

## **18. Debugging and Troubleshooting**

* Using `kubectl describe pod` → check `securityContext` applied values.
* Use `kubectl exec` to verify UID/GID:

  ```bash
  id
  whoami
  ```
* Common errors: “permission denied”, “operation not permitted”.
* Fixes when SELinux or seccomp blocks execution.

---

## **19. Compliance & Hardening Standards**

* How CIS Kubernetes Benchmark and NSA guidelines define required controls.
* How `securityContext` fields map to compliance checks.
* Example: “No container should run as root” (CIS 5.2.6).

---

## **20. Real-World Scenarios**

* Secure web server pod with least privilege.
* DevOps agent needing `SYS_ADMIN` capability.
* Database container requiring writable `/var/lib/mysql`.
* Enforcing read-only container images in production.

---

## **21. Interview & Certification Readiness**

Be ready to answer:

* Difference between `privileged` and `capabilities.add`.
* How to enforce non-root execution.
* What happens when seccomp profile blocks a syscall.
* Compare Pod-level vs container-level security context.
* How to debug SELinux denials.

---

## **22. Best Practices**

* Always set `runAsNonRoot: true`.
* Drop all capabilities unless required.
* Avoid privileged containers.
* Use `readOnlyRootFilesystem: true` whenever possible.
* Apply `RuntimeDefault` seccomp profile.
* Enforce policies using Kyverno/OPA.
* Test with compliance tools like `kubescape` or `trivy`.

---
