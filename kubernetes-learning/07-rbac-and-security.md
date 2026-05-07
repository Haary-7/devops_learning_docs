# RBAC, Security, and Service Accounts

## RBAC (Role-Based Access Control)

### RBAC API Primitives

| Resource | Scope | Description |
|----------|-------|-------------|
| **Role** | Namespace | Grants access within a namespace |
| **ClusterRole** | Cluster-wide | Grants access across all namespaces |
| **RoleBinding** | Namespace | Binds Role to subjects in a namespace |
| **ClusterRoleBinding** | Cluster-wide | Binds ClusterRole to subjects |

### Subjects

| Subject Type | Description |
|-------------|-------------|
| **User** | Human users (external managed) |
| **Group** | Group of users |
| **ServiceAccount** | Pod identity within cluster |

### Role and RoleBinding

```yaml
# Role: Define permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
# Read pods
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

# Read and write deployments
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]

# Delete pods
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["delete"]
  resourceNames: ["specific-pod"]  # Only these pods

# Non-resource URL (kubectl proxy)
- nonResourceURLs: ["/healthz", "/version"]
  verbs: ["get"]
---
# RoleBinding: Grant to users/service accounts
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# User
- kind: User
  name: hari@example.com
  apiGroup: rbac.authorization.k8s.io
# Group
- kind: Group
  name: dev-team
  apiGroup: rbac.authorization.k8s.io
# ServiceAccount
- kind: ServiceAccount
  name: my-app-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole and ClusterRoleBinding

```yaml
# ClusterRole: Cluster-wide permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/status"]
  verbs: ["get", "list", "watch"]

# Can also be used for:
# - Non-namespaced resources (nodes, PVs, namespaces)
# - Named resources across all namespaces
# - Non-resource endpoints (/healthz, /metrics)
---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
- kind: Group
  name: monitoring-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole in a RoleBinding

A ClusterRole can be bound within a namespace using a RoleBinding:

```yaml
# Uses ClusterRole but only grants access in the 'default' namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-global
  namespace: default
subjects:
- kind: User
  name: hari@example.com
roleRef:
  kind: ClusterRole
  name: pod-reader-cluster  # ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

### RBAC Verbs

| Verb | Description |
|------|-------------|
| `get` | Read single resource |
| `list` | Read collection of resources |
| `watch` | Watch for changes |
| `create` | Create resource |
| `update` | Update existing resource |
| `patch` | Partially update resource |
| `delete` | Delete resource |
| `deletecollection` | Delete multiple resources |
| `bind` | Bind roles |
| `escalate` | Escalate permissions |
| `impersonate` | Impersonate another user |
| `use` | Use a resource (PodSecurityPolicy, etc.) |

### Common RBAC Patterns

#### Read-Only Access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: readonly
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

#### Admin Access to Namespace

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-admin
  namespace: production
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

#### Pod Exec Access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-exec
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
```

### Built-in ClusterRoles

| ClusterRole | Description |
|------------|-------------|
| `cluster-admin` | Full access to everything |
| `admin` | Admin access within namespace |
| `edit` | Read/write within namespace (no RBAC) |
| `view` | Read-only within namespace |

```bash
# Bind built-in role
kubectl create rolebinding hari-admin \
  --clusterrole=admin \
  --user=hari@example.com \
  --namespace=production
```

## Service Accounts

### What is a Service Account?

A Service Account provides an identity for processes running in a Pod.

### Default Service Account

Every namespace has a `default` service account:

```bash
kubectl get serviceaccounts -n default
# NAME      SECRETS   AGE
# default   0         10d
```

If no `serviceAccountName` is specified, Pods use the `default` SA.

### Creating and Using Service Accounts

```yaml
# Create Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456:role/my-role
    iam.gke.io/gcp-service-account: my-sa@project.iam.gserviceaccount.com
automountServiceAccountToken: false
---
# Use in Pod
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  serviceAccountName: my-app-sa
  automountServiceAccountToken: true  # Override SA setting
  containers:
  - name: app
    image: myapp:1.0
```

### Service Account Token

When mounted in a Pod:

```
/var/run/secrets/kubernetes.io/serviceaccount/
├── token      # JWT token for API authentication
├── ca.crt     # Cluster CA certificate
└── namespace  # Current namespace
```

### Cloud Provider SA Integration

#### AWS EKS (IRSA)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/MyAppRole
```

#### GCP GKE (Workload Identity)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default
  annotations:
    iam.gke.io/gcp-service-account: my-app@my-project.iam.gserviceaccount.com
```

## Security Context

