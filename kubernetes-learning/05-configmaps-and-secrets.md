# ConfigMaps and Secrets

## ConfigMaps

Store non-confidential configuration data as key-value pairs.

### Creating ConfigMaps

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_PORT=8080 \
  --from-literal=LOG_LEVEL=info

# From a file
kubectl create configmap nginx-config \
  --from-file=nginx.conf

# From a directory
kubectl create configmap app-config-dir \
  --from-file=/path/to/config-dir/

# From env file
kubectl create configmap app-env-config \
  --from-env-file=.env

# Declarative
kubectl apply -f configmap.yaml
```

### ConfigMap YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # Individual key-value pairs
  APP_ENV: production
  APP_PORT: "8080"
  LOG_LEVEL: info
  DATABASE_HOST: postgres.default.svc.cluster.local
  DATABASE_PORT: "5432"
  DATABASE_NAME: myapp

  # Multi-line data
  nginx.conf: |
    server {
        listen 80;
        server_name myapp.example.com;

        location / {
            proxy_pass http://localhost:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        location /static {
            alias /app/static;
            expires 30d;
        }
    }

  application.properties: |
    spring.datasource.url=jdbc:postgresql://postgres:5432/myapp
    spring.datasource.username=admin
    spring.jpa.hibernate.ddl-auto=update
    spring.jpa.show-sql=false
---
# ConfigMap with binary data
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-binary-config
binaryData:
  logo.png: <base64-encoded-data>
  certificate.crt: <base64-encoded-data>
```

### Using ConfigMaps in Pods

#### 1. As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    
    # Individual keys
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    - name: APP_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_PORT
    
    # All key-value pairs
    envFrom:
    - configMapRef:
        name: app-config
    - configMapRef:
        name: database-config
        optional: true   # Won't fail if ConfigMap doesn't exist
```

#### 2. As Volume Mounts

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    
    # Mount entire ConfigMap as directory
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/conf.d
    
    # Mount specific keys as individual files
    - name: app-config-volume
      mountPath: /etc/config/app.properties
      subPath: application.properties

  volumes:
  # Entire ConfigMap as volume
  - name: config-volume
    configMap:
      name: app-config
  
  # Selective keys
  - name: app-config-volume
    configMap:
      name: app-config
      items:
      - key: application.properties
        path: app.properties
      - key: nginx.conf
        path: nginx.conf
  
  # With default mode
  - name: config-volume
    configMap:
      name: app-config
      defaultMode: 0644  # File permissions
      optional: false
```

### ConfigMap Volume Structure

```
/etc/nginx/conf.d/
├── APP_ENV          → "production"
├── APP_PORT         → "8080"
├── nginx.conf       → server { ... }
└── application.properties → spring.datasource.url=...
```

### Updating ConfigMaps

| Method | Env Vars | Volume Mount |
|--------|----------|--------------|
| **Update ConfigMap** | NO auto-update | Auto-updates (within ~1 min) |
| **Restart Pod** | Required to get new values | Gets new values |

**Note:** Environment variables are set at container start and don't change. Use volume mounts for dynamic config updates.

## Secrets

Store sensitive data (passwords, tokens, keys).

### Secret Types

| Type | Description |
|------|-------------|
| `Opaque` | Generic secrets (default) |
| `kubernetes.io/service-account-token` | Service account tokens |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/tls` | TLS certificates |
| `kubernetes.io/ssh-auth` | SSH credentials |
| `bootstrap.kubernetes.io/token` | Bootstrap tokens |

### Creating Secrets

```bash
# From literal (values are base64 encoded automatically)
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=S3cur3P@ssw0rd!

# From file
kubectl create secret generic tls-secret \
  --from-file=tls.crt=/path/to/tls.crt \
  --from-file=tls.key=/path/to/tls.key

# From env file
kubectl create secret generic app-secret \
  --from-env-file=.env

# Declarative
kubectl apply -f secret.yaml
```

### Secret YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: default
type: Opaque
data:
  # Values must be base64 encoded
  username: YWRtaW4=                    # echo -n 'admin' | base64
  password: UzNjdXIzUEBzc3cwcmQh        # echo -n 'S3cur3P@ssw0rd!' | base64
---
# String data (plain text, auto-encoded by K8s)
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  username: admin
  password: "S3cur3P@ssw0rd!"
---
# TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
---
# Docker Registry Secret
apiVersion: v1
kind: Secret
metadata:
  name: registry-secret
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>

# Or create with kubectl:
# kubectl create secret docker-registry registry-secret \
#   --docker-server=https://index.docker.io/v1/ \
#   --docker-username=myuser \
#   --docker-password=mypassword \
#   --docker-email=myuser@example.com
```

### Using Secrets in Pods

#### 1. As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    
    # Individual secret keys
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    
    # All secrets from one Secret
    envFrom:
    - secretRef:
        name: db-secret
    - secretRef:
        name: api-keys
        optional: true
