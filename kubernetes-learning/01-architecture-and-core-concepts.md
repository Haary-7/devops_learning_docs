# Kubernetes Architecture and Core Concepts

## What is Kubernetes?

Kubernetes (K8s) is an open-source container orchestration platform originally developed by Google, now maintained by the Cloud Native Computing Foundation (CNCF). It automates deployment, scaling, and management of containerized applications.

## Kubernetes Architecture

### Control Plane Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CONTROL PLANE (Master)                        │
│                                                                     │
│  ┌─────────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │   kube-apiserver │  │   etcd       │  │   kube-scheduler       │ │
│  │                 │  │   (Key-Value   │  │                       │ │
│  │  - Frontend     │  │    Store)      │  │  - Assigns pods       │ │
│  │  - REST API     │  │               │  │    to nodes           │ │
│  │  - AuthN/AuthZ  │  │  - Stores     │  │  - Resource aware     │ │
│  │  - Validation   │  │    cluster     │  │  - Constraint aware   │ │
│  │  - Admission    │  │    state       │  │  - Affinity/Anti-     │ │
│  │    Controllers  │  │  - Watchable  │  │    affinity           │ │
│  └────────┬────────┘  └──────────────┘  └───────────┬────────────┘ │
│           │                                          │              │
│           └──────────────────────────────────────────┘              │
│                                      │                              │
│  ┌───────────────────────────────────▼──────────────────────────┐  │
│  │              kube-controller-manager                          │  │
│  │                                                               │  │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐  │  │
│  │  │ Node         │ │ Replication  │ │ Endpoint             │  │  │
│  │  │ Controller   │ │ Controller   │ │ Controller           │  │  │
│  │  └──────────────┘ └──────────────┘ └──────────────────────┘  │  │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐  │  │
│  │  │ Service      │ │ Resource     │ │ Namespace            │  │  │
│  │  │ Account Ctrl │ │ Quota Ctrl   │ │ Controller           │  │  │
│  │  └──────────────┘ └──────────────┘ └──────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              cloud-controller-manager                         │  │
│  │  - Routes, Service, LoadBalancer controllers                  │  │
│  │  - Cloud-provider specific logic (AWS, GCP, Azure)           │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

#### kube-apiserver
- **Only component** that communicates with etcd
- Validates and configures data for API objects
- Stateless, horizontally scalable
- Authentication, Authorization, Admission Control
- Supports REST and WebSocket protocols
- **Interview Point:** All `kubectl` commands go through the API server

#### etcd
- Distributed, consistent key-value store
- Stores ALL cluster state and configuration
- **Critical:** Regular backups are essential
- Uses Raft consensus algorithm
- **Interview Point:** If etcd is lost, the entire cluster state is lost

#### kube-scheduler
- Watches for newly created Pods with no assigned node
- Selects optimal node based on:
  - Resource requirements (CPU, memory)
  - Hardware constraints (GPU, SSD)
  - Affinity/anti-affinity rules
  - Taints and tolerations
  - Data locality

#### kube-controller-manager
Runs multiple controllers as a single binary:
- **Node Controller:** Monitors node health
- **Replication Controller:** Maintains correct number of pod replicas
- **Endpoint Controller:** Populates Endpoints objects
- **Service Account & Token Controllers:** Creates default accounts and API tokens
- **Namespace Controller:** Manages namespace lifecycle
- **Resource Quota Controller:** Enforces resource limits

### Worker Node Components

```
┌──────────────────────────────────────────────────────┐
│                    WORKER NODE                         │
│                                                       │
│  ┌─────────────────┐  ┌────────────────────────────┐ │
│  │    kubelet       │  │       kube-proxy           │ │
│  │                  │  │                            │ │
│  │  - Agent on node │  │  - Network proxy           │ │
│  │  - Registers     │  │  - Maintains network       │ │
│  │    node with     │  │    rules                   │ │
│  │    control plane │  │  - Load balancing          │ │
│  │  - Starts/stops  │  │  - Service discovery       │ │
│  │    pods          │  │  - Supports iptables/IPVS  │ │
│  │  - Pod health    │  └────────────────────────────┘ │
│  │    reporting     │                                  │
│  │  - Mounts        │  ┌────────────────────────────┐ │
│  │    volumes       │  │     Container Runtime      │ │
│  └────────┬─────────┘  │                            │ │
│           │            │  - containerd, CRI-O,      │ │
│           │            │    Docker (via shim)       │ │
│           │            │  - Runs containers          │ │
│  ┌────────▼─────────┐  └────────────────────────────┘ │
│  │      Pods        │                                  │
│  │  ┌────┐ ┌────┐  │  ┌────────────────────────────┐ │
│  │  │ C1 │ │ C2 │  │  │    Volumes                 │ │
│  │  └────┘ └────┘  │  │  - EmptyDir, HostPath,     │ │
│  │  ┌────┐ ┌────┐  │  │    PVC, ConfigMap, Secret  │ │
│  │  │ C3 │ │ C4 │  │  └────────────────────────────┘ │
│  │  └────┘ └────┘  │                                  │
│  └─────────────────┘                                  │
└──────────────────────────────────────────────────────┘
```

#### kubelet
- Primary "node agent"
- Registers node with API server
- Watches for PodSpecs assigned to this node
- Mounts volumes, downloads secrets
- Runs probes (liveness, readiness, startup)
- Reports node and pod status

#### kube-proxy
- Maintains network rules on nodes
- Enables network communication to/from Pods
- Implements Service concept using:
  - **iptables mode** (default, up to ~5000 services)
  - **IPVS mode** (better performance for large clusters)
