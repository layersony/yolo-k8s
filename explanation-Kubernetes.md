# YOLO E-Commerce Application - Kubernetes Implementation Explanation

This Explanation demostates how I used Kubernetes to automate the deployment of my Dockerized YOLO e-commerce application with 3-teir stack (Frontend, Backend, and MongoDB).


## 1. Choice of Kubernetes Objects for Deployment

### MongoDB - StatefulSet

#### Why I used StatefulSet for MongoDB:

I used StatefulSet for MongoDB instead of a regular Deployment because of:

1. Stable Network Identity where it provides each pod with a unique, persistent hostname like `mongodb-0`. This is impotant for database connections that need to reference specific instances.

2. Ordered Deployment and Scaling: Pods are created, scaled, and deleted in a predictable order like 0,1,2,.... it ensures the primary database instance starts before replicas.

3. Persistent Storage: It automatically creates and manages PVCs (persistent volume claims) for each pod so even if the pod is deleted, the data persists and reattaches when the pod is recreated.


**Configuration:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb-service  # Headless service for stable DNS
  replicas: 1
  volumeClaimTemplates:  # Automatic PVC creation
  - metadata:
      name: mongodb-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: yolo-storage
      resources:
        requests:
          storage: 5Gi
```

Some Benefits:
- Predictable pod naming: `mongodb-0`
- Stable DNS: `mongodb-0.mongodb-service.yolo-app.svc.cluster.local`

---

### Frontend and Backend - Deployment

I chose to use regular deployment for both frontend and backend because of:

1. Stateless Applications where these services don't store data locally in that can be replaced or scaled without data loss concerns.

2. Can have multiple identical replicas that can handle requests interchangeably. Any pod can serve any request.

3. Deployments support zero-downtime updates by gradually replacing old pods with new ones.

4. Pods can be created in any order and replaced easily without concerns about identity or storage.

**Configuration:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1  # Allow 1 extra pod during updates
      maxUnavailable: 0  # Ensure no downtime
```

### Summary

| Component | Object Type | Reason |
|-----------|-------------|--------|
| MongoDB | StatefulSet | Needs persistent storage, stable network identity, ordered scaling |
| Backend | Deployment | Stateless, horizontally scalable, no local storage needed |
| Frontend | Deployment | Stateless, serves static files, no local storage needed |

---

## 2. Method Used to Expose Pods to Internet Traffic

### Three-Layer Exposure Strategy

### Layer 1: ClusterIP Services (Internal Communication)

**Frontend Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: ClusterIP # Internal cluster IP only
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  sessionAffinity: ClientIP  # Sticky sessions for better UX
```

**Backend Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 5001
    targetPort: 5001
```

**MongoDB Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  type: ClusterIP
  clusterIP: None  # Headless service for StatefulSet
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
```

The reason I used ClusterIP?
- Services are only accessible within the cluster
- Provides internal DNS names (`backend-service.yolo-app.svc.cluster.local`)
- It's more secure coz there is no direct external access
- Load balances traffic across multiple pods


### Layer 2: NGINX Ingress Controller (External Access)

**Ingress Resource:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: yolo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /api(/|$)(.*)  # Routes /api/* to backend
        backend:
          service:
            name: backend-service
            port:
              number: 5001
      - path: /  # Routes everything else to frontend
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

**How it works:**

1. Single Entry Point :- One external IP address serves the entire application
2. Path-Based Routing :- 
   - Requests to `/api/*` -> Backend service
   - All other requests -> Frontend service
3. URL Rewriting :- The annotation `rewrite-target: /$2` removes `/api` prefix before forwarding to backend

**Traffic Flow:**
```
User Request: http://<INGRESS_IP>/api/products
    ↓
Ingress rewrites to: /products
    ↓
Routes to: backend-service:5001/products
    ↓
Backend Pod handles request
```

---

### Layer 3: Frontend Nginx Proxy (Additional Routing)

The frontend container also includes nginx configuration for API proxying:

```nginx
location /api/ {
    proxy_pass http://backend-service.yolo-app.svc.cluster.local:5001/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

I choose this approach for it provides fallback routing if ingress is not available, best for development/testing environments and it's flexibility in deployment scenarios


### Why This Exposure Method?

#### Advantages:

1. Security :- Only the ingress controller is exposed to the internet
2. Cost-Effective :- Single load balancer IP instead of multiple
3. Centralized Management :- All routing rules in one place
4. SSL/TLS Termination :- Can easily add HTTPS at the ingress level
5. Scalability :- Backend services can scale independently

#### Alternative Methods (Not Used):


1. NodePort :- Exposes services on each node's IP which is not production ready 
2. LoadBalancer :- Creates separate cloud load balancer per service and it's expensive 
3. Direct Pod Access :- No load balancing, not fault-tolerant 

---

## 3. Use of Persistent Storage

### Persistent Storage Implementation

MongoDB requires persistent storage to ensure data survives:
- Pod restarts
- Node failures
- Cluster maintenance
- Container crashes

Without persistent storage, all data would be lost when a pod is deleted.

#### Storage Architecture

##### Storage Class Definition:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: yolo-storage
provisioner: pd.csi.storage.gke.io   # Google Cloud persistent disk
parameters:
  type: pd-standard  # Standard HDD (slow and cost-effective)
  replication-type: none
volumeBindingMode: WaitForFirstConsumer  # Creates volume only when pod is scheduled
allowVolumeExpansion: true  # Can increase size later
reclaimPolicy: Delete  # Deletes disk when PVC is deleted
```

##### Persistent Volume Claim Template (in StatefulSet):
```yaml
volumeClaimTemplates:
- metadata:
    name: mongodb-data
  spec:
    accessModes: ["ReadWriteOnce"] # One node can mount at a time
    storageClassName: yolo-storage
    resources:
      requests:
        storage: 5Gi # 5GB storage allocation
```

---
**This is how persistent storage Works**

1. When MongoDB StatefulSet is created, Kubernetes automatically creates a PVC (Persistent Volume Claim)

2. The storage class provisions a Google Cloud persistent disk based on the PVC request

3. The persistent volume is mounted to `/data/db` in the MongoDB container
   ```yaml
   volumeMounts:
   - name: mongodb-data
     mountPath: /data/db
   ```

4. All MongoDB data written to `/data/db` is stored on the persistent disk and not in the container filesystem

5. Even if the MongoDB pod is deleted, the persistent volume remains intact with all data


## Storage Lifecycle

```
Create StatefulSet
    ↓
Create PVC (mongodb-data-mongodb-0)
    ↓
StorageClass provisions PV (Google Cloud Disk)
    ↓
PV binds to PVC
    ↓
Volume mounts to Pod at /data/db
    ↓
MongoDB writes data to persistent storage
    ↓
Pod deleted/restarted
    ↓
New pod attaches to SAME persistent volume
    ↓
Data still available
```


#### Verification Commands

Check persistent volumes and claims:
```bash
# View persistent volume claims
kubectl get pvc -n yolo-app

# View persistent volumes
kubectl get pv

# Check storage class
kubectl get storageclass yolo-storage
```

Expected output:
```
NAME                        STATUS   VOLUME        CAPACITY   STORAGECLASS
mongodb-data-mongodb-0      Bound    pvc-xxxxx     5Gi        yolo-storage
```


## Summary

This YOLO application Kubernetes deployment demonstrates:

1. **Smart Object Selection**: StatefulSet for databases, Deployments for stateless apps
2. **Secure Exposure**: Multi-layer approach with ClusterIP services and Ingress controller
3. **Data Persistence**: Proper persistent storage for critical database data

The architecture is production-ready, scalable, and follows Kubernetes best practices.