```

#### 2. As Volume Mounts

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
    
    # Mount specific keys
    - name: tls-volume
      mountPath: /etc/tls
      readOnly: true

  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
      defaultMode: 0400  # Owner read-only
  
  - name: tls-volume
    secret:
      secretName: tls-secret
      items:
      - key: tls.crt
        path: tls.crt
      - key: tls.key
        path: tls.key
        mode: 0400
```

**Secret volume structure:**
```
/etc/secrets/
├── username  → "admin"
└── password  → "S3cur3P@ssw0rd!"
```

#### 3. As imagePullSecrets

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-registry-pod
spec:
  imagePullSecrets:
  - name: registry-secret
  containers:
  - name: app
    image: private-registry.example.com/myapp:1.0
```

### Service Account Token Secrets

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  serviceAccountName: my-service-account
  automountServiceAccountToken: true
  containers:
  - name: app
    image: myapp:1.0
    # Automatically mounted at:
    # /var/run/secrets/kubernetes.io/serviceaccount/
    # Contains: token, ca.crt, namespace
```

## External Secrets Management

### Sealed Secrets (Bitnami)

Encrypts secrets so they can be safely stored in Git.

```bash
# Install kubeseal CLI
# Encrypt a secret
kubectl create secret generic db-secret \
  --from-literal=password=S3cur3P@ssw0rd! \
  --dry-run=client -o yaml | kubeseal > sealed-secret.yaml

# Apply sealed secret
kubectl apply -f sealed-secret.yaml

# Controller decrypts it in the cluster
```

### External Secrets Operator

Fetches secrets from external providers.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: db-secret
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: myapp/database
      property: username
  - secretKey: password
    remoteRef:
      key: myapp/database
      property: password
```

### SOPS + Mozilla SOPS

```yaml
# .sops.yaml
creation_rules:
- path_regex: .*\.enc\.yaml$
  kms: 'arn:aws:kms:us-east-1:123456789:key/abc123'
  pgp: 'ABCDEF1234567890'
  age: 'age1...'

# Encrypt
sops -e -i secret.yaml

# Decrypt
sops -d secret.yaml | kubectl apply -f -
```

## Best Practices

### ConfigMaps

1. **Don't store sensitive data** - use Secrets instead
2. **Use descriptive names** - `frontend-config`, `database-config`
3. **Namespace them** - ConfigMaps are namespace-scoped
4. **Version your configs** - Include version in name: `app-config-v2`
5. **Keep configs small** - ConfigMaps limited to 1MB

### Secrets

1. **Enable encryption at rest** - Encrypt etcd data:
```yaml
# kube-apiserver configuration
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-key>
  - identity: {}
```

2. **Use RBAC** - Restrict who can read secrets
3. **Don't log secrets** - Be careful with `kubectl describe` and logs
4. **Use external secret managers** - AWS Secrets Manager, HashiCorp Vault
5. **Rotate secrets regularly** - Use automated rotation tools

## Security Considerations

### Secrets are NOT Encrypted by Default

- Secrets are base64 encoded, NOT encrypted
- Anyone with `get secrets` permission can read them
- Stored in etcd in plain text (unless encryption at rest is enabled)

### RBAC for Secrets

```yaml
# Role to read specific secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-secret", "api-keys"]
  verbs: ["get", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: default
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## ConfigMap + Secret Together

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: full-app
spec:
  containers:
  - name: app
    image: myapp:1.0
    
    # Config from ConfigMap
    envFrom:
    - configMapRef:
        name: app-config
    
    # Secrets as env vars
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: api-keys
          key: stripe-key
    
    # Config as files
    volumeMounts:
    - name: config-files
      mountPath: /etc/config
    - name: tls-certs
      mountPath: /etc/tls
      readOnly: true

  volumes:
  - name: config-files
    configMap:
      name: app-config
  - name: tls-certs
    secret:
      secretName: tls-secret
```

## Interview Questions and Answers

### Q: Are Kubernetes Secrets encrypted?

No, secrets are only base64 encoded by default. To encrypt them, you must enable encryption at rest in etcd using an EncryptionConfiguration with providers like aescbc, secretbox, or kms.

### Q: Can ConfigMaps store binary data?

Yes, ConfigMaps have a `binaryData` field for base64-encoded binary data. The total size of a ConfigMap (data + binaryData) is limited to 1MB.

### Q: What happens when you update a ConfigMap used as a volume?

The changes are propagated to the volume automatically within about 1 minute (kubelet sync period). However, the application must be designed to reload configuration, or you need to restart the pods.

### Q: How do you share a ConfigMap between namespaces?

ConfigMaps are namespace-scoped and cannot be shared directly. Options:
1. Copy the ConfigMap to each namespace
2. Use a tool like kustomize or Helm to manage cross-namespace configs
3. Use an external config management system

### Q: Difference between data and stringData in Secrets?

- `data`: Values must be base64 encoded before applying
- `stringData`: Values are plain text, K8s auto-encodes them
- Both produce the same result in etcd

### Q: How to prevent secrets from appearing in etcd backups?

Enable encryption at rest so secrets are encrypted before being stored in etcd. When backing up etcd, the backup will contain encrypted data that requires the encryption key to decrypt.
