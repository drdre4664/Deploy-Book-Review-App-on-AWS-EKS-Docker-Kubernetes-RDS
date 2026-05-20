# Deploy the Book Review App — 3-Tier Architecture (Docker, Kubernetes, AWS EKS, RDS)

A full-stack book review application deployed as a 3-tier architecture on AWS using EKS (Kubernetes), RDS MySQL, and Docker Hub.

## Architecture

```
Internet
   │
   ▼
[AWS LoadBalancer] ──► Frontend Pod (Next.js)  ← EKS
                              │
                    (HTTP to public LoadBalancer)
                              ▼
                    [AWS LoadBalancer] ──► Backend Pod (Node.js)  ← EKS
                                                   │
                                                   ▼
                                          AWS RDS MySQL
```

| Tier     | Technology      | Hosting          |
|----------|----------------|------------------|
| Frontend | Next.js (React) | EKS (Kubernetes) |
| Backend  | Node.js/Express | EKS (Kubernetes) |
| Database | MySQL           | AWS RDS          |

## Features
- Browse books
- User registration and login (JWT auth)
- Submit and view book reviews

## Tech Stack
- **Frontend**: Next.js, React, Tailwind CSS
- **Backend**: Node.js, Express, Sequelize ORM
- **Database**: MySQL (AWS RDS)
- **Containerization**: Docker (multi-stage builds, linux/amd64)
- **Orchestration**: Kubernetes (AWS EKS via eksctl)
- **Image Registry**: Docker Hub

---

## Project Structure

```
├── backend/
│   ├── Dockerfile              # Multi-stage production build
│   ├── .dockerignore
│   └── src/
│       ├── server.js
│       ├── config/db.js
│       ├── models/
│       ├── controllers/
│       └── routes/
├── frontend/
│   ├── Dockerfile              # Multi-stage production build
│   ├── .dockerignore
│   └── src/app/
│       ├── page.js             # Books listing
│       ├── login/
│       ├── register/
│       └── book/[id]/          # Book detail + reviews
├── k8s/
│   ├── backend/
│   │   ├── configmap.yaml      # DB config, port, CORS origins
│   │   ├── secret.yaml.example # Template — copy to secret.yaml and fill values
│   │   ├── deployment.yaml     # 2 replicas, health probes
│   │   └── service.yaml        # LoadBalancer (public)
│   └── frontend/
│       ├── configmap.yaml      # NEXT_PUBLIC_API_URL
│       ├── deployment.yaml     # 2 replicas, health probes
│       └── service.yaml        # LoadBalancer (public, port 80)
└── docker-compose.yml          # Local development
```

---

## Prerequisites
- AWS CLI configured
- eksctl installed
- kubectl installed
- Docker with buildx support
- Docker Hub account

---

## Deployment Steps

### 1. Create AWS RDS MySQL
1. AWS Console → RDS → Create database → MySQL → Free tier
2. Set a strong master password
3. Connectivity: set **Public access = Yes**
4. Add inbound rule to RDS security group: MySQL 3306 from 0.0.0.0/0
5. Note the endpoint after creation

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

### 3. Build and Push Docker Images

> Mac users (Apple Silicon): EKS runs linux/amd64 — you must cross-compile.

```bash
# Backend
cd backend
docker buildx build --platform linux/amd64 -t YOUR_DOCKERHUB_USERNAME/book-review-backend:v1 --push .

# Frontend — NEXT_PUBLIC_API_URL is baked in at build time, must be set before building
echo "NEXT_PUBLIC_API_URL=http://BACKEND_LOADBALANCER_URL:3001" > .env.production
cd ../frontend
docker buildx build --platform linux/amd64 -t YOUR_DOCKERHUB_USERNAME/book-review-frontend:v2 --push .
```

### 4. Deploy to Kubernetes

