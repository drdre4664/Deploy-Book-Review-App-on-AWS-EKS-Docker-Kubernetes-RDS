# Deploy the Book Review App — 3-Tier Architecture (Docker, Kubernetes, AWS EKS, RDS)

A full-stack book review application deployed as a 3-tier architecture on AWS using EKS (Kubernetes), RDS MySQL, and Docker Hub.

---

## Architecture
            Internet
               │
               ▼
        [AWS LoadBalancer]
               │
               ▼
     Frontend Pod (Next.js)        <-- EKS
               │
     (HTTP to public LoadBalancer)
               │
               ▼
        [AWS LoadBalancer]
               │
               ▼
     Backend Pod (Node.js)         <-- EKS
               │
               ▼
          AWS RDS MySQL

| Tier     | Technology         | Hosting          |
| -------- | ------------------ | ---------------- |
| Frontend | Next.js (React)    | EKS (Kubernetes) |
| Backend  | Node.js / Express  | EKS (Kubernetes) |
| Database | MySQL              | AWS RDS          |

## Features

- Browse books
- User registration and login (JWT auth)
- Submit and view book reviews

## Tech Stack

- **Frontend:** Next.js, React, Tailwind CSS
- **Backend:** Node.js, Express, Sequelize ORM
- **Database:** MySQL (AWS RDS)
- **Containerization:** Docker (multi-stage builds, linux/amd64)
- **Orchestration:** Kubernetes (AWS EKS via eksctl)
- **Image Registry:** Docker Hub

## Project Structure
.
├── backend/
│   ├── Dockerfile               # Multi-stage production build
│   ├── .dockerignore
│   └── src/
│       ├── server.js
│       ├── config/db.js
│       ├── models/
│       ├── controllers/
│       └── routes/
│
├── frontend/
│   ├── Dockerfile               # Multi-stage production build
│   ├── .dockerignore
│   └── src/app/
│       ├── page.js              # Books listing
│       ├── login/
│       ├── register/
│       └── book/[id]/           # Book detail + reviews
│
├── k8s/
│   ├── backend/
│   │   ├── configmap.yaml       # DB config, port, CORS origins
│   │   ├── secret.yaml.example  # Template — copy to secret.yaml and fill values
│   │   ├── deployment.yaml      # 2 replicas, health probes
│   │   └── service.yaml         # LoadBalancer (public)
│   └── frontend/
│       ├── configmap.yaml       # NEXT_PUBLIC_API_URL
│       ├── deployment.yaml      # 2 replicas, health probes
│       └── service.yaml         # LoadBalancer (public, port 80)
│
└── docker-compose.yml           # Local development

## Prerequisites

- AWS CLI configured
- eksctl installed
- kubectl installed
- Docker with buildx support
- Docker Hub account

---

## Deployment Steps

### 1. Create AWS RDS MySQL

- AWS Console → RDS → Create database → MySQL → Free tier
- Set a strong master password
- Connectivity: set **Public access = Yes**
- Add inbound rule to RDS security group: MySQL 3306 from 0.0.0.0/0
- Note the **endpoint** after creation

### 2. Create EKS Cluster

> If on a corporate network, use a personal hotspot — some company networks block AWS endpoints.

```bash
eksctl create cluster \
  --name book-review-cluster \
  --region us-east-1 \
  --nodegroup-name book-review-nodes \
  --node-type t3.micro \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

Verify nodes are ready:

```bash
kubectl get nodes
```

---

## Container Registry Flow (Docker Hub)

EKS worker nodes cannot pull images from your laptop — they need a public registry. This project uses **Docker Hub** as the image registry.

### Why a registry is required
Your laptop                Docker Hub                 EKS Worker Nodes
+-----------+   push     +--------------+   pull     +----------------+
| docker    | ---------> | dre4664/     | <--------- | kubelet on     |
| buildx    |            | book-review- |            | each EC2 node  |
| build     |            | backend:v1   |            | (image: tag)   |
+-----------+            +--------------+            +----------------+

When Kubernetes schedules a Pod onto a node, the kubelet on that node reads the `image:` field in `deployment.yaml` and pulls the image from the referenced registry. The image must be reachable from the node.

### Authentication

Before pushing, authenticate Docker CLI with Docker Hub:

```bash
docker login
# Username: YOUR_DOCKERHUB_USERNAME
# Password: <Docker Hub access token, not your account password>
```

### Tagging strategy — why `:v1` instead of `:latest`

This project uses **immutable tags** (`:v1`, `:v2`) rather than `:latest`. Here's why:

| Tag Strategy        | Behavior                                                    | Production Risk |
| ------------------- | ----------------------------------------------------------- | --------------- |
| `:latest`           | Mutable — same tag points to different images over time     | Kubernetes caches images. If you push a new `:latest` but the node already has an older `:latest` cached, new Pods may run **stale code** with no error |
| `:v1`, `:v2`, SHA   | Immutable — each tag points to exactly one image forever    | Forces explicit version bumps in `deployment.yaml`. Rollbacks are trivial — just change the tag |

A new deployment requires bumping the tag in `deployment.yaml` and re-applying — this is the correct, auditable pattern.

### How Kubernetes finds the image

In `k8s/backend/deployment.yaml`:

```yaml
spec:
  containers:
    - name: backend
      image: dre4664/book-review-backend:v1
      #      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      #      registry/repository:tag
