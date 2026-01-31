# YOLO - E-commerce Application Deployment Via Kubernetes (GKE)

A full-stack e-commerce application built with Node.js, Express, MongoDB, and React. 
It Demonstrates modern web development practices with containerized deployment using Docker and Kubernetes

- *Frontend*: React application served by Nginx
- *Backend*: Node.js API server
- *Database*: MongoDB with persistent storage

## Features
- Product Management - List, Add, Edit and Delete Products
- Modern React Frontend
- Restful API Backend
- Persistent Data Storage with MongoDB

## Live Application

Application URL: http://34.149.47.249/

## Architecture
```
Internet
    ↓
Ingress Controller
    ↓

Frontend   |    Backend    
(Nginx)    |   (Node.js)   
Port: 80   |  Port: 5001  

      ↓

   MongoDB    
   StatefulSet  
   Port: 27017 

         ↓
[Persistent Volume]
```

## Prerequisites

- Kubernetes cluster (GKE, EKS, AKS, or local with Minikube/Kind)
- kubectl CLI configured
- NGINX Ingress Controller installed
- Docker images available on Docker Hub:
  - `sammaingi/yolo-frontend:1.2.1`
  - `sammaingi/yolo-backend:1.1.2`

## Project Structure

```

|-- namespace.yaml              # Creates yolo-app namespace
│
|-- mongodb/
|   |-- storage-class.yaml     # Defines storage for MongoDB
│   |-- configmap.yaml         # MongoDB configuration
│   |-- secret.yaml            # MongoDB credentials (base64 encoded)
│   |-- deployment.yaml        # MongoDB StatefulSet
│   |-- service.yaml           # Headless service for MongoDB
│
|-- backend/
│   |-- configmap.yaml         # Backend environment variables
│   |-- deployment.yaml        # Backend deployment with 2 replicas
│   |-- service.yaml           # ClusterIP service
│   |-- backendconfig.yaml     # GKE health check configuration
│
|-- frontend/
│   |-- configmap.yaml         # Frontend configuration
│   |-- deployment.yaml        # Frontend deployment with 2 replicas
│   |-- service.yaml           # ClusterIP service with session affinity
│
|-- ingress/
|   |-- ingress.yaml               # Routes external traffic to services
```


## Installation & Deployment Options

### Step 1: Clone Repo
```bash

git clone https://github.com/layersony/yolo-k8s.git

```

### Step 2: Create Namespace

```bash
cd manifest
kubectl apply -f namespace.yaml
```

### Step 3: Deploy MongoDB (Database Layer)

```bash
# Create storage class for persistent volumes
kubectl apply -f storage-class.yaml

# Deploy MongoDB configuration and secrets
kubectl apply -f mongodb/configmap.yaml
kubectl apply -f mongodb/secret.yaml

# Deploy MongoDB StatefulSet
kubectl apply -f mongodb/deployment.yaml
kubectl apply -f mongodb/service.yaml
```

**Wait for MongoDB to be ready:**
```bash
kubectl wait --for=condition=ready pod -l app=mongodb -n yolo-app --timeout=300s
```

### Step 4: Deploy Backend (Application Layer)

```bash
kubectl apply -f backend/configmap.yaml
kubectl apply -f backend/deployment.yaml
kubectl apply -f backend/service.yaml

# For GKE only
kubectl apply -f backend/backendconfig.yaml
```

**Verify backend is running:**
```bash
kubectl get pods -n yolo-app -l app=backend
```

### Step 5: Deploy Frontend (Presentation Layer)

```bash
kubectl apply -f frontend/configmap.yaml
kubectl apply -f frontend/deployment.yaml
kubectl apply -f frontend/service.yaml
```

### Step 6: Configure Ingress

```bash
kubectl apply -f ingress/ingress.yaml
```

**Get the ingress IP address:**
```bash
kubectl get ingress -n yolo-app
```

## Accessing the Application

Once deployed, access the application using the Ingress IP address:

- **Frontend**: `http://<INGRESS-IP>/`
- **Backend API**: `http://<INGRESS-IP>/api/products`

## Configuration Details

### MongoDB Credentials

The MongoDB credentials are stored in `mongodb/secret.yaml` (base64 encoded):
- **Username**: `admin`
- **Password**: `password123`

**Security Note**: Change these credentials in production by encoding new values:
```bash
echo -n 'username' | base64
echo -n 'password' | base64
```

### Environment Variables

**Backend Configuration** (`backend/configmap.yaml`):
- `NODE_ENV`: production
- `PORT`: 5001
- `MONGODB_URI`: Connection string to MongoDB StatefulSet

**Frontend Configuration** (`frontend/configmap.yaml`):
- `REACT_APP_API_URL`: /api

## Resource Allocation

| Component | Replicas | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|----------|-------------|-----------|----------------|--------------|
| Frontend  | 2        | 100m        | 250m      | 128Mi          | 256Mi        |
| Backend   | 2        | 250m        | 500m      | 256Mi          | 512Mi        |
| MongoDB   | 1        | 250m        | 500m      | 256Mi          | 512Mi        |

## Health Checks

All components have configured health probes:

**Frontend & Backend**:
- Liveness probes ensure containers restart if unhealthy
- Readiness probes prevent traffic to pods that aren't ready

**MongoDB**:
- Uses `db.adminCommand('ping')` for health verification

## Monitoring and Troubleshooting

### Check pod status
```bash
kubectl get pods -n yolo-app
```

### View logs
```bash
# Frontend logs
kubectl logs -f deployment/frontend -n yolo-app

# Backend logs
kubectl logs -f deployment/backend -n yolo-app

# MongoDB logs
kubectl logs -f statefulset/mongodb -n yolo-app
```

## Cleanup

To remove all resources:

```bash
# Delete all resources in the namespace
kubectl delete namespace yolo-app

# Delete storage class
kubectl delete storageclass yolo-storage
```

## Support

For issues or questions, please refer to:
- Kubernetes documentation: https://kubernetes.io/docs/
- NGINX Ingress Controller: https://kubernetes.github.io/ingress-nginx/

## License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE) file for details

## Authors

- layersony - [GitHub Profile](https://github.com/layersony)

## Acknowledgments

- Original project forked from [kadimasum/yolo](https://github.com/kadimasum/yolo)
- Built with the MERN stack
- Inspired by modern e-commerce platforms