```bash
# Create your secret file from the example template
cp k8s/backend/secret.yaml.example k8s/backend/secret.yaml
# Edit secret.yaml — fill in your real DB_PASS and JWT_SECRET

# Deploy backend
kubectl apply -f k8s/backend/secret.yaml
kubectl apply -f k8s/backend/configmap.yaml
kubectl apply -f k8s/backend/deployment.yaml
kubectl apply -f k8s/backend/service.yaml

# Get the backend public URL (needed for frontend configmap and image build)
kubectl get svc backend-service

# Update k8s/frontend/configmap.yaml with backend EXTERNAL-IP, then rebuild frontend image
# Deploy frontend
kubectl apply -f k8s/frontend/configmap.yaml
kubectl apply -f k8s/frontend/deployment.yaml
kubectl apply -f k8s/frontend/service.yaml
```

### 5. Access the App
```bash
kubectl get svc frontend-service
# Open EXTERNAL-IP in your browser
```

---

## Key Lessons Learned

| Problem | Root Cause | Fix |
|---------|-----------|-----|
| ImagePullBackOff | Built on Mac ARM64, EKS needs AMD64 | docker buildx build --platform linux/amd64 |
| Books not loading | NEXT_PUBLIC_API_URL was internal cluster DNS — doesn't work in browser | Rebuild frontend image with public backend LoadBalancer URL |
| CORS blocked | Backend ALLOWED_ORIGINS missing frontend's public URL | Update ConfigMap with frontend LoadBalancer URL |
| Pods stuck Pending | t3.micro nodes max ~4 pods (system pods take slots) | Set replicas: 1 or use larger node type |
| NodeCreationFailure | Nodes from AWS Console can't join — missing aws-auth ConfigMap | Use eksctl create nodegroup instead |
| RDS unreachable from EKS | RDS in default VPC, EKS in its own VPC | Make RDS publicly accessible |

---

## Docker Images

| Image | Tag | Description |
|-------|-----|-------------|
| dre4664/book-review-backend | v1 | Node.js Express API (linux/amd64) |
| dre4664/book-review-frontend | v2 | Next.js with backend URL baked in (linux/amd64) |
# Deploy the Book Review App — 3-Tier Architecture (Docker, Kubernetes, AWS EKS, RDS)

A full-stack book review application deployed as a 3-tier architecture on AWS using EKS (Kubernetes), RDS MySQL, and Docker Hub.

## Architecture

```
Internet
   │
   ▼
[AWS LoadBalancer] ──► Frontend Pod (Next.js)  ← EKS
                              │
                    (HTTP to public LoadBalancer)
                              ▼
                    [AWS LoadBalancer] ──► Backend Pod (Node.js)  ← EKS
                                                   │
                                                   ▼
                                          AWS RDS MySQL
```

| Tier     | Technology      | Hosting          |
|----------|----------------|------------------|
| Frontend | Next.js (React) | EKS (Kubernetes) |
| Backend  | Node.js/Express | EKS (Kubernetes) |
| Database | MySQL           | AWS RDS          |

## Features
- Browse books
- User registration and login (JWT auth)
- Submit and view book reviews

## Tech Stack
- **Frontend**: Next.js, React, Tailwind CSS
- **Backend**: Node.js, Express, Sequelize ORM
- **Database**: MySQL (AWS RDS)
- **Containerization**: Docker (multi-stage builds, linux/amd64)
- **Orchestration**: Kubernetes (AWS EKS via eksctl)
- **Image Registry**: Docker Hub

---

## Project Structure

