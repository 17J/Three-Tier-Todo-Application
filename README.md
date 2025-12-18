# Three-Tier Todo Application (Kubernetes Implementation)

A production-ready full-stack Todo application demonstrating **Three-Tier Architecture** deployed on **Kubernetes**.

## 🏗️ Architecture

```
┌─────────────────┐
│   Frontend      │  React.js (Port 3000)
│  (Nginx/React)  │
└────────┬────────┘
         │
┌────────▼────────┐
│    Backend      │  Spring Boot (Port 8080)
│  (Spring Boot)  │  REST API
└────────┬────────┘
         │
┌────────▼────────┐
│    Database     │  MySQL (Port 3306)
│     (MySQL)     │  Persistent Storage
└─────────────────┘
```

**Three Distinct Layers:**

1. **Presentation Tier:** React.js frontend with responsive UI
2. **Application Tier:** Java Spring Boot REST API
3. **Data Tier:** MySQL database with persistent volume

---

## 🛠️ Tech Stack

- **Frontend:** React.js, Nginx
- **Backend:** Java, Spring Boot 3.x
- **Database:** MySQL 8.0
- **Containerization:** Docker
- **Orchestration:** Kubernetes
- **Networking:** Kubernetes Services, Ingress

---

## ✨ Features

- ✅ Create, read, update, delete todos (CRUD operations)
- ✏️ Edit existing todos with real-time updates
- 🗑️ Soft delete with confirmation
- 🔍 Filter by status (All/Completed/Pending)
- 📱 Fully responsive design (mobile-first)
- 🔐 User authentication with JWT tokens
- 💾 Persistent storage with MySQL
- 🔄 Auto-scaling with Kubernetes HPA
- 📊 Health check endpoints for monitoring

---

## 📋 Prerequisites

Before deploying, ensure you have:

- **Kubernetes cluster** (v1.20+)
  - Minikube, Kind, or cloud provider (GKE, EKS, AKS)
- **kubectl** installed and configured (v1.20+)
- **Docker** installed (for building custom images)
- **Ingress Controller** (nginx-ingress recommended)
- **Cluster Resources:**
  - Minimum 4GB RAM
  - 2 CPU cores
  - 10GB storage for persistent volumes
- **StorageClass** configured for dynamic provisioning

### Verify Prerequisites

```bash
# Check Kubernetes version
kubectl version --short

# Check cluster nodes
kubectl get nodes

# Verify StorageClass
kubectl get storageclass

# Check Ingress Controller
kubectl get pods -n ingress-nginx
```

---

## 🚀 Quick Start

### Step 1: Clone the Repository

```bash
git clone https://github.com/17J/Three-Tier-Todo-Application.git
cd Three-Tier-Todo-Application
```

### Step 2: Create Namespace

```bash
kubectl create namespace todo-app
```

### Step 3: Configure Secrets & ConfigMaps

```bash
kubectl apply -f k8s/secrets-configmap.yml -n todo-app
```

### Step 4: Deploy Database Layer

```bash
# Deploy MySQL with persistent storage
kubectl apply -f k8s/db-ds-service.yml -n todo-app

# Wait for database to be ready (may take 1-2 minutes)
kubectl wait --for=condition=ready pod -l app=mysql -n todo-app --timeout=300s

# Verify database is running
kubectl get pods -l app=mysql -n todo-app
```

### Step 5: Initialize Database (Optional)

```bash
# If you need to run initial migrations
kubectl exec -it <mysql-pod-name> -n todo-app -- mysql -u root -p
# Then run your schema.sql or migrations
```

### Step 6: Deploy Backend Layer

```bash
# Deploy Spring Boot API
kubectl apply -f k8s/backend-ds-service.yml -n todo-app

# Wait for backend to be ready
kubectl wait --for=condition=ready pod -l app=backend -n todo-app --timeout=300s

# Check backend logs
kubectl logs -l app=backend -n todo-app --tail=50
```

