# **Checking Environment Variables in a Pod**

## **1. Use `env` or `printenv` inside the Pod**

Once the Pod is running, you can view its environment variables using the Linux commands `env` or `printenv`.

### Example

```bash
kubectl exec -it <pod-name> -- env
```

or

```bash
kubectl exec -it <pod-name> -- printenv
```

✅ Output (example):

```
HOSTNAME=app-pod
APP_MODE=production
DB_HOST=localhost
DB_PORT=5432
POD_NAME=app-pod
POD_NAMESPACE=default
```

### Explanation:

* `kubectl exec -it` → Opens an **interactive terminal** inside the container.
* `env` or `printenv` → Displays all environment variables currently available inside the container.

---

## **2. Check a Specific Environment Variable**

If you only want one specific variable:

```bash
kubectl exec -it <pod-name> -- printenv APP_MODE
```

✅ Output:

```
production
```

---

## **3. Open an Interactive Shell and Explore**

Sometimes you might want to explore interactively:

```bash
kubectl exec -it <pod-name> -- sh
```

Then inside the container shell:

```bash
env
# or
printenv
# or check specific
echo $DB_HOST
```

---

## **4. Using `kubectl describe pod` (Without Entering Container)**

If you just want to **see what environment variables were defined** in YAML (not their runtime values):

```bash
kubectl describe pod <pod-name>
```

Scroll to section:

```
Environment:
  APP_MODE:     production
  DB_HOST:      from configmap app-config, key=database_host
  DB_PASSWORD:  from secret app-secret, key=db_password
```

✅ Shows **source of each variable** (ConfigMap, Secret, literal, etc.), but **not secret values** themselves (for security).

---

## **5. Check via Pod YAML (Full Spec)**

To see full YAML configuration:

```bash
kubectl get pod <pod-name> -o yaml
```

Look for:

```yaml
spec:
  containers:
    - name: app
      env:
        - name: APP_MODE
          value: "production"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database_host
```

---

## **6. Debugging Secrets**

Secrets are base64-encoded.
If you want to confirm their content before being injected:

```bash
kubectl get secret db-secret -o yaml
```

Output:

```yaml
data:
  DB_USER: YWRtaW4=
  DB_PASSWORD: cGFzc3dvcmQxMjM=
```

To decode manually:

```bash
echo "YWRtaW4=" | base64 --decode
```

✅ Output:

```
admin
```

---

## **Summary Table**

| Method                          | Command                           | Shows Runtime Values? | Notes                                  |
| ------------------------------- | --------------------------------- | --------------------- | -------------------------------------- |
| Inside Pod (`env` / `printenv`) | `kubectl exec -it pod -- env`     | ✅ Yes                 | Best for verifying active environment  |
| Interactive shell               | `kubectl exec -it pod -- sh`      | ✅ Yes                 | Explore multiple vars                  |
| Describe Pod                    | `kubectl describe pod podname`    | ⚙️ Partial            | Shows source (ConfigMap, Secret, etc.) |
| YAML output                     | `kubectl get pod podname -o yaml` | ⚙️ Partial            | Shows declared variables               |
| Check Secret                    | `kubectl get secret name -o yaml` | ❌ Encoded             | Need `base64 --decode`                 |

---