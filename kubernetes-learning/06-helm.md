# Helm - Kubernetes Package Manager

## What is Helm?

Helm is the package manager for Kubernetes. It packages Kubernetes manifests into charts, manages releases, and supports templating for reusable configurations.

## Helm Architecture

```
┌─────────────────────────────────────┐
│            Helm CLI                 │
│                                     │
│  helm install/upgrade/rollback/etc  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│           Helm Library              │
│                                     │
│  - Chart rendering (templates)      │
│  - Release management               │
│  - Hooks execution                  │
│  - Validation                       │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│        Kubernetes API               │
│                                     │
│  - Creates/Updates K8s resources    │
│  - Stores release info as Secrets   │
└─────────────────────────────────────┘
```

## Key Concepts

| Term | Description |
|------|-------------|
| **Chart** | Package of pre-configured Kubernetes resources |
| **Release** | A running instance of a chart in a cluster |
| **Repository** | Location where charts are stored and shared |
| **Values** | Configuration values passed to templates |
| **Template** | Go template files that generate K8s manifests |
| **Hook** | Lifecycle events (pre-install, post-upgrade, etc.) |

## Helm Chart Structure

```
my-chart/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default configuration values
├── values.schema.json      # JSON Schema for values validation (optional)
├── Chart.lock              # Dependencies lock file
├── charts/                 # Subcharts/dependencies
├── templates/              # Kubernetes manifest templates
│   ├── NOTES.txt           # Post-install notes
│   ├── _helpers.tpl        # Template helpers/partials
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl
│   └── tests/
│       └── test-connection.yaml
└── crds/                   # Custom Resource Definitions
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-web-app
description: A Helm chart for deploying my web application
type: application           # application or library
version: 1.0.0              # Chart version (semver)
appVersion: "2.5.0"         # Application version
icon: https://example.com/icon.png
sources:
- https://github.com/example/my-web-app
keywords:
- web
- api
maintainers:
- name: Hari
  email: hari@example.com
  url: https://example.com
dependencies:
- name: postgresql
  version: "12.1.0"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
  tags:
  - database
- name: redis
  version: "17.0.0"
  repository: https://charts.bitnami.com/bitnami
  alias: cache-redis      # Use different name than subchart
annotations:
  example.com/license: Apache-2.0
```

### values.yaml

```yaml
# Default values for my-web-app

replicaCount: 3

image:
  repository: myregistry/myapp
  pullPolicy: IfNotPresent
  tag: ""  # Overrides appVersion

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9090"

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
    - ALL

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
  - host: myapp.example.com
    paths:
    - path: /
      pathType: Prefix
  tls:
  - secretName: myapp-tls
    hosts:
    - myapp.example.com

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}

# Config
config:
  appEnv: production
  logLevel: info
  databaseHost: postgres
  databasePort: 5432

# Secrets (use external secrets in production)
secrets:
  databasePassword: ""
  apiKey: ""
```

### Templates

#### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-web-app.fullname" . }}
  labels:
    {{- include "my-web-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-web-app.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "my-web-app.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "my-web-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /healthz
            port: http
          initialDelaySeconds: 15
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
        envFrom:
        - configMapRef:
            name: {{ include "my-web-app.fullname" . }}-config
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

#### _helpers.tpl

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "my-web-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-web-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "my-web-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-web-app.labels" -}}
helm.sh/chart: {{ include "my-web-app.chart" . }}
{{ include "my-web-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-web-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-web-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "my-web-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-web-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

#### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-web-app.fullname" . }}
  labels:
    {{- include "my-web-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: http
    protocol: TCP
    name: http
  selector:
    {{- include "my-web-app.selectorLabels" . | nindent 4 }}
```

#### ingress.yaml

```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "my-web-app.fullname" . }}
  labels:
    {{- include "my-web-app.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "my-web-app.fullname" $ }}
                port:
                  name: http
          {{- end }}
    {{- end }}
{{- end }}
```

#### hpa.yaml

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "my-web-app.fullname" . }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "my-web-app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

## Helm Commands

### Repository Management

```bash
# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Update repositories
helm repo update

# List repositories
helm repo list

# Search for charts
helm search repo nginx
helm search hub nginx        # Searches Artifact Hub

# Remove repository
helm repo remove bitnami
```

### Installing Charts

```bash
# Install with defaults
helm install my-release bitnami/nginx

# Install with custom values
helm install my-release bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=LoadBalancer

# Install from values file
helm install my-release ./my-chart \
  -f values-production.yaml

# Install with multiple values files (last wins)
helm install my-release ./my-chart \
  -f values-base.yaml \
  -f values-production.yaml

# Install in specific namespace
helm install my-release ./my-chart \
  --namespace production \
  --create-namespace

# Dry run (see what would be deployed)
helm install my-release ./my-chart --dry-run --debug

# Install and wait for resources
helm install my-release ./my-chart --wait --timeout 5m
```

### Managing Releases

```bash
# List releases
helm list
helm list -n production
helm list --all-namespaces

# Check status
helm status my-release
helm status my-release --show-resources

# Upgrade
helm upgrade my-release ./my-chart \
  -f values-production.yaml

# Upgrade with rollback on failure
helm upgrade my-release ./my-chart \
  --atomic \
  --timeout 5m

# View history
helm history my-release

# Rollback
helm rollback my-release          # Previous revision
helm rollback my-release 2        # Specific revision

# Uninstall
helm uninstall my-release
helm uninstall my-release --keep-history

# Test
helm test my-release
```

### Template Debugging

```bash
# Render templates without installing
helm template my-release ./my-chart

# Render with specific values
helm template my-release ./my-chart \
  -f values-production.yaml

# View Kubernetes manifests that would be applied
helm template my-release ./my-chart | kubectl apply -f - --dry-run=client

# Lint chart
helm lint ./my-chart

# Verify chart
helm template my-release ./my-chart --debug
```

### Packaging

```bash
# Package a chart
helm package ./my-chart

# Install from packaged chart
helm install my-release my-chart-1.0.0.tgz

# Start local chart repository server
helm serve
```

## Environment-Specific Values

```
my-chart/
├── values.yaml              # Defaults
├── values-dev.yaml          # Development overrides
├── values-staging.yaml      # Staging overrides
├── values-production.yaml   # Production overrides
└── templates/
```

### values-dev.yaml
```yaml
replicaCount: 1
image:
  tag: "latest"
resources:
  requests:
    cpu: 50m
    memory: 64Mi
ingress:
  enabled: false
autoscaling:
  enabled: false
```

### values-production.yaml
```yaml
replicaCount: 3
image:
  pullPolicy: Always
  tag: "2.5.0"
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 512Mi
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
ingress:
  enabled: true
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```

## Helm Hooks

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-db-migrate"
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["npm", "run", "migrate"]
```

### Hook Types

| Hook | When |
|------|------|
| `pre-install` | Before any resources created |
| `post-install` | After all resources created |
| `pre-delete` | Before deletion |
| `post-delete` | After deletion |
| `pre-upgrade` | Before upgrade |
| `post-upgrade` | After upgrade |
| `pre-rollback` | Before rollback |
| `post-rollback` | After rollback |
| `test` | When `helm test` is run |

### Hook Delete Policies

| Policy | Behavior |
|--------|----------|
| `hook-succeeded` | Delete hook resource after success |
| `hook-failed` | Delete hook resource after failure |
| `before-hook-creation` | Delete before next hook creation |

## Subcharts and Dependencies

### Chart.yaml with dependencies

```yaml
dependencies:
- name: postgresql
  version: "12.1.0"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
  tags:
  - database
- name: redis
  version: "17.0.0"
  repository: https://charts.bitnami.com/bitnami
  alias: cache

# Override subchart values in parent values.yaml
postgresql:
  enabled: true
  auth:
    postgresPassword: secret
  primary:
    persistence:
      size: 10Gi

cache:  # This is the alias
  enabled: true
  architecture: standalone
```

### Update dependencies

```bash
helm dependency update ./my-chart
# or
helm dep up ./my-chart

# Build from Chart.lock
helm dependency build ./my-chart
```

## Best Practices

1. **Use standard labels** - `app.kubernetes.io/name`, `app.kubernetes.io/instance`
2. **Include resource limits** - Always set requests and limits
3. **Use values validation** - `values.schema.json` for validation
4. **Document values** - Add comments in `values.yaml`
5. **Use `_helpers.tpl`** - Reusable template partials
6. **Test with `helm lint`** - Catch errors before deploying
7. **Use `--atomic` for upgrades** - Auto-rollback on failure
8. **Pin dependency versions** - Use specific versions, not ranges
9. **Namespace isolation** - Use `--namespace` for environment separation
10. **Store releases as secrets** - Default since Helm 3

## Helm vs Kustomize

| Feature | Helm | Kustomize |
|---------|------|-----------|
| Templating | Go templates | Overlays/patches |
| Package management | Yes (charts, repos) | No |
| Parameterization | values.yaml | kustomization.yaml |
| Built into kubectl | No | Yes (kubectl apply -k) |
| Complexity | Higher learning curve | Simpler |
| Best for | Reusable packages, third-party apps | Environment-specific configs |

## Interview Questions and Answers

### Q: What changed from Helm 2 to Helm 3?

- **No Tiller** - Removed server-side component
- **Release storage** - Now stored as Kubernetes Secrets (not ConfigMaps in kube-system)
- **JSON Patch** - Uses JSON merge patch instead of strategic merge
- **Better security** - No cluster-wide admin permissions needed
- **XDG paths** - Follows XDG Base Directory Specification
- **CRD support** - CRDs placed in separate directory with special handling

### Q: How does Helm handle upgrades?

1. Renders templates with new values
2. Compares with current release (3-way merge)
3. Generates a patch
4. Applies patch to Kubernetes API
5. Stores new release version as Secret
6. If `--atomic` is set and upgrade fails, auto-rolls back

### Q: What is a Helm hook?

A hook is a Kubernetes manifest with special annotations that runs at specific lifecycle events. Hooks run outside the normal release lifecycle, allowing operations like database migrations before a new deployment.

### Q: How to rollback a Helm release?

```bash
helm rollback <release> [revision]
```

Rollback creates a new release revision with the previous manifest and applies it. The failed release is preserved in history.

### Q: Can you use Helm with GitOps?

Yes. Helm charts can be managed declaratively with GitOps tools:
- **ArgoCD** - Has native Helm support
- **Flux** - Helm Controller for declarative Helm releases
- Charts are stored in Git, GitOps tools handle installation/upgrades
