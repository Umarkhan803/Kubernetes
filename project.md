# Real-World Kubernetes Scenario: Deploying a MERN App on Docker Desktop

## The Scenario

You're the DevOps engineer for a startup. The team has a MERN app:

- **Frontend**: React (served as static build via Nginx)
- **Backend**: Node/Express API
- **Database**: MongoDB

You need to containerize and deploy this to Kubernetes so it behaves like a real production system: survives pod crashes, scales under load, keeps secrets out of code, and supports zero-downtime deploys.

This mirrors real interview asks: *"Walk me through how you'd deploy a full-stack app on K8s."*

---

## Step 0: Enable Kubernetes in Docker Desktop

**How:** Docker Desktop → Settings → Kubernetes → check "Enable Kubernetes" → Apply & Restart.

**Why:** Docker Desktop ships a single-node K8s cluster (control plane + worker on one machine) using `kubeadm` under the hood. It's not production-grade infrastructure, but the API objects (Deployments, Services, etc.) behave identically to a real cloud cluster — this is why practicing here transfers directly to EKS/AKS/GKE.

Verify:

```bash
kubectl config get-contexts
kubectl config use-context docker-desktop
kubectl get nodes
```

**Purpose of this check:** Confirms `kubectl` is talking to the right cluster. In real jobs you'll often have multiple contexts (staging, prod, personal) — mixing them up is a classic, career-limiting mistake. Always verify context before applying anything.

---



## Step 1: Namespace — Isolating the app

```bash
kubectl create namespace mern-prod
kubectl config set-context --current --namespace=mern-prod
```

**Why we do this:** In production, a cluster hosts many apps/teams. Namespaces give logical isolation — separate RBAC, resource quotas, and network policies per namespace. Deploying everything into `default` is a beginner mistake that becomes painful once you have 10+ services.

**Benefit:** You can `kubectl delete namespace mern-prod` to nuke the entire environment cleanly during testing — no orphaned resources.

---



## Step 2: Containerize each service

You already have (or need) a `Dockerfile` per service. Example for the backend:

```dockerfile
# backend/Dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

FROM node:20-alpine
WORKDIR /app
COPY --from=build /app .
USER node
EXPOSE 5000
CMD ["node", "server.js"]
```

**Why multi-stage +** `USER node`**:**

- Multi-stage keeps the final image lean (no build tools/dev deps shipped).
- Running as non-root (`USER node`) is a security requirement in most real clusters — Pod Security Standards / OPA policies will reject root containers.

Build and load into Docker Desktop's local registry (no push needed since Docker Desktop's K8s shares the same image cache as Docker Engine):

```bash
docker build -t mern-backend:v1 ./backend
docker build -t mern-frontend:v1 ./frontend
```

**Why this matters:** In real clusters (EKS/GKE) you'd push to ECR/GCR and reference the registry path. Locally, Docker Desktop's K8s can see images built with `docker build` directly — a convenience that trips people up when they later move to a real registry and forget `imagePullPolicy`.

---



## Step 3: Secrets and Config — never hardcode credentials

```bash
kubectl create secret generic mongo-secret \
  --from-literal=MONGO_ROOT_USER=admin \
  --from-literal=MONGO_ROOT_PASSWORD='ChangeMe123!' \
  -n mern-prod

kubectl create configmap backend-config \
  --from-literal=MONGO_HOST=mongo-service \
  --from-literal=NODE_ENV=production \
  -n mern-prod
```

**Why split Secret vs ConfigMap:**

- **Secrets** hold sensitive data (base64-encoded at rest, and in real clusters typically encrypted via KMS + RBAC-restricted).
- **ConfigMaps** hold non-sensitive config (env names, feature flags).

**Purpose:** This decouples configuration from the container image. The *same image* can run in dev/staging/prod just by swapping which ConfigMap/Secret it's bound to — this is the core 12-factor app principle interviewers look for.

**Real-world resolve:** If you accidentally hardcode a DB password in a Dockerfile or repo, that's a security incident — you'd rotate the credential immediately, purge git history, and move it to a Secret (or better, a vault like AWS Secrets Manager synced via External Secrets Operator).

---



