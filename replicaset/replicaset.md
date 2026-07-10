# ReplicaSet – Basic `kubectl` Commands

A **ReplicaSet** ensures a specified number of identical Pods (`replicas`) are
always running. If a Pod dies, the ReplicaSet automatically creates a new one.
The manifest here (`replicaset.yaml`) keeps **3 replicas** of an nginx Pod
selected by the label `app: myapp`.

> Note: In real projects you usually use a **Deployment** (which manages
> ReplicaSets for you). Using a ReplicaSet directly is mainly for learning.

---

## 1. Create / Apply a ReplicaSet


| Command                             | Use                                           |
| ----------------------------------- | --------------------------------------------- |
| `kubectl apply -f replicaset.yaml`  | Create or update the ReplicaSet. Recommended. |
| `kubectl create -f replicaset.yaml` | Create it; fails if it already exists.        |


**Syntax**

```bash
kubectl apply -f <file-name>.yaml
```

---



## 2. View / List ReplicaSets


| Command                           | Use                                               |
| --------------------------------- | ------------------------------------------------- |
| `kubectl get replicaset`          | List all ReplicaSets.                             |
| `kubectl get rs`                  | Short alias for `replicaset`.                     |
| `kubectl get rs myapp-replicaset` | Show one ReplicaSet (DESIRED / CURRENT / READY).  |
| `kubectl get pods -l app=myapp`   | List the Pods this ReplicaSet manages (by label). |
| `kubectl get rs -o wide`          | Show extra info like images and selectors.        |


**Syntax**

```bash
kubectl get rs [<name>] [-o wide]
kubectl get pods -l <label-key>=<label-value>
```

---



## 3. Inspect / Describe a ReplicaSet


| Command                                   | Use                                          |
| ----------------------------------------- | -------------------------------------------- |
| `kubectl describe rs myapp-replicaset`    | Full details + events (creations, failures). |
| `kubectl get rs myapp-replicaset -o yaml` | Show live config as YAML.                    |


**Syntax**

```bash
kubectl describe rs <name>
```

---



## 4. Scale a ReplicaSet


| Command                                          | Use                                 |
| ------------------------------------------------ | ----------------------------------- |
| `kubectl scale rs myapp-replicaset --replicas=5` | Change the number of Pods to 5.     |
| `kubectl scale rs myapp-replicaset --replicas=0` | Scale down to zero (stop all Pods). |


**Syntax**

```bash
kubectl scale rs <name> --replicas=<count>
```

Tip: You can also edit `replicas:` in `replicaset.yaml` and re-run
`kubectl apply -f replicaset.yaml`.

---



## 5. Test Self-Healing


| Command                         | Use                                                                 |
| ------------------------------- | ------------------------------------------------------------------- |
| `kubectl delete pod <pod-name>` | Delete one managed Pod — the ReplicaSet recreates it automatically. |
| `kubectl get pods -w`           | Watch the replacement Pod appear in real time.                      |


---



## 6. Edit / Delete a ReplicaSet


| Command                              | Use                                      |
| ------------------------------------ | ---------------------------------------- |
| `kubectl edit rs myapp-replicaset`   | Edit the live ReplicaSet in your editor. |
| `kubectl delete rs myapp-replicaset` | Delete the ReplicaSet **and** its Pods.  |
| `kubectl delete -f replicaset.yaml`  | Delete using the manifest file.          |


**Syntax**

```bash
kubectl delete rs <name>
kubectl delete -f <file-name>.yaml
```

---



## Quick Tip

`kubectl delete rs myapp-replicaset --cascade=orphan` deletes only the ReplicaSet
but **keeps** its Pods running.