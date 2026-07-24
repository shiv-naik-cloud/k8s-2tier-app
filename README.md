# Two-Tier Application on Kubernetes with GitOps

A production-style, multi-tier application deployed on Kubernetes and managed
declaratively through **GitOps with Argo CD**. Git is the single source of truth —
the cluster continuously syncs itself to match the manifests in this repository.

## 🏗️ Architecture

```
                        Browser (User)
                             │
                             ▼
              ┌──────────────────────────────┐
              │   FRONTEND TIER (nginx)        │
              │   Deployment: 3 replicas       │
              │   Service: NodePort (public)   │
              │   Config from: ConfigMap       │
              └───────────────┬──────────────┘
                              │  service discovery
                              │  (via DNS name: "redis-service")
                              ▼
              ┌──────────────────────────────┐
              │   BACKEND TIER (Redis)         │
              │   Deployment: 1 replica        │
              │   Service: ClusterIP (private) │
              │   Password from: Secret        │
              └──────────────────────────────┘

        All resources isolated in the "twotier-app" namespace
```

The frontend is publicly reachable through a **NodePort** service, while the
backend database is kept private using a **ClusterIP** service — accessible only
from inside the cluster. The frontend locates the backend by its **service DNS
name** rather than a hard-coded IP, demonstrating Kubernetes service discovery.

## 🔄 GitOps Flow (Argo CD)

```
   Developer                GitHub Repo               Argo CD              Cluster
       │                         │                       │                    │
       │  git commit + push ───▶ │                       │                    │
       │                         │  ◀── polls for change │                    │
       │                         │                       │  ── syncs state ──▶│
       │                         │                       │                    │
       │                         │  ◀────── continuously compares ───────────▶│
       │                         │        (drift is automatically reverted)   │
```

Nothing is deployed with `kubectl apply` by hand. A commit to this repository is
the deployment. Argo CD watches the repo, applies any change automatically, and
reverts any manual drift in the cluster back to what Git says.

## 🧰 Tech Stack

- **Kubernetes** (minikube for local development)
- **Argo CD** — GitOps continuous delivery
- **nginx** — frontend web server
- **Redis 7** — backend datastore
- Declarative **YAML** manifests (Infrastructure as Code)

## 📁 Project Structure

```
k8s-2tier-app/
├── manifests/
│   ├── namespace.yaml            # Isolated namespace for the project
│   ├── backend-secret.yaml       # Redis password (Secret)
│   ├── backend-deployment.yaml   # Redis Deployment
│   ├── backend-service.yaml      # Redis Service (ClusterIP - private)
│   ├── frontend-configmap.yaml   # Frontend configuration
│   ├── frontend-deployment.yaml  # nginx Deployment
│   └── frontend-service.yaml     # nginx Service (NodePort - public)
├── argocd/
│   └── application.yaml          # Argo CD Application (the GitOps config itself)
└── screenshots/
```

The Argo CD `Application` is kept in Git as well, so the GitOps setup itself is
version controlled and reproducible — not something that only exists as clicks in
a UI.

## 🚀 How to Run

**Prerequisites:** Docker, minikube, and kubectl installed.

### 1. Start the cluster

```bash
minikube start --driver=docker
```

### 2. Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# wait until all pods are Running
kubectl get pods -n argocd
```

### 3. Access the Argo CD UI

```bash
# get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
# (decode the base64 value)

# open a tunnel to the UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open **https://localhost:8080** and log in as `admin`.

### 4. Deploy the application via GitOps

```bash
kubectl apply -f argocd/application.yaml
```

That's it. Argo CD reads the `manifests/` folder from this repository and creates
everything — namespace, secret, config, deployments, and services.

### 5. Verify

```bash
kubectl get all -n twotier-app
minikube service frontend-service -n twotier-app
```

## 🧪 Demonstrating GitOps

**Change through Git (the intended workflow):**

Edit `replicas` in `manifests/frontend-deployment.yaml`, commit, and push. Argo CD
detects the new commit and scales the deployment automatically — no `kubectl`
involved. Only the difference is applied; existing healthy Pods are left running.

**Self-healing (drift correction):**

```bash
kubectl scale deploy frontend -n twotier-app --replicas=5
```

Argo CD detects that the cluster no longer matches Git and scales it back to the
committed value. Manual changes to the cluster do not survive — Git wins.

## ✅ Verifying Service Discovery

```bash
kubectl exec -n twotier-app <frontend-pod-name> -- getent hosts redis-service
```

This resolves to the backend's internal cluster IP
(`redis-service.twotier-app.svc.cluster.local`), proving frontend-to-backend
communication works without any hard-coded IP addresses.

## 🔑 Key Concepts Demonstrated

| Concept | Where it's used |
|---------|-----------------|
| **Namespace** | Isolates all project resources (`twotier-app`) |
| **Deployment** | Self-healing & scaling for both tiers |
| **Service (NodePort)** | Exposes the frontend publicly |
| **Service (ClusterIP)** | Keeps the backend private (security) |
| **ConfigMap** | Externalizes frontend configuration from the image |
| **Secret** | Stores the database password separately from config |
| **Service Discovery** | Frontend finds backend via DNS name |
| **GitOps** | Git as the single source of truth for cluster state |
| **Auto-sync** | Commits are applied to the cluster automatically |
| **Self-heal** | Manual drift is detected and reverted |
| **Pruning** | Resources removed from Git are removed from the cluster |

## 🧹 Cleanup

```bash
kubectl delete -f argocd/application.yaml   # remove the app (prune deletes resources)
kubectl delete namespace argocd             # remove Argo CD
minikube stop
```

## 📌 Notes

- `stringData` is used in the Secret so the password can be written in plain text
  and Kubernetes handles base64 encoding automatically.
- Note that base64 is **encoding, not encryption**. In a production environment,
  secrets would be further protected using tools such as sealed-secrets or an
  external secrets manager (e.g. AWS Secrets Manager), combined with RBAC.

---

*Built as a hands-on Kubernetes learning project — every manifest written and
tested manually, then converted to a GitOps workflow with Argo CD.*