```

When `kubectl apply` runs, the EKS node pulls `dre4664/book-review-backend:v1` from Docker Hub and starts the container.

---

### 3. Build and push the **backend** image

> **Why backend first?** The frontend bakes the backend's public LoadBalancer URL into its build at compile time (`NEXT_PUBLIC_API_URL`). That URL doesn't exist until the backend service is deployed. So the order is: backend image → deploy backend → get URL → build frontend with the URL → deploy frontend.

> Mac users (Apple Silicon): EKS runs `linux/amd64` — you must cross-compile.

```bash
cd backend
docker buildx build \
  --platform linux/amd64 \
  -t YOUR_DOCKERHUB_USERNAME/book-review-backend:v1 \
  --push .
```

### 4. Deploy the backend to Kubernetes

```bash
# Create your secret file from the example template
cp k8s/backend/secret.yaml.example k8s/backend/secret.yaml
# Edit secret.yaml — fill in your real DB_PASS and JWT_SECRET (RDS endpoint from Step 1)

kubectl apply -f k8s/backend/secret.yaml
kubectl apply -f k8s/backend/configmap.yaml
kubectl apply -f k8s/backend/deployment.yaml
kubectl apply -f k8s/backend/service.yaml
```

Wait for the LoadBalancer to be provisioned, then capture the public URL:

```bash
kubectl get svc backend-service
# Copy the EXTERNAL-IP — this is the backend public URL
```

### 5. Build and push the **frontend** image (with backend URL baked in)

Now that the backend URL exists, set it in the frontend build environment:

```bash
cd ../frontend
echo "NEXT_PUBLIC_API_URL=http://BACKEND_EXTERNAL_IP:3001" > .env.production

docker buildx build \
  --platform linux/amd64 \
  -t YOUR_DOCKERHUB_USERNAME/book-review-frontend:v2 \
  --push .
```

Also update `k8s/frontend/configmap.yaml` so `NEXT_PUBLIC_API_URL` matches the backend URL.

### 6. Deploy the frontend to Kubernetes

```bash
kubectl apply -f k8s/frontend/configmap.yaml
kubectl apply -f k8s/frontend/deployment.yaml
kubectl apply -f k8s/frontend/service.yaml

kubectl get svc frontend-service
# Copy the EXTERNAL-IP — this is the frontend public URL
```

### 7. Fix CORS — update backend with the frontend URL

The backend's `ALLOWED_ORIGINS` must include the frontend's public URL, or the browser will block requests:

```bash
# Edit k8s/backend/configmap.yaml — add the frontend EXTERNAL-IP to ALLOWED_ORIGINS
kubectl apply -f k8s/backend/configmap.yaml
kubectl rollout restart deployment backend
```

### 8. Access the App

Open the frontend EXTERNAL-IP in your browser. You should see the book review app live.

---

## Key Lessons Learned

| Problem | Root Cause | Fix |
|---|---|---|
| `ImagePullBackOff` | Built on Mac ARM64, EKS needs AMD64 | `docker buildx build --platform linux/amd64` |
| Books not loading | `NEXT_PUBLIC_API_URL` was internal cluster DNS — doesn't work in browser | Rebuild frontend image with public backend LoadBalancer URL |
| CORS blocked | Backend `ALLOWED_ORIGINS` missing frontend's public URL | Update ConfigMap with frontend LoadBalancer URL, then `kubectl rollout restart` |
| Pods stuck `Pending` | `t3.micro` nodes max ~4 pods (system pods take slots) | Set `replicas: 1` or use larger node type |
| `NodeCreationFailure` | Nodes from AWS Console can't join — missing `aws-auth` ConfigMap | Use `eksctl create nodegroup` instead |
| RDS unreachable from EKS | RDS in default VPC, EKS in its own VPC | Make RDS publicly accessible |

## Docker Images

| Image                          | Tag | Description                                  |
| ------------------------------ | --- | -------------------------------------------- |
| `dre4664/book-review-backend`  | v1  | Node.js Express API (linux/amd64)            |
| `dre4664/book-review-frontend` | v2  | Next.js with backend URL baked in (linux/amd64) |
