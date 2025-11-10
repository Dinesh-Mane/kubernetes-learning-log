
> The `kubectl run` command is **imperative**, meaning you tell Kubernetes exactly what to do.
> For repeatable deployments, convert the command into YAML using:
>
> ```bash
> kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
> ```

---

## 1. `--image`

**Meaning:** Specifies the container image to use inside the Pod.  
**When to Use:** Always (mandatory flag).  
**Use Case:** Launch a simple web or test Pod.  

```bash
kubectl run nginx --image=nginx:latest
```

✅ Creates a Pod named **nginx** using the **nginx:latest** image.

---

## 2. `--port`

**Meaning:** Exposes a specific port from the container.  
**When to Use:** When app listens on a specific port.  
**Use Case:** Expose internal app port for Service usage.  

```bash
kubectl run hazelcast --image=hazelcast/hazelcast --port=5701
```

✅ Exposes port **5701**.

---

## 3. `--env`

**Meaning:** Sets environment variables inside the container.  
**When to Use:** When app needs config via environment variables.  
**Use Case:** Pass DB credentials or config keys.  

```bash
kubectl run myapp --image=myapp:1.0 --env="DB_USER=admin" --env="DB_PASS=secret"
```

✅ Adds DB_USER and DB_PASS to container environment.

---

## 4. `--labels`

**Meaning:** Adds metadata labels to the Pod.  
**When to Use:** For grouping or filtering Pods.  
**Use Case:** Service selection or monitoring.  

```bash
kubectl run hazelcast --image=hazelcast/hazelcast --labels="app=hazelcast,env=prod"
```

✅ Adds labels `app=hazelcast`, `env=prod`.

---

## 5. `--dry-run`

**Meaning:** Tests command without creating resources.  
**When to Use:** To preview YAML before applying.  
**Use Case:** Generate YAML for version control.  

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml
```

✅ Prints YAML without creating Pod.

---

## 6. `--overrides`

**Meaning:** Overrides PodSpec inline using JSON.  
**When to Use:** To modify Pod settings temporarily.  
**Use Case:** Add nodeSelector or tolerations on the fly.  

```bash
kubectl run nginx --image=nginx \
--overrides='{ "apiVersion":"v1","spec":{"nodeSelector":{"disktype":"ssd"}}}'
```

✅ Schedules Pod only on SSD nodes.

---

## 7. `-i` / `--stdin`

**Meaning:** Keeps stdin open for input.  
**When to Use:** For interactive input inside a container.  
**Use Case:** Run shell inside Pod.  
```bash
kubectl run -i busybox --image=busybox --restart=Never -- sh
```

✅ Opens shell session in container.

---

## 8. `-t` / `--tty`

**Meaning:** Allocates a pseudo-terminal (TTY).  
**When to Use:** With `-i` for interactive sessions.  
**Use Case:** Debug interactively via terminal.  

```bash
kubectl run -it busybox --image=busybox --restart=Never
```

✅ Opens terminal for BusyBox Pod.

---

## 9. `--restart`

**Meaning:** Sets Pod restart policy (`Always`, `OnFailure`, `Never`).   
**When to Use:** For short-lived or test Pods.  
**Use Case:** Stop Pod from restarting after exit.  

```bash
kubectl run testpod --image=busybox --restart=Never
```

✅ Pod won’t restart after it exits.

---

## 10. `--command`

**Meaning:** Treats following arguments as a command to run.  
**When to Use:** To override default container entrypoint.  
**Use Case:** Run custom command inside Pod.  

```bash
kubectl run mypod --image=busybox --command -- ls /etc
```

✅ Runs `ls /etc` instead of default CMD.

---

## 11. `-- <args>`

**Meaning:** Passes arguments to container’s default command.  
**When to Use:** Add flags to existing app command.  
**Use Case:** Enable debug or verbose mode.  

```bash
kubectl run nginx --image=nginx -- --debug --verbose
```

✅ Adds `--debug --verbose` to default nginx command.

---

## 12. `--generator` (⚠️ Deprecated)

**Meaning:** Used older generators for different API objects.  
**When to Use:** ❌ Avoid — deprecated post v1.18.  
**Use Case:** Not recommended anymore.  

---