```
├── backend/
│   ├── Dockerfile              # Multi-stage production build
│   ├── .dockerignore
│   └── src/
│       ├── server.js
│       ├── config/db.js
│       ├── models/
│       ├── controllers/
│       └── routes/
├── frontend/
│   ├── Dockerfile              # Multi-stage production build
│   ├── .dockerignore
│   └── src/app/
│       ├── page.js             # Books listing
│       ├── login/
│       ├── register/
│       └── book/[id]/          # Book detail + reviews
├── k8s/
│   ├── backend/
│   │   ├── configmap.yaml      # DB config, port, CORS origins
│   │   ├── secret.yaml.example # Template — copy to secret.yaml and fill values
│   │   ├── deployment.yaml     # 2 replicas, health probes
│   │   └── service.yaml        # LoadBalancer (public)
│   └── frontend/
│       ├── configmap.yaml      # NEXT_PUBLIC_API_URL
│       ├── deployment.yaml     # 2 replicas, health probes
│       └── service.yaml        # LoadBalancer (public, port 80)
└── docker-compose.yml          # Local development
```

---

## Prerequisites
- AWS CLI configured
- eksctl installed
- kubectl installed
- Docker with buildx support
- Docker Hub account

---

## Deployment Steps

### 1. Create AWS RDS MySQL
1. AWS Console → RDS → Create database → MySQL → Free tier
2. Set a strong master password
3. Connectivity: set **Public access = Yes**
4. Add inbound rule to RDS security group: MySQL 3306 from 0.0.0.0/0
5. Note the endpoint after creation

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

### 3. Build and Push Docker Images

> Mac users (Apple Silicon): EKS runs linux/amd64 — you must cross-compile.

```bash
# Backend
cd backend
docker buildx build --platform linux/amd64 -t YOUR_DOCKERHUB_USERNAME/book-review-backend:v1 --push .

# Frontend — NEXT_PUBLIC_API_URL is baked in at build time, must be set before building
echo "NEXT_PUBLIC_API_URL=http://BACKEND_LOADBALANCER_URL:3001" > .env.production
cd ../frontend
docker buildx build --platform linux/amd64 -t YOUR_DOCKERHUB_USERNAME/book-review-frontend:v2 --push .
```

### 4. Deploy to Kubernetes

```bash
# Create your secret file from the example template
cp k8s/backend/secret.yaml.example k8s/backend/secret.yaml
# Edit secret.yaml — fill in your real DB_PASS and JWT_SECRET

# Deploy backend
kubectl apply -f k8s/backend/secret.yaml
kubectl apply -f k8s/backend/configmap.yaml
kubectl apply -f k8s/backend/deployment.yaml
kubectl apply -f k8s/backend/service.yaml

# Get the backend public URL (needed for frontend configmap and image build)
kubectl get svc backend-service

# Update k8s/frontend/configmap.yaml with backend EXTERNAL-IP, then rebuild frontend image
# Deploy frontend
kubectl apply -f k8s/frontend/configmap.yaml
kubectl apply -f k8s/frontend/deployment.yaml
kubectl apply -f k8s/frontend/service.yaml
```

### 5. Access the App
```bash
kubectl get svc frontend-service
# Open EXTERNAL-IP in your browser
```

---

## Key Lessons Learned

| Problem | Root Cause | Fix |
|---------|-----------|-----|
| ImagePullBackOff | Built on Mac ARM64, EKS needs AMD64 | docker buildx build --platform linux/amd64 |
| Books not loading | NEXT_PUBLIC_API_URL was internal cluster DNS — doesn't work in browser | Rebuild frontend image with public backend LoadBalancer URL |
| CORS blocked | Backend ALLOWED_ORIGINS missing frontend's public URL | Update ConfigMap with frontend LoadBalancer URL |
| Pods stuck Pending | t3.micro nodes max ~4 pods (system pods take slots) | Set replicas: 1 or use larger node type |
| NodeCreationFailure | Nodes from AWS Console can't join — missing aws-auth ConfigMap | Use eksctl create nodegroup instead |
| RDS unreachable from EKS | RDS in default VPC, EKS in its own VPC | Make RDS publicly accessible |

---

## Docker Images

| Image | Tag | Description |
|-------|-----|-------------|
| dre4664/book-review-backend | v1 | Node.js Express API (linux/amd64) |
| dre4664/book-review-frontend | v2 | Next.js with backend U
