# Deployment – Basic `kubectl` Commands

A **Deployment** is the most common way to run apps in Kubernetes. It manages a
**ReplicaSet**, which in turn manages the **Pods**. Deployments add powerful
features on top: rolling updates, rollbacks, and easy scaling.
The manifest here (`deployment.yaml`) runs **3 replicas** of `nginx:alpine`
selected by the label `app: myapp`.

```
Deployment  ->  ReplicaSet  ->  Pods
```

---

## 1. Create / Apply a Deployment

| Command | Use |
|---------|-----|
| `kubectl apply -f deployment.yaml` | Create or update the Deployment. Recommended. |
| `kubectl create -f deployment.yaml` | Create it; fails if it already exists. |
| `kubectl create deployment nginx --image=nginx:alpine` | Create a Deployment without YAML. |

**Syntax**

```bash
kubectl apply -f <file-name>.yaml
kubectl create deployment <name> --image=<image>
```

---

## 2. View / List Deployments

| Command | Use |
|---------|-----|
| `kubectl get deployment` | List all Deployments. |
| `kubectl get deploy` | Short alias for `deployment`. |
| `kubectl get deploy nginx-deployment` | Show one Deployment (READY / UP-TO-DATE / AVAILABLE). |
| `kubectl get rs` | See the ReplicaSet created by this Deployment. |
| `kubectl get pods -l app=myapp` | See the Pods managed by this Deployment. |

**Syntax**

```bash
kubectl get deploy [<name>] [-o wide]
```

---

## 3. Inspect / Describe a Deployment

| Command | Use |
|---------|-----|
| `kubectl describe deploy nginx-deployment` | Full details + rollout events. |
| `kubectl get deploy nginx-deployment -o yaml` | Show live config as YAML. |

**Syntax**

```bash
kubectl describe deploy <name>
```

---

## 4. Scale a Deployment

| Command | Use |
|---------|-----|
| `kubectl scale deploy nginx-deployment --replicas=5` | Change number of Pods to 5. |
| `kubectl autoscale deploy nginx-deployment --min=2 --max=10 --cpu-percent=80` | Auto-scale based on CPU. |

**Syntax**

```bash
kubectl scale deploy <name> --replicas=<count>
```

---

## 5. Update the App (Rolling Update)

| Command | Use |
|---------|-----|
| `kubectl set image deploy/nginx-deployment nginx-container=nginx:1.25` | Update the container image (triggers a rolling update). |
| `kubectl edit deploy nginx-deployment` | Edit the live Deployment in your editor. |
| `kubectl rollout status deploy/nginx-deployment` | Watch the update progress. |

**Syntax**

```bash
kubectl set image deploy/<name> <container-name>=<new-image>
kubectl rollout status deploy/<name>
```

---

## 6. Rollback / History

| Command | Use |
|---------|-----|
| `kubectl rollout history deploy/nginx-deployment` | List previous revisions. |
| `kubectl rollout undo deploy/nginx-deployment` | Roll back to the previous version. |
| `kubectl rollout undo deploy/nginx-deployment --to-revision=2` | Roll back to a specific revision. |
| `kubectl rollout restart deploy/nginx-deployment` | Restart all Pods (e.g. to pick up config changes). |

**Syntax**

```bash
kubectl rollout undo deploy/<name> [--to-revision=<n>]
```

---

## 7. Expose the Deployment (Service)

| Command | Use |
|---------|-----|
| `kubectl expose deploy nginx-deployment --port=80 --type=NodePort` | Create a Service to access the app. |
| `kubectl port-forward deploy/nginx-deployment 8080:80` | Forward a local port for quick testing. |

**Syntax**

```bash
kubectl expose deploy <name> --port=<port> --type=<ClusterIP|NodePort|LoadBalancer>
```

---

## 8. Delete a Deployment

| Command | Use |
|---------|-----|
| `kubectl delete deploy nginx-deployment` | Delete the Deployment (also removes its ReplicaSet + Pods). |
| `kubectl delete -f deployment.yaml` | Delete using the manifest file. |

**Syntax**

```bash
kubectl delete deploy <name>
kubectl delete -f <file-name>.yaml
```

---

## Quick Tip
Add `--record` to update commands (or use annotations) to keep a clear rollout
history, and always test with `--dry-run=client -o yaml` before applying changes.
