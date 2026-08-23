
# Online Shop GitOps

Kubernetes manifests and GitOps configuration for the Online Shop application.

This repository contains the Kubernetes resources used to run the Online Shop locally on a Kind cluster and is intended to evolve into the GitOps source of truth for CI/CD and Argo CD.

## Current Stack

- Frontend: React + Vite
- Backend: Node.js + TypeScript + Express
- Database: PostgreSQL 17
- ORM / migrations: Prisma 7
- Container runtime: Docker
- Kubernetes: Kind
- Kubernetes configuration: Kustomize
- GitOps target: Argo CD (planned)

## Repository Structure

```text
online-shop-gitops/
├── apps/
│   └── online-shop/
│       └── base/
│           ├── backend-configmap.yaml
│           ├── backend-deployment.yaml
│           ├── backend-migration-job.yaml
│           ├── backend-secret.example.yaml
│           ├── backend-service.yaml
│           ├── frontend-configmap.yaml
│           ├── frontend-deployment.yaml
│           ├── frontend-service.yaml
│           ├── namespace.yaml
│           ├── postgres-secret.example.yaml
│           ├── postgres-service.yaml
│           ├── postgres-statefulset.yaml
│           └── kustomization.yaml
└── README.md
```

Kustomize keeps the Kubernetes resources as plain YAML and composes them through `kustomization.yaml`. The manifests can be rendered with `kubectl kustomize` and applied with `kubectl apply -k`. citeturn0search0

## Kubernetes Architecture

```text
                         Kind Cluster
                              │
                    ┌─────────┴─────────┐
                    │                   │
                    ▼                   ▼
             Frontend Service     Backend Service
                :5173                 :5050
                    │                   │
                    ▼                   ▼
             Frontend Pod         Backend Pod
                    │                   │
                    │ Vite proxy        │ Prisma
                    │                   │
                    └──────────────┐    ▼
                                   │  PostgreSQL
                                   └── Service :5432
                                         │
                                         ▼
                                  postgres-0
                                         │
                                         ▼
                                  PersistentVolume
                                      5 GiB
```

### Components

- **Frontend Deployment** runs the Vite application on port `5173`.
- **Frontend Service** exposes the frontend through a local `NodePort` (`30173`) for the Kind setup.
- **Frontend ConfigMap** sets `VITE_API_PROXY_TARGET` to `http://online-shop-backend:5050`.
- **Backend Deployment** runs the Node.js API on port `5050`.
- **Backend Service** provides stable in-cluster access to the API.
- **PostgreSQL StatefulSet** runs PostgreSQL 17 with persistent storage.
- **PostgreSQL Service** provides stable DNS access as `postgres` on port `5432`.
- **Prisma Migration Job** applies the database migrations before application use.
- **PersistentVolumeClaim** requests `5Gi` using the Kind default `standard` StorageClass.

## Secrets

Real secrets are intentionally not stored in Git.

Example documentation files are provided instead:

- `apps/online-shop/base/backend-secret.example.yaml`
- `apps/online-shop/base/postgres-secret.example.yaml`

For local development, create the Kubernetes Secrets manually.

### Backend Secret

```bash
kubectl create secret generic online-shop-backend-secret \\
  -n online-shop \\
  --from-literal=DATABASE_URL='postgresql://shopuser:shoppassword@postgres:5432/onlineshop?schema=public' \\
  --from-literal=JWT_ACCESS_SECRET='replace-with-local-secret' \\
  --from-literal=JWT_REFRESH_SECRET='replace-with-local-secret'
```

### PostgreSQL Secret

```bash
kubectl create secret generic online-shop-postgres-secret \\
  -n online-shop \\
  --from-literal=POSTGRES_DB='onlineshop' \\
  --from-literal=POSTGRES_USER='shopuser' \\
  --from-literal=POSTGRES_PASSWORD='shoppassword'
```

> Do not commit real secret values to this repository. The example files are documentation/templates only.

## Prerequisites

- Docker
- Kind
- `kubectl`
- Git

Check the active Kubernetes context:

```bash
kubectl config current-context
```

For this project the local Kind context is:

```text
kind-online-shop
```

## Create / Start the Local Cluster

If the Kind cluster already exists, start Docker and verify it:

```bash
kubectl config current-context
kubectl get nodes
```

Expected node state:

```text
online-shop-control-plane   Ready   control-plane
```

## Deploy the Application

From the repository root:

```bash
kubectl apply -k apps/online-shop/base
```

Kustomize can be rendered without applying anything using:

```bash
kubectl kustomize apps/online-shop/base
```

You can also preview changes with:

```bash
kubectl diff -k apps/online-shop/base
```

These are standard Kustomize workflows supported by `kubectl`. citeturn0search0turn0search3

## Verify the Deployment

Check all application Pods:

```bash
kubectl get pods -n online-shop
```

Expected healthy state:

```text
online-shop-backend-...            1/1   Running
online-shop-frontend-...           1/1   Running
postgres-0                         1/1   Running
online-shop-backend-migration-...  0/1   Completed
```

Check Services:

```bash
kubectl get svc -n online-shop
```

Check persistent storage:

```bash
kubectl get pvc -n online-shop
kubectl get pv
kubectl get storageclass
```

## Database Migration

The backend image contains the Prisma CLI and migration files.

The Kubernetes migration Job runs:

```bash
npm run db:migrate
```

Check the Job:

```bash
kubectl get jobs -n online-shop
```

Expected:

```text
online-shop-backend-migration   Complete   1/1
```

Verify the PostgreSQL schema:

```bash
kubectl exec -it postgres-0 -n online-shop -- \\
  psql -U shopuser -d onlineshop -c '\dt'
```

## Test Backend

Forward the backend Service to the local machine:

```bash
kubectl port-forward svc/online-shop-backend 5050:5050 -n online-shop
```

In another terminal:

```bash
curl http://localhost:5050/health
```

Expected:

```json
{"message":"Online Shop API is running"}
```

Test a database-backed endpoint:

```bash
curl http://localhost:5050/api/categories
```

An empty database may correctly return:

```json
[]
```

This confirms the request reaches the backend and the backend can query PostgreSQL through the Kubernetes Service.

## Test Frontend

Forward the frontend Service:

```bash
kubectl port-forward svc/online-shop-frontend 5173:5173 -n online-shop
```

Open:

```text
http://localhost:5173
```

The frontend Vite proxy sends `/api` and `/uploads` requests to:

```text
http://online-shop-backend:5050
```

The Kubernetes Service name is used for in-cluster communication rather than `localhost`.

## Useful Troubleshooting Commands

### Pod status

```bash
kubectl get pods -n online-shop
```

### Backend logs

```bash
kubectl logs deployment/online-shop-backend -n online-shop
```

### Frontend logs

```bash
kubectl logs deployment/online-shop-frontend -n online-shop
```

### PostgreSQL logs

```bash
kubectl logs postgres-0 -n online-shop
```

### Describe a failing Pod

```bash
kubectl describe pod <pod-name> -n online-shop
```

### Check backend environment variables

```bash
kubectl exec -it deployment/online-shop-backend -n online-shop -- sh
```

Inside the container:

```bash
echo $DATABASE_URL
```

### Test PostgreSQL DNS from the backend Pod

```bash
kubectl exec -it deployment/online-shop-backend -n online-shop -- sh
```

Inside the container:

```bash
getent hosts postgres
```

## GitOps Workflow

The intended workflow is:

```text
Application Repository
        │
        │ build / test
        ▼
   Docker Image
        │
        │ push
        ▼
    Container Registry
        │
        ▼
   GitOps Repository
        │
        │ desired Kubernetes state
        ▼
      Argo CD
        │
        ▼
   Kubernetes Cluster
```

The current repository provides the Kubernetes base manifests. CI/CD and Argo CD integration will be added as the project progresses.

## Current Implementation Status

- [x] Kind Kubernetes cluster
- [x] `online-shop` namespace
- [x] Backend Deployment
- [x] Backend Service
- [x] Backend Kubernetes Secret documentation
- [x] PostgreSQL 17 StatefulSet
- [x] PostgreSQL Service
- [x] PostgreSQL Secret documentation
- [x] PersistentVolumeClaim
- [x] Prisma migration Job
- [x] Frontend Deployment
- [x] Frontend Service
- [x] Frontend ConfigMap
- [x] Frontend → Backend Kubernetes networking
- [x] Backend → PostgreSQL Kubernetes networking
- [x] Database migration verified
- [x] Frontend portal verified locally
- [ ] CI/CD pipeline integration
- [ ] Automated image version updates
- [ ] Argo CD integration
- [ ] Kubernetes Ingress
- [ ] Development / staging / production overlays

## Notes

This repository is currently focused on a local Kubernetes learning environment. The manifests will be hardened as the project moves toward a production-style GitOps workflow, including proper secret management, image promotion, overlays, ingress, and Argo CD reconciliation.