## Step 4: MongoDB — StatefulSet + PersistentVolumeClaim

Databases need stable storage and identity, so we use a `StatefulSet`, not a `Deployment`.

```yaml
# mongo-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
  namespace: mern-prod
spec:
  serviceName: mongo-service
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
        - name: mongo
          image: mongo:7
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef: { name: mongo-secret, key: MONGO_ROOT_USER }
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef: { name: mongo-secret, key: MONGO_ROOT_PASSWORD }
          volumeMounts:
            - name: mongo-data
              mountPath: /data/db
          resources:
            requests: { cpu: "250m", memory: "256Mi" }
            limits: { cpu: "500m", memory: "512Mi" }
  volumeClaimTemplates:
    - metadata:
        name: mongo-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
  namespace: mern-prod
spec:
  clusterIP: None   # headless service
  selector:
    app: mongo
  ports:
    - port: 27017
```

**Why StatefulSet over Deployment for the DB:**

- Gives Mongo a stable network identity (`mongo-0.mongo-service`) — important if you scale to a replica set later.
- Each pod gets its **own** PVC (`volumeClaimTemplates`), so if the pod restarts, it reattaches to the *same* disk instead of a random one.

**Why** `resources.requests/limits`**:** Without these, one runaway pod can starve the node (noisy neighbor problem). `requests` is what the scheduler reserves; `limits` is the hard ceiling. This is one of the most commonly missing things in junior K8s manifests — always call it out in interviews.

**Why headless service (**`clusterIP: None`**):** StatefulSets need direct pod-to-pod DNS resolution, not load-balanced routing, since each replica is not interchangeable (each may hold different data as primary/secondary).

Apply:

```bash
kubectl apply -f mongo-statefulset.yaml
kubectl get pods -w
```

---



## Step 5: Backend — Deployment + probes

```yaml
# backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: mern-prod
spec:
  replicas: 2
  selector:
    matchLabels: { app: backend }
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }
  template:
    metadata:
      labels: { app: backend }
    spec:
      containers:
        - name: backend
          image: mern-backend:v1
          imagePullPolicy: IfNotPresent
          ports: [{ containerPort: 5000 }]
          envFrom:
            - configMapRef: { name: backend-config }
            - secretRef: { name: mongo-secret }
          readinessProbe:
            httpGet: { path: /health, port: 5000 }
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /health, port: 5000 }
            initialDelaySeconds: 15
            periodSeconds: 20
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits: { cpu: "300m", memory: "256Mi" }
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: mern-prod
spec:
  selector: { app: backend }
  ports:
    - port: 5000
      targetPort: 5000
```

**Why** `replicas: 2`**:** High availability — if one pod crashes or is being updated, the other keeps serving traffic. This is the whole point of using K8s over a single Docker container.

**Why readiness vs liveness probes (this trips up almost everyone at first):**

- **Readiness probe** — "is this pod ready to receive traffic?" If it fails, the pod is pulled out of the Service's load-balancing pool, but *not* restarted. Use case: pod is alive but still connecting to Mongo on startup.
- **Liveness probe** — "is this pod still healthy?" If it fails repeatedly, K8s kills and restarts the container. Use case: app is deadlocked/hung.

**Real production incident this prevents:** Without readiness probes, K8s sends traffic to a pod before it's finished connecting to the DB → users get 500 errors during every rollout. This is one of the most common real-world outages.

**Why** `maxUnavailable: 0, maxSurge: 1`**:** Ensures zero-downtime rolling updates — K8s spins up 1 new pod before killing an old one, rather than taking capacity down first.

You'll need a `/health` route in Express:

```js
app.get('/health', (req, res) => res.status(200).send('OK'));
```

---



## Step 6: Frontend — Deployment + Service

```yaml
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: mern-prod
spec:
  replicas: 2
  selector:
    matchLabels: { app: frontend }
  template:
    metadata:
      labels: { app: frontend }
    spec:
      containers:
        - name: frontend
          image: mern-frontend:v1
          ports: [{ containerPort: 80 }]
          resources:
            requests: { cpu: "50m", memory: "64Mi" }
            limits: { cpu: "150m", memory: "128Mi" }
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: mern-prod
spec:
  selector: { app: frontend }
  ports:
    - port: 80
      targetPort: 80
```