### Step 7: Deploy Frontend Layer

```bash
# Deploy React application
kubectl apply -f k8s/frontend-ds-service.yml -n todo-app

# Wait for frontend to be ready
kubectl wait --for=condition=ready pod -l app=frontend -n todo-app --timeout=300s
```

### Step 8: Configure Ingress

```bash
# Deploy Ingress for external access
kubectl apply -f k8s/ingress.yml -n todo-app

# Get Ingress address
kubectl get ingress -n todo-app
```

### Step 9: Verify Deployment

```bash
# Check all resources
kubectl get all -n todo-app

# Check pod status and events
kubectl get pods -n todo-app -o wide
kubectl describe pods -n todo-app

# Check persistent volumes
kubectl get pvc -n todo-app

# View logs for debugging
kubectl logs -l app=backend -n todo-app
kubectl logs -l app=frontend -n todo-app
```

---

## 🌐 Accessing the Application

### Option 1: Using NodePort (Development)

```bash
# Get NodePort
kubectl get svc frontend-service -n todo-app

# Access at: http://<node-ip>:<node-port>
# For Minikube: minikube service frontend-service -n todo-app
```

### Option 2: Using LoadBalancer (Cloud)

```bash
kubectl get svc frontend-service -n todo-app
# Access at: http://<EXTERNAL-IP>
```

### Option 3: Using Ingress (Production)

```bash
# Get Ingress hostname
kubectl get ingress -n todo-app

# Access at: http://todo-app.example.com
# (Configure DNS to point to Ingress controller IP)
```

### Option 4: Port Forward (Local Testing)

```bash
# Frontend
kubectl port-forward svc/frontend-service 3000:80 -n todo-app
# Access at: http://localhost:3000

# Backend (for direct API testing)
kubectl port-forward svc/backend-service 8080:8080 -n todo-app
# Access at: http://localhost:8080
```

---

## 🔧 Configuration

### Environment Variables

**Backend requires:**

- `SPRING_DATASOURCE_URL` - MySQL service hostname
- `SPRING_DATASOURCE_USERNAME` - Database user
- `SPRING_DATASOURCE_PASSWORD` - Database password

**Frontend requires:**

- `API_BASE_URL` - Backend API endpoint

## 🔍 Health Checks

**Backend health endpoint:**

```bash
curl http://backend-service:8080/actuator/health
```

**Frontend health endpoint:**

```bash
curl http://frontend-service/health
```

---

## 📊 Monitoring

```bash
# Watch pod status
kubectl get pods -n todo-app -w

# View resource usage
kubectl top pods -n todo-app
kubectl top nodes

# Check service endpoints
kubectl get endpoints -n todo-app
```

---

## 📈 Scaling

### Manual Scaling

```bash
# Scale backend replicas
kubectl scale deployment backend --replicas=3 -n todo-app

# Scale frontend replicas
kubectl scale deployment frontend --replicas=2 -n todo-app
```

### Auto-Scaling (HPA)

```bash
# Enable metrics server (if not already installed)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Create HPA
kubectl autoscale deployment backend --cpu-percent=70 --min=2 --max=10 -n todo-app
```

---

## 🧹 Cleanup

### Delete all resources

```bash
# Delete entire namespace (recommended)
kubectl delete namespace todo-app
```

### Delete individual resources

```bash
# Delete deployments and services
kubectl delete -f k8s/ -n todo-app

# Delete persistent volume claims (warning: deletes data)
kubectl delete pvc -n todo-app --all
```

---


### Snapshots

Some Screenshots of this project are given below
<br />

<p align="center">
    <img src="snapshot/taskmanager.jpg"/>
    <img src="snapshot/taskmanager1.jpg"/>
    <img src="snapshot/taskmanager2.jpg"/>
    <img src="snapshot/taskmanager3.jpg"/>
</p>




## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request





---
