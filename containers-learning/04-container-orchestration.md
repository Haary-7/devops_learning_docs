# Container Orchestration

## What is Container Orchestration?

Container orchestration automates the deployment, management, scaling, and networking of containers across multiple hosts. It solves problems that arise when running containers in production.

## Why Orchestration?

| Problem | Orchestration Solution |
|---------|------------------------|
| Running containers on multiple hosts | Scheduling and placement |
| Containers crashing | Automatic restart and self-healing |
| Scaling up/down | Horizontal Pod/Service autoscaling |
| Load balancing | Built-in service discovery and routing |
| Rolling updates | Zero-downtime deployments |
| Configuration management | Secrets and ConfigMaps |
| Storage provisioning | Persistent Volume management |
| Service discovery | DNS-based service discovery |

## Major Orchestration Platforms

### 1. Kubernetes (K8s)

The industry standard for container orchestration.

**Key Concepts:**

| Concept | Description |
|---------|-------------|
| **Pod** | Smallest deployable unit (one or more containers) |
| **Node** | Worker machine (VM or physical) |
| **Cluster** | Set of nodes managed by Kubernetes |
| **Deployment** | Manages replica sets and pod updates |
| **Service** | Stable network endpoint for pods |
| **Namespace** | Virtual cluster within a cluster |
| **ConfigMap** | Configuration data storage |
| **Secret** | Sensitive data storage |
| **PersistentVolume** | Storage abstraction |
| **Ingress** | HTTP/HTTPS routing rules |

**Architecture:**

```
┌─────────────────────────────────────────┐
│            Control Plane                │
│  ┌──────────┐  ┌───────────┐           │
│  │ API      │  │ Scheduler │           │
│  │ Server   │  │           │           │
│  └──────────┘  └───────────┘           │
│  ┌──────────┐  ┌───────────┐           │
│  │ etcd     │  │ Controller│           │
│  │ (Store)  │  │ Manager   │           │
│  └──────────┘  └───────────┘           │
└─────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
   ┌────▼───┐  ┌────▼───┐  ┌───▼────┐
   │ Node 1 │  │ Node 2 │  │ Node 3 │
   │ ┌────┐ │  │ ┌────┐ │  │ ┌────┐ │
   │ │Kube│ │  │ │Kube│ │  │ │Kube│ │
   │ │let │ │  │ │let │ │  │ │let │ │
   │ └────┘ │  │ └────┘ │  │ └────┘ │
   │ ┌────┐ │  │ ┌────┐ │  │ ┌────┐ │
   │ │Pods│ │  │ │Pods│ │  │ │Pods│ │
   │ └────┘ │  │ └────┘ │  │ └────┘ │
   └────────┘  └────────┘  └────────┘
```

**Example Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: myapp:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8000
  type: LoadBalancer
```

**Kubernetes Commands:**

```bash
# Get resources
kubectl get pods
kubectl get services
kubectl get deployments
kubectl get nodes
kubectl get all -n <namespace>

# Create resources
kubectl apply -f deployment.yaml
kubectl create -f deployment.yaml

# Update resources
kubectl apply -f deployment.yaml
kubectl set image deployment/web-app web-app=myapp:v2

# Debug
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs -f <pod-name>
kubectl exec -it <pod-name> -- /bin/bash

# Scale
kubectl scale deployment/web-app --replicas=5

# Delete
kubectl delete -f deployment.yaml
kubectl delete pod <pod-name>
```

### 2. Docker Swarm

Docker's native clustering and orchestration solution.

**Key Concepts:**

| Concept | Description |
|---------|-------------|
| **Manager Node** | Manages cluster state and schedules services |
| **Worker Node** | Runs containers (tasks) |
| **Service** | Definition of tasks to run on workers |
| **Task** | A single container running on a node |
| **Stack** | Group of services defined in compose file |

**Swarm Commands:**

```bash
# Initialize swarm
docker swarm init

# Join as worker
docker swarm join --token <token> <manager-ip>:2377

# List nodes
docker node ls

# Deploy a service
docker service create --name web --replicas 3 -p 8080:80 nginx

# Deploy a stack
docker stack deploy -c docker-compose.yml mystack

# List services
docker service ls

# Scale a service
docker service scale web=5

# View service logs
docker service logs web
```

**Swarm Compose Example:**

```yaml
version: "3.8"

services:
  web:
    image: myapp:latest
    ports:
      - "8080:8000"
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: "0.50"
          memory: 512M

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
    deploy:
      placement:
        constraints:
          - node.role == manager

volumes:
  db-data:
```

### 3. Other Orchestration Tools

| Tool | Description | Status |
|------|-------------|--------|
| **Amazon ECS** | AWS managed container orchestration | Active |
| **Azure AKS** | Azure managed Kubernetes | Active |
| **Google GKE** | Google managed Kubernetes | Active |
| **HashiCorp Nomad** | Simple, flexible orchestrator | Active |
| **Apache Mesos** | Distributed systems kernel | Deprecated |
| **OpenShift** | Enterprise Kubernetes (Red Hat) | Active |
| **Rancher** | Kubernetes management platform | Active |

## Key Orchestration Features

### 1. Auto-Scaling

**Horizontal Pod Autoscaler (HPA):**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### 2. Rolling Updates

```bash
# Update image with rollout
kubectl set image deployment/web-app web-app=myapp:v2 --record

# Check rollout status
kubectl rollout status deployment/web-app

# Rollback if issues
kubectl rollout undo deployment/web-app

# View rollout history
kubectl rollout history deployment/web-app
```

### 3. Service Discovery & Load Balancing

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP  # Internal only
```

Services are accessible via DNS: `backend.default.svc.cluster.local`

### 4. Health Checks

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 5. Secrets Management

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=          # base64 encoded
  password: cGFzc3dvcmQxMjM=  # base64 encoded
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp
        env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
```

## Comparison: Kubernetes vs Docker Swarm

| Feature | Kubernetes | Docker Swarm |
|---------|------------|--------------|
| Setup complexity | High | Low |
| Learning curve | Steep | Gentle |
| Auto-scaling | Yes (HPA/VPA) | Manual |
| Self-healing | Advanced | Basic |
| GUI Dashboard | Many options | Built-in |
| Load balancing | Advanced (Ingress) | Built-in |
| Storage orchestration | Extensive | Basic |
| Community | Very large | Smaller |
| Best for | Production at scale | Small teams/simple apps |
