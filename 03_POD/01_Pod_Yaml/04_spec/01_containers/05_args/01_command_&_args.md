## **Dockerfile Reference (Base Behavior)**

Let’s assume the Docker image (`busybox`) has this structure:

```dockerfile
# Example Dockerfile for busybox
ENTRYPOINT ["sh"]
CMD ["-c", "echo Default CMD running..."]
```

Now, when you create Pods with different combinations of `command` and `args`,
Kubernetes overrides these values differently.

---

## ⚙️ **Case 1: Only `args` Provided**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: args-only
spec:
  containers:
    - name: busybox
      image: busybox
      args: ["-c", "echo Hello from ARGS only"]
```

### Explanation:

* Since **`command`** is not defined, Kubernetes uses the image’s **ENTRYPOINT** (`sh`).
* Your **`args`** replace the image’s default CMD.

✅ **Final executed command inside container:**

```bash
sh -c "echo Hello from ARGS only"
```

✅ **When to use:**
When you’re happy with the image’s ENTRYPOINT but want to change what it does.

---

## ⚙️ **Case 2: Only `command` Provided**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-only
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["echo", "Hello from COMMAND only"]
```

### Explanation:

* `command` overrides the image’s ENTRYPOINT entirely.
* Since `args` is missing, **no CMD arguments** are passed.

✅ **Final executed command inside container:**

```bash
echo "Hello from COMMAND only"
```

✅ **When to use:**
When you want to **replace** what the image does completely
(e.g., instead of starting nginx, you just want to sleep or debug).

---

## ⚙️ **Case 3: Both `command` + `args` Provided**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-and-args
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["sh", "-c"]
      args: ["echo Hello from both COMMAND and ARGS && sleep 5"]
```

### Explanation:

* The `command` replaces ENTRYPOINT (`sh -c`).
* The `args` are passed **as parameters** to that command.

✅ **Final executed command inside container:**

```bash
sh -c "echo Hello from both COMMAND and ARGS && sleep 5"
```

✅ **When to use:**
When you want full control — you specify both the entry command and what it should execute.

---

## 🧾 **Summary Table**

| Case                | Defined Fields     | Docker ENTRYPOINT | Docker CMD   | Final Execution                     | When to Use                         |
| ------------------- | ------------------ | ----------------- | ------------ | ----------------------------------- | ----------------------------------- |
| **1. Only args**    | ✅ args only        | ✅ Used from image | ❌ Overridden | `sh -c "echo Hello from ARGS only"` | Modify image behavior               |
| **2. Only command** | ✅ command only     | ❌ Replaced        | ❌ Ignored    | `echo "Hello from COMMAND only"`    | Completely replace startup          |
| **3. Both**         | ✅ command + ✅ args | ❌ Replaced        | ✅ Used       | `sh -c "echo Hello from both..."`   | Full control over container startup |

---

## **Mnemonic Tip (स्मरण टीप)**

Think of it like **“C before A” → Command before Args**

> “First decide **what to run** (command),
> then decide **how to run it** (args).”

---