---



## Step 7: Ingress — single entry point

Enable the Ingress controller (Docker Desktop supports NGINX ingress):

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mern-ingress
  namespace: mern-prod
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: mern.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service: { name: backend-service, port: { number: 5000 } }
          - path: /
            pathType: Prefix
            backend:
              service: { name: frontend-service, port: { number: 80 } }
```

Add to `/etc/hosts` (or Windows equivalent): `127.0.0.1 mern.local`

**Why Ingress instead of exposing every Service via** `NodePort`**/**`LoadBalancer`**:** One entry point, path-based routing (`/api` → backend, `/` → frontend), and it's where you'd later attach TLS certs (cert-manager + Let's Encrypt in real prod). Exposing every service individually doesn't scale past a handful of microservices and complicates DNS/SSL management.

---



## Step 8: Horizontal Pod Autoscaler (simulating real load handling)

```bash
kubectl autoscale deployment backend --cpu-percent=70 --min=2 --max=6 -n mern-prod
```

**Why:** Real traffic isn't constant. HPA watches CPU (or custom metrics) and adds/removes pods automatically. This is what interviewers mean by "how do you handle scale" — you don't manually run more `kubectl scale` commands, you let the cluster react.

**Caveat to mention in interviews:** HPA needs the `metrics-server` running to read CPU usage; on Docker Desktop you may need to install it separately since it's not bundled by default.

---



## Step 9: Deploy everything and verify

```bash
kubectl apply -f mongo-statefulset.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f ingress.yaml

kubectl get all -n mern-prod
kubectl get ingress -n mern-prod
```

Visit `http://mern.local` in the browser.

---



## Step 10: Troubleshooting — the part that actually matters in interviews


| Symptom                     | Command to diagnose                         | Likely cause                                                                           |
| --------------------------- | ------------------------------------------- | -------------------------------------------------------------------------------------- |
| Pod stuck `Pending`         | `kubectl describe pod <name>`               | Insufficient CPU/memory on node, or PVC not bound                                      |
| Pod `CrashLoopBackOff`      | `kubectl logs <pod> --previous`             | App crashing on start — often missing env var or bad DB connection                     |
| Pod `ImagePullBackOff`      | `kubectl describe pod <name>`               | Wrong image tag, or image not built locally before Docker Desktop K8s tried to pull it |
| Service not routing traffic | `kubectl get endpoints <svc>`               | Label selector mismatch between Service and Pod                                        |
| Ingress 404                 | `kubectl describe ingress`                  | `ingressClassName` missing, or controller not installed                                |
| Rolling update hangs        | `kubectl rollout status deployment/backend` | New pods failing readiness probe — check `/health` endpoint                            |


**Why this table matters:** In real production, you rarely get "code is broken" — you get symptoms like these. The debugging *sequence* (describe → logs → events → endpoints) is exactly what's evaluated in a DevOps interview or an actual on-call incident.

**Rollback command** (know this cold — it's asked constantly):

```bash
kubectl rollout undo deployment/backend -n mern-prod
kubectl rollout history deployment/backend -n mern-prod
```

---



## What you've now demonstrated (useful framing for interviews/resume)

- Multi-service orchestration (Deployment vs StatefulSet — and *why* each was chosen)
- Config/secret separation from image (12-factor principles)
- Zero-downtime rolling updates via readiness/liveness probes
- Resource requests/limits to prevent noisy-neighbor issues
- Ingress-based traffic routing instead of per-service exposure
- Autoscaling based on load
- A real troubleshooting workflow, not just "kubectl apply and hope"

This directly maps to the Kubernetes/CI-CD points already in your resume — you can describe this exact flow when asked "walk me through a K8s deployment you've done."

## Suggested next steps

- Add a `values.yaml` and convert this into a **Helm chart** (very commonly asked about in DevOps interviews)
- Wire this into a CI/CD pipeline (GitHub Actions → build image → push to a registry → `kubectl apply` or ArgoCD sync)
- Try deliberately breaking things (bad env var, wrong probe path) and practicing the diagnosis table above until it's muscle memory

