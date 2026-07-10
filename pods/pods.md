# Pods – Basic `kubectl` Commands

A **Pod** is the smallest deployable unit in Kubernetes. It wraps one or more
containers that share the same network and storage. The manifests in this folder
(`pod.yaml`, `nginx.yaml`) define single-container nginx Pods.

---

## 1. Create / Apply a Pod

| Command | Use |
|---------|-----|
| `kubectl apply -f pod.yaml` | Create the Pod (or update it if it already exists). Recommended way. |
| `kubectl create -f pod.yaml` | Create the Pod. Fails if it already exists. |
| `kubectl run nginx-pod --image=nginx:latest` | Quickly create a Pod without a YAML file. |

**Syntax**

```bash
kubectl apply -f <file-name>.yaml
kubectl run <pod-name> --image=<image-name>
```

---

## 2. View / List Pods

| Command | Use |
|---------|-----|
| `kubectl get pods` | List all Pods in the current namespace. |
| `kubectl get pods -o wide` | List Pods with extra info (node, IP). |
| `kubectl get pod nginx-pod` | Show a single Pod's status. |
| `kubectl get pods --show-labels` | List Pods along with their labels (e.g. `env=production`). |
| `kubectl get pods -w` | Watch Pods live as their status changes. |

**Syntax**

```bash
kubectl get pods [-o wide] [--show-labels]
kubectl get pod <pod-name>
```

---

## 3. Inspect / Describe a Pod

| Command | Use |
|---------|-----|
| `kubectl describe pod nginx-pod` | Full details + events (great for debugging). |
| `kubectl get pod nginx-pod -o yaml` | Show the Pod's live config as YAML. |
| `kubectl logs nginx-pod` | View container logs. |
| `kubectl logs -f nginx-pod` | Stream (follow) container logs live. |

**Syntax**

```bash
kubectl describe pod <pod-name>
kubectl logs [-f] <pod-name>
```

---

## 4. Interact with a Pod

| Command | Use |
|---------|-----|
| `kubectl exec -it nginx-pod -- sh` | Open a shell inside the Pod's container. |
| `kubectl exec nginx-pod -- ls /` | Run a single command inside the Pod. |
| `kubectl port-forward nginx-pod 8080:80` | Forward local port 8080 to Pod port 80. |
| `kubectl cp nginx-pod:/path ./local` | Copy files from the Pod to your machine. |

**Syntax**

```bash
kubectl exec -it <pod-name> -- <command>
kubectl port-forward <pod-name> <local-port>:<pod-port>
```

---

## 5. Delete a Pod

| Command | Use |
|---------|-----|
| `kubectl delete pod nginx-pod` | Delete a Pod by name. |
| `kubectl delete -f pod.yaml` | Delete the Pod defined in the file. |

**Syntax**

```bash
kubectl delete pod <pod-name>
kubectl delete -f <file-name>.yaml
```

---

## Quick Tip
Use `kubectl apply -f pod.yaml --dry-run=client` to validate the YAML **without**
actually creating the Pod.
