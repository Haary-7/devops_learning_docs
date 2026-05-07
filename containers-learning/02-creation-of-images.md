# Creation of Container Images

## Dockerfile Basics

A Dockerfile is a text file containing instructions to build a Docker image.

### Common Dockerfile Instructions

| Instruction | Description |
|-------------|-------------|
| `FROM` | Base image to start from |
| `WORKDIR` | Set working directory |
| `COPY` | Copy files from host to image |
| `ADD` | Like COPY, but also supports URLs and tar extraction |
| `RUN` | Execute commands during build |
| `ENV` | Set environment variables |
| `EXPOSE` | Document which ports the container listens on |
| `CMD` | Default command to run when container starts |
| `ENTRYPOINT` | Configure container to run as executable |
| `USER` | Set user for subsequent instructions |
| `VOLUME` | Create mount points for data persistence |
| `ARG` | Define build-time variables |
| `LABEL` | Add metadata to the image |

## Example Dockerfiles

### Python Application

```dockerfile
# Use official Python base image
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Copy requirements first (for better caching)
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Set environment variable
ENV PYTHONUNBUFFERED=1

# Expose port
EXPOSE 8000

# Run the application
CMD ["python", "app.py"]
```

### Node.js Application

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci --only=production

COPY . .

ENV NODE_ENV=production

EXPOSE 3000

CMD ["node", "server.js"]
```

### Go Application (Multi-Stage Build)

```dockerfile
# Stage 1: Build
FROM golang:1.21 AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp .

# Stage 2: Run
FROM scratch

COPY --from=builder /app/myapp /myapp

EXPOSE 8080

ENTRYPOINT ["/myapp"]
```

## Building Images

### Basic Build Command

```bash
# Build with a tag
docker build -t myapp:latest .

# Build with a specific Dockerfile
docker build -t myapp:latest -f Dockerfile.prod .

# Build with build arguments
docker build -t myapp:latest --build-arg VERSION=1.0 .

# Build without cache
docker build -t myapp:latest --no-cache .

# Build for specific platform
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .
```

## Build Context

The build context is the set of files at the specified location. The `.` at the end of `docker build -t myapp .` means the current directory.

### Using .dockerignore

Create a `.dockerignore` file to exclude files from the build context:

```
.git
.gitignore
node_modules
__pycache__
*.md
.env
Dockerfile
.dockerignore
```

## Best Practices

### 1. Use Specific Base Image Tags

```dockerfile
# Good
FROM python:3.11.5-slim

# Bad (unpredictable)
FROM python:latest
```

### 2. Minimize Layers

```dockerfile
# Good (single layer)
RUN apt-get update && \
    apt-get install -y curl wget && \
    rm -rf /var/lib/apt/lists/*

# Bad (multiple layers, larger image)
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
```

### 3. Use Multi-Stage Builds

```dockerfile
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

### 4. Run as Non-Root User

```dockerfile
FROM python:3.11-slim

RUN useradd -m appuser
USER appuser

WORKDIR /home/appuser/app
COPY --chown=appuser:appuser . .
```

### 5. Leverage Build Cache

Order instructions from least to most frequently changed:

```dockerfile
FROM python:3.11-slim

# Rarely changes
COPY requirements.txt .
RUN pip install -r requirements.txt

# Changes frequently
COPY . .
```

### 6. Use HEALTHCHECK

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
```

## Alternative Image Builders

| Tool | Description |
|------|-------------|
| `docker build` | Traditional Docker builder |
| `docker buildx` | Extended builder with multi-platform support |
| `buildkit` | Next-generation builder (used by buildx) |
| `kaniko` | Build images in Kubernetes without Docker daemon |
| `Buildpacks` | Transform source code into images automatically |
| `Podman build` | Daemonless alternative to Docker build |
| `img` | Standalone, daemonless builder |

## Image Scanning and Security

```bash
# Scan image for vulnerabilities (Docker Scout)
docker scout cve myapp:latest

# Scan with Trivy
trivy image myapp:latest

# Check for secrets
docker build --secret id=mysecret,src=./secret.txt .
```
