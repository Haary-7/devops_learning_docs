# Services, Ingress, and Networking

## Kubernetes Networking Model

Every implementation must provide:
1. **Pod-to-Pod:** All pods can communicate without NAT
2. **Node-to-Pod:** All nodes can reach all pods
3. **Pod-to-Node:** All pods can reach all nodes

## Pod Networking

- Each Pod gets a unique IP address
- Containers in the same Pod share the same network namespace (same IP, same port space)
- Pod IPs are only routable within the cluster

## Services

### Why Services?

Pods are ephemeral - they get new IPs when restarted. Services provide:
- **Stable IP address and DNS name**
- **Load balancing** across pod replicas
- **Service discovery** within the cluster

### Service Types

#### 1. ClusterIP (Default)

Internal access only, within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - name: http
    protocol: TCP
    port: 80          # Service port
    targetPort: 8080  # Pod port
  sessionAffinity: None  # None or ClientIP
```

**Access:** `my-service.default.svc.cluster.local:80`

#### 2. NodePort

Exposes service on each node's IP at a static port.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # Optional, range: 30000-32767
```

**Access:** `<NodeIP>:30080`

```
     External
        │
   ┌────▼────┐
   │ Node 1  │:30080  ─┐
   └─────────┘        │
   ┌─────────┐        ▼
   │ Node 2  │:30080  ──► [Pods]
   └─────────┘        ▲
   ┌─────────┐        │
   │ Node 3  │:30080  ─┘
   └─────────┘
```

#### 3. LoadBalancer

Creates external load balancer (cloud provider).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  externalTrafficPolicy: Cluster  # Cluster or Local
  loadBalancerIP: ""               # Request specific IP
  loadBalancerSourceRanges:        # IP whitelist
  - 10.0.0.0/8
```

#### 4. ExternalName

Maps service to external DNS name.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-database
spec:
  type: ExternalName
  externalName: mydb.example.com
```

**Access:** `my-database.default.svc.cluster.local` → resolves to `mydb.example.com`

### Headless Services

No ClusterIP, returns all Pod IPs directly.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-headless
spec:
  clusterIP: None  # HEADLESS
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

**Use Cases:**
- StatefulSets (each pod needs direct access)
- Custom service discovery
- Direct pod-to-pod communication

**DNS returns A records for ALL pods:**
```
$ nslookup my-headless.default.svc.cluster.local
my-headless.default.svc.cluster.local → 10.244.1.5, 10.244.1.6, 10.244.1.7
```

### Service Discovery

#### DNS-Based

Services are accessible via DNS:
```
# Same namespace
my-service

# Different namespace
my-service.other-namespace

# FQDN
my-service.other-namespace.svc.cluster.local
```

#### Environment Variables

```bash
# Automatically injected into pods
MY_SERVICE_SERVICE_HOST=10.96.100.1
MY_SERVICE_SERVICE_PORT=80
MY_SERVICE_PORT=tcp://10.96.100.1:80
MY_SERVICE_PORT_80_TCP=tcp://10.96.100.1:80
MY_SERVICE_PORT_80_TCP_PROTO=tcp
MY_SERVICE_PORT_80_TCP_PORT=80
MY_SERVICE_PORT_80_TCP_ADDR=10.96.100.1
```

### Endpoints

Endpoints track which pods back a service.

```bash
# View endpoints
kubectl get endpoints my-service
kubectl get endpoints my-service -o yaml
```

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
- addresses:
  - ip: 10.244.1.5
    targetRef:
      kind: Pod
      name: myapp-abc123
  - ip: 10.244.1.6
    targetRef:
      kind: Pod
      name: myapp-def456
  ports:
  - port: 8080
    protocol: TCP
```

### EndpointSlices (K8s 1.21+)

More scalable than Endpoints for large services.

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-abc12
  labels:
    kubernetes.io/service-name: my-service
addressType: IPv4
ports:
- name: http
  protocol: TCP
  port: 8080
endpoints:
- addresses:
  - 10.244.1.5
  conditions:
    ready: true
- addresses:
  - 10.244.1.6
  conditions:
    ready: true
```

## Ingress

### What is Ingress?

Manages external HTTP/HTTPS access to services with routing rules.

```
  Internet
      │
   ┌──▼──────────────┐
   │  Ingress        │
   │  Controller     │
   │  (nginx,        │
   │   traefik, etc) │
   └──┬──────────────┘
      │
   ┌──▼──────────────┐     ┌─────────────┐
   │  /api           │────▶│ api-service  │
   │  /web           │────▶│ web-service  │
   │  /admin         │────▶│ admin-service│
   └─────────────────┘     └─────────────┘
```

### Basic Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### TLS Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Path Types

| Path Type | Matching | Example |
|-----------|----------|---------|
| `Exact` | Exact match | `/api` matches only `/api` |
| `Prefix` | Prefix with `/` | `/api` matches `/api`, `/api/`, `/api/v1` |
| `ImplementationSpecific` | Controller-specific | Varies |