- Handles session affinity, load balancing algorithms

#### Container Runtime
- Software responsible for running containers
- Must implement the Kubernetes Container Runtime Interface (CRI)
- Options: containerd (recommended), CRI-O, Docker (via cri-dockerd shim)

## Core Kubernetes Objects

### Namespace
Virtual cluster within a physical cluster.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
```

**Default Namespaces:**
- `default` - For user resources
- `kube-system` - Kubernetes system components
- `kube-public` - Publicly readable resources
- `kube-node-lease` - Node lease objects

### Labels and Selectors
Key-value pairs for organizing and selecting resources.

```yaml
metadata:
  labels:
    app: frontend
    env: production
    tier: web
    version: v2
```

**Selector Types:**
```yaml
# Equality-based
matchLabels:
  app: frontend
  env: production

# Set-based
matchExpressions:
  - key: env
    operator: In
    values: [production, staging]
  - key: tier
    operator: NotIn
    values: [test]
  - key: version
    operator: Exists
  - key: canary
    operator: DoesNotExist
```

### Annotations
Non-identifying metadata for tools and libraries.

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    kubernetes.io/ingress.class: nginx
    description: "My frontend application"
```

## Kubernetes API

### API Groups
- **Core API Group:** `/api/v1` (Pods, Services, Nodes, Namespaces)
- **Named API Groups:** `/apis/<group>/<version>` (Deployments, Ingress, etc.)

### API Versions
- `v1` - Stable, production-ready
- `beta` - Well-tested, may change
- `alpha` - Experimental, may be buggy

### Declarative vs Imperative

```bash
# Imperative (commands)
kubectl run nginx --image=nginx
kubectl expose deploy nginx --port=80 --type=ClusterIP
kubectl scale deploy nginx --replicas=3

# Declarative (YAML - Recommended)
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ./manifests/

# Imperative Configuration
kubectl create -f deployment.yaml  # Must exist
kubectl replace -f deployment.yaml # Must exist
```

## Resource Requirements and Limits

```yaml
resources:
  requests:
    cpu: "250m"      # 0.25 CPU cores (guaranteed)
    memory: "256Mi"  # 256 MiB (guaranteed)
  limits:
    cpu: "500m"      # 0.5 CPU max
    memory: "512Mi"  # 512 MiB max
```

### CPU Units
- `1` = 1 full core
- `500m` = 500 millicores = 0.5 cores
- `100m` = 100 millicores = 0.1 cores

### Memory Units
- `Mi` = Mebibytes (binary)
- `Gi` = Gibibytes (binary)
- `M` = Megabytes (decimal)
- `G` = Gigabytes (decimal)

### QoS Classes

| Class | Conditions | Eviction Priority |
|-------|-----------|-------------------|
| **Guaranteed** | requests = limits for all containers | Last to be evicted |
| **Burstable** | At least one request or limit set | Middle |
| **BestEffort** | No requests or limits | First to be evicted |

```yaml
# Guaranteed
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# BestEffort
# (no resources section)

# Burstable
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

## Interview Critical Concepts

### The Kubernetes Reconciliation Loop

Controllers follow a pattern:
1. **Observe** the desired state (from YAML/API)
2. **Observe** the current state (from cluster)
3. **Diff** the two states
4. **Act** to make current match desired

This is called **level-triggered** (not edge-triggered). The controller continuously reconciles.

### Declarative vs Event-Driven

- **Kubernetes is declarative:** You specify desired state, system figures out how
- **NOT event-driven:** Controllers don't react to events, they observe and reconcile
- **Level-triggered:** Controllers can restart and still converge to correct state

### Kubernetes as an API-Driven System

Everything in Kubernetes is an API object. When you `kubectl apply -f file.yaml`:
1. kubectl sends YAML to kube-apiserver
2. API server validates and stores in etcd
3. Controller notices new/changed object
4. Controller takes actions to match desired state
5. Status is reported back through API server

### Control Plane High Availability

```
┌──────────────────────────────────────────────┐
│            HA Control Plane                   │
│                                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ Master1 │  │ Master2 │  │ Master3 │      │
│  │         │  │         │  │         │      │
│  │  API    │  │  API    │  │  API    │      │
│  │Scheduler│  │Scheduler│  │Scheduler│      │
│  │ Ctr-Mgr │  │ Ctr-Mgr │  │ Ctr-Mgr │      │
│  └─────────┘  └─────────┘  └─────────┘      │
│        │           │           │             │
│  ┌─────▼───────────▼───────────▼─────┐       │
│  │         etcd cluster (odd#)       │       │
│  │         (Raft consensus)          │       │
│  └───────────────────────────────────┘       │
└──────────────────────────────────────────────┘
```

- Minimum 3 masters for HA
- etcd cluster should have odd number of members (3, 5, 7)
- Load balancer fronts the API servers
- Leader election for scheduler and controller-manager

### Kubernetes Networking Requirements

Every Kubernetes implementation MUST satisfy:
1. All pods can communicate with all other pods without NAT
2. All nodes can communicate with all pods
3. All pods can communicate with all nodes without NAT

### Key Interview Distinctions

| Concept | Description |
|---------|-------------|
| Desired State | What you declare in YAML |
| Current State | What actually exists in the cluster |
| Reconciliation | Process of making current = desired |
| Convergence | Eventual achievement of desired state |
| Drift | Difference between desired and current |
| Idempotent | Applying same config multiple times = same result |
