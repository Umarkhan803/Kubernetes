# ReplicationController – Basic `kubectl` Commands

A **ReplicationController (RC)** is the *original* Kubernetes controller that
ensures a specified number of Pod **replicas** are always running. If a Pod dies,
the RC creates a new one to keep the count stable.

> Note: The ReplicationController is the **older** version of a **ReplicaSet**.
> A ReplicaSet supports more powerful label selectors (`matchLabels`,
> `matchExpressions`), while an RC only supports simple equality selectors.
> In modern projects you should use a **Deployment** instead. RC is mainly for
> learning the fundamentals.

```
ReplicationController  ->  Pods   (older)
ReplicaSet            ->  Pods   (newer, richer selectors)
Deployment -> ReplicaSet -> Pods (recommended)
```

---



## Sample Manifest (`replicationController.yaml`)

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp-rc
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    app: myapp
  template:
    metadata:
      name: nginx-2
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx-container
          image: nginx:latest
```

Key difference vs ReplicaSet: the RC `selector` is a **flat map** (`app: myapp`),
not `matchLabels`.

---



## 1. Create / Apply a ReplicationController


| Command                                        | Use                                    |
| ---------------------------------------------- | -------------------------------------- |
| `kubectl apply -f replicationController.yaml`  | Create or update the RC. Recommended.  |
| `kubectl create -f replicationController.yaml` | Create it; fails if it already exists. |


**Syntax**

```bash
kubectl apply -f <file-name>.yaml
```

---



## 2. View / List ReplicationControllers


| Command                             | Use                                        |
| ----------------------------------- | ------------------------------------------ |
| `kubectl get replicationcontroller` | List all RCs.                              |
| `kubectl get rc`                    | Short alias for `replicationcontroller`.   |
| `kubectl get rc myapp-rc`           | Show one RC (DESIRED / CURRENT / READY).   |
| `kubectl get pods -l app=myapp`     | List the Pods this RC manages (by label).  |
| `kubectl get rc -o wide`            | Show extra info like images and selectors. |


**Syntax**

```bash
kubectl get rc [<name>] [-o wide]
kubectl get pods -l <label-key>=<label-value>
```

---



## 3. Inspect / Describe a ReplicationController


| Command                           | Use                                          |
| --------------------------------- | -------------------------------------------- |
| `kubectl describe rc myapp-rc`    | Full details + events (creations, failures). |
| `kubectl get rc myapp-rc -o yaml` | Show live config as YAML.                    |


**Syntax**

```bash
kubectl describe rc <name>
```

---



## 4. Scale a ReplicationController


| Command                                  | Use                                 |
| ---------------------------------------- | ----------------------------------- |
| `kubectl scale rc myapp-rc --replicas=5` | Change the number of Pods to 5.     |
| `kubectl scale rc myapp-rc --replicas=0` | Scale down to zero (stop all Pods). |


**Syntax**

```bash
kubectl scale rc <name> --replicas=<count>
```

Tip: You can also edit `replicas:` in the YAML and re-run
`kubectl apply -f replicationController.yaml`.

---



## 5. Test Self-Healing


| Command                         | Use                                                         |
| ------------------------------- | ----------------------------------------------------------- |
| `kubectl delete pod <pod-name>` | Delete one managed Pod — the RC recreates it automatically. |
| `kubectl get pods -w`           | Watch the replacement Pod appear in real time.              |


---



## 6. Edit / Delete a ReplicationController


| Command                                        | Use                              |
| ---------------------------------------------- | -------------------------------- |
| `kubectl edit rc myapp-rc`                     | Edit the live RC in your editor. |
| `kubectl delete rc myapp-rc`                   | Delete the RC **and** its Pods.  |
| `kubectl delete -f replicationController.yaml` | Delete using the manifest file.  |


**Syntax**

```bash
kubectl delete rc <name>
kubectl delete -f <file-name>.yaml
```

---



## Quick Tip

`kubectl delete rc myapp-rc --cascade=orphan` deletes only the RC but **keeps**
its Pods running.