### Ingress Controllers

Popular options:
- **NGINX Ingress** (most popular)
- **Traefik** (cloud-native)
- **HAProxy Ingress**
- **Istio Gateway** (service mesh)
- **AWS ALB Controller**
- **GKE Ingress**

## Network Policies

Control traffic flow between pods.

### Default Behavior

Without NetworkPolicies: **All pods can communicate with all pods**

### NetworkPolicy Examples

#### Deny all ingress traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}  # All pods in namespace
  policyTypes:
  - Ingress
```

#### Allow from specific namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

#### Allow from specific CIDR

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8
        except:
        - 10.0.5.0/24
    ports:
    - protocol: TCP
      port: 443
```

#### Egress policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to: []
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  # Allow database
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
```

### Network Policy Flowchart

```
Incoming traffic to pod
         │
         ▼
┌────────────────────┐
│ Any NetworkPolicy  │
│ selects this pod?  │
└────────┬───────────┘
         │
    ┌────┴────┐
    │         │
   Yes        No → ALLOWED (default open)
    │
    ▼
┌────────────────────┐
│ Does traffic match │
│ any ingress rule?  │
└────────┬───────────┘
         │
    ┌────┴────┐
    │         │
   Yes        No → DENIED
    │
    ▼
  ALLOWED
```

## DNS in Kubernetes

### CoreDNS

Default DNS provider.

```yaml
# CoreDNS ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
            max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

### DNS Records

| Record Type | Format | Example |
|-------------|--------|---------|
| Service | `<name>.<namespace>.svc.cluster.local` | `my-service.default.svc.cluster.local` |
| Pod | `<pod-ip>.<namespace>.pod.cluster.local` | `10-244-1-5.default.pod.cluster.local` |
| Headless Service | Returns all pod IPs | `mysql-0.mysql.default.svc.cluster.local` |

### Pod DNS Policy

```yaml
dnsPolicy: ClusterFirst       # Default - uses cluster DNS
dnsPolicy: Default            # Uses host DNS
dnsPolicy: ClusterFirstWithHostNet  # Cluster DNS even with hostNetwork
dnsPolicy: None               # Custom DNS

# Custom DNS config
dnsConfig:
  nameservers:
  - 1.2.3.4
  searches:
  - ns1.svc.cluster-domain.example
  - my.dns.search.suffix
  options:
  - name: ndots
    value: "5"
  - name: edns0
```

## kube-proxy Modes

### iptables Mode (Default)

- Creates iptables rules for each Service
- Performance degrades with many services (~5000 max recommended)
- Random load balancing

### IPVS Mode

- Uses Linux IPVS (IP Virtual Server)
- Better performance for large clusters
- Supports multiple load balancing algorithms:
  - `rr` - Round Robin
  - `lc` - Least Connections
  - `dh` - Destination Hashing
  - `sh` - Source Hashing
  - `sed` - Shortest Expected Delay
  - `nq` - Never Queue

```yaml
# Enable IPVS in kube-proxy ConfigMap
mode: "ipvs"
ipvs:
  scheduler: "rr"
```

## Interview Questions and Answers

### Q: How does Service load balancing work?

**kube-proxy** creates iptables/IPVS rules on each node. When traffic hits the Service IP:
1. iptables/IPVS intercepts the traffic
2. Selects a backend pod (random in iptables, configurable in IPVS)
3. DNATs (rewrites) the destination IP to the pod IP
4. Pod receives the traffic and responds

### Q: What happens when a Pod fails a readiness probe?

1. kubelet marks the Pod as not ready
2. Endpoint controller removes the Pod's IP from the Service's Endpoints
3. kube-proxy updates iptables/IPVS rules
4. Service stops routing traffic to this Pod
5. Pod continues running, not restarted

### Q: Difference between ClusterIP, NodePort, and LoadBalancer?

- **ClusterIP:** Internal only, virtual IP within cluster
- **NodePort:** Exposes on each node's IP at port 30000-32767, built on ClusterIP
- **LoadBalancer:** Creates external LB (cloud provider), built on NodePort

### Q: What is an Ingress Controller vs Ingress resource?

- **Ingress:** YAML resource defining routing rules (declarative)
- **Ingress Controller:** Actual software (nginx, traefik) that implements those rules

Ingress does nothing without a controller running.

### Q: How does NetworkPolicy work?

NetworkPolicies are enforced by the CNI plugin (Calico, Cilium, etc.). Not all CNIs support NetworkPolicy. If a pod is selected by a NetworkPolicy with ingress rules, only matching traffic is allowed; all other traffic is denied.

### Q: What is the difference between NodePort and HostPort?

- **NodePort:** Service feature, creates iptables rules, load balanced across all pods
- **HostPort:** Binds directly to node port, no load balancing, one pod per node