### Pod Security Context

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000           # UID to run containers
    runAsGroup: 3000          # Primary GID
    fsGroup: 2000             # Group for volumes
    runAsNonRoot: true        # Must run as non-root
    seccompProfile:
      type: RuntimeDefault    # RuntimeDefault, Localhost, Unconfined
    seLinuxOptions:
      level: "s0:c123,c456"
    supplementalGroups: [4000, 5000]
    sysctls:
    - name: net.core.somaxconn
      value: "1024"
```

### Container Security Context

```yaml
containers:
- name: app
  image: myapp:1.0
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    runAsNonRoot: true
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
    capabilities:
      add: ["NET_BIND_SERVICE"]
      drop: ["ALL"]
    procMount: Default
    seccompProfile:
      type: RuntimeDefault
```

### Capabilities

Linux capabilities grant specific privileges:

```yaml
# Drop all, add only needed
capabilities:
  drop: ["ALL"]
  add:
  - NET_BIND_SERVICE    # Bind to ports < 1024
  - CHOWN               # Change file ownership
  - SETUID              # Set UID
  - SETGID              # Set GID
  - NET_ADMIN           # Network configuration
  - SYS_PTRACE          # Process tracing
```

**Dangerous Capabilities:**
- `SYS_ADMIN` - Near-root access
- `NET_ADMIN` - Network manipulation
- `SYS_PTRACE` - Process inspection

## Pod Security Standards

K8s 1.25+ replaced PodSecurityPolicy with Pod Security Admission.

### Three Standards

| Standard | Description | Use Case |
|----------|-------------|---------|
| **Privileged** | No restrictions | System pods, infrastructure |
| **Baseline** | Prevents known escalations | General workloads |
| **Restricted** | Heavily restricted, best practices | Security-sensitive apps |

### Enforce at Namespace Level

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: baseline
    pod-security.kubernetes.io/warn-version: latest
```

### Restricted Standard Requirements

- Containers must run as non-root
- `allowPrivilegeEscalation: false`
- Drop ALL capabilities
- Use seccomp profile
- No privileged containers
- No host namespaces
- No host ports

## Network Security

### Network Policies

```yaml
# Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

## Secrets Encryption at Rest

```yaml
# EncryptionConfiguration for kube-apiserver
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}  # Fallback (reads unencrypted)
```

## Audit Logging

```yaml
# Audit Policy
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log all requests at RequestResponse level
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]

# Log metadata for pod changes
- level: Metadata
  resources:
  - group: ""
    resources: ["pods"]
  verbs: ["create", "update", "delete"]

# Don't log read requests to configmaps
- level: None
  resources:
  - group: ""
    resources: ["configmaps"]
  verbs: ["get", "list", "watch"]

# Default: log at Metadata level
- level: Metadata
```

## Best Practices

### Security Checklist

1. **Enable RBAC** - Never disable RBAC
2. **Least privilege** - Grant minimum required permissions
3. **No default SA** - Create dedicated SAs per application
4. **Disable automount** - Set `automountServiceAccountToken: false` unless needed
5. **Encrypt secrets** - Enable encryption at rest in etcd
6. **Network policies** - Default deny, allow specific traffic
7. **Pod security** - Enforce restricted standard
8. **Image scanning** - Scan images for vulnerabilities
9. **Read-only filesystem** - Use `readOnlyRootFilesystem: true`
10. **Resource limits** - Always set limits
11. **Drop capabilities** - Drop ALL, add only needed
12. **Run as non-root** - Always use non-root user
13. **Audit logging** - Enable and monitor audit logs
14. **Keep K8s updated** - Regular updates for security patches
15. **Use OPA/Gatekeeper** - Enforce policies

## Interview Questions and Answers

### Q: How does RBAC work in Kubernetes?

RBAC uses four objects:
1. **Role/ClusterRole** - Define permissions (what you can do)
2. **RoleBinding/ClusterRoleBinding** - Bind roles to subjects (who can do it)
3. **Subjects** - Users, Groups, or ServiceAccounts
4. **Verbs** - Actions (get, list, create, delete, etc.)

### Q: Difference between Role and ClusterRole?

- **Role** - Namespace-scoped, grants access within one namespace
- **ClusterRole** - Cluster-wide, grants access across all namespaces

### Q: How to restrict a pod from accessing the Kubernetes API?

1. Set `automountServiceAccountToken: false`
2. Use a dedicated ServiceAccount with no RBAC permissions
3. NetworkPolicy to block access to API server IP

### Q: What is the difference between runAsUser and runAsGroup?

- `runAsUser` - The UID that runs the container's entrypoint process
- `runAsGroup` - The primary GID for the container process
- `fsGroup` - Group ID applied to volume mounts

### Q: How to enforce that all pods run as non-root?

Use Pod Security Standards:
```yaml
labels:
  pod-security.kubernetes.io/enforce: restricted
```

Or OPA/Gatekeeper constraint:
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPMustRunAsNonRoot
```
