# Two-Tier Application on Kubernetes

A production-style, multi-tier application deployed on Kubernetes, demonstrating
core Kubernetes concepts: Deployments, Services, ConfigMaps, Secrets, Namespaces,
and service discovery — all managed declaratively with YAML manifests.

## 🏗️ Architecture

```
                        Browser (User)
                             │
                             ▼
              ┌──────────────────────────────┐
              │   FRONTEND TIER (nginx)        │
              │   Deployment: 2 replicas (HA)  │
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
backend database is intentionally kept private using a **ClusterIP** service —
accessible only from inside the cluster. The frontend locates the backend by its
**service DNS name** rather than a hard-coded IP, demonstrating Kubernetes
service discovery.

## 🧰 Tech Stack

- **Kubernetes** (minikube for local development)
- **nginx** — frontend web server
- **Redis 7** — backend datastore
- **kubectl** — cluster management
- Declarative **YAML** manifests (Infrastructure as Code)

## 📁 Project Structure

```
k8s-2tier-app/
└── manifests/
    ├── namespace.yaml            # Isolated namespace for the project
    ├── backend-secret.yaml       # Redis password (Secret)
    ├── backend-deployment.yaml   # Redis Deployment
    ├── backend-service.yaml      # Redis Service (ClusterIP - private)
    ├── frontend-configmap.yaml   # Frontend configuration
    ├── frontend-deployment.yaml  # nginx Deployment (2 replicas)
    └── frontend-service.yaml     # nginx Service (NodePort - public)
```

## 🚀 How to Run

**Prerequisites:** Docker, minikube, and kubectl installed.

```bash
# 1. Start the local cluster
minikube start --driver=docker

# 2. Apply all manifests (order matters: namespace first)
kubectl apply -f manifests/namespace.yaml
kubectl apply -f manifests/backend-secret.yaml
kubectl apply -f manifests/backend-deployment.yaml
kubectl apply -f manifests/backend-service.yaml
kubectl apply -f manifests/frontend-configmap.yaml
kubectl apply -f manifests/frontend-deployment.yaml
kubectl apply -f manifests/frontend-service.yaml

# (or apply the whole folder at once)
kubectl apply -f manifests/

# 3. Verify everything is running
kubectl get all -n twotier-app

# 4. Open the frontend in your browser
minikube service frontend-service -n twotier-app
```

## ✅ Verifying Service Discovery

Confirm that the frontend can reach the backend by its service name:

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
| **Secret** | Stores the database password securely |
| **Service Discovery** | Frontend finds backend via DNS name |
| **High Availability** | Frontend runs 2 replicas |

## 🧹 Cleanup

```bash
# Delete everything in the namespace
kubectl delete namespace twotier-app

# Stop the cluster
minikube stop
```

## 📌 Notes

- `stringData` is used in the Secret so the password can be written in plain text
  and Kubernetes handles base64 encoding automatically.
- Note that base64 is **encoding, not encryption**. In a production environment,
  secrets would be further protected using tools such as sealed-secrets or an
  external secrets manager (e.g. AWS Secrets Manager).

---

*Built as a hands-on Kubernetes learning project — every manifest written and
tested manually to understand how multi-tier applications run on Kubernetes.*
