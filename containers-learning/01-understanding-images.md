# Understanding Container Images

## What is a Container Image?

A container image is a lightweight, standalone, executable package of software that includes everything needed to run an application:

- **Code** (application binaries, scripts)
- **Runtime** (Python, Node.js, Java, etc.)
- **System tools** (curl, vim, etc.)
- **System libraries** (shared dependencies)
- **Environment variables**
- **Configuration files**

Think of an image as a **blueprint** or **template** for containers. You can create multiple containers from a single image.

## Key Concepts

### Layers

Images are built in layers. Each instruction in a Dockerfile creates a new layer:

```
Layer 1: Base OS (Ubuntu 22.04)          ~80MB
Layer 2: Install Python                   ~50MB
Layer 3: Copy application code            ~5MB
Layer 4: Install dependencies             ~20MB
```

**Benefits of layers:**
- Shared layers between images save disk space
- Only changed layers need to be rebuilt
- Layers are cached, speeding up builds

### Image Registries

Places where images are stored and distributed:

| Registry | Description |
|----------|-------------|
| Docker Hub | Default public registry (hub.docker.com) |
| GitHub Container Registry | GitHub's registry (ghcr.io) |
| Amazon ECR | AWS container registry |
| Google Artifact Registry | GCP container registry |
| Azure Container Registry | Azure container registry |
| Harbor | Open-source enterprise registry |

### Image Tags

Tags are labels that identify specific versions of an image:

```
nginx:latest      # Latest version (default tag)
nginx:1.25        # Specific major.minor version
nginx:1.25.3      # Exact version
nginx:alpine      # Alpine-based smaller image
python:3.11-slim  # Slim variant of Python 3.11
```

### Multi-Architecture Images

Modern images can support multiple CPU architectures:

- `linux/amd64` (x86_64)
- `linux/arm64` (Apple Silicon, AWS Graviton)
- `linux/arm/v7` (Raspberry Pi)

A single image tag can contain variants for different architectures.

## Common Base Images

| Image | Size | Use Case |
|-------|------|----------|
| `alpine` | ~5MB | Minimal, security-focused |
| `debian-slim` | ~80MB | Debian with minimal packages |
| `ubuntu` | ~78MB | Full Ubuntu experience |
| `busybox` | ~1.2MB | Extremely minimal |
| `scratch` | 0MB | Empty image for statically compiled binaries |

## Image vs Container

| Aspect | Image | Container |
|--------|-------|-----------|
| Nature | Read-only template | Running instance |
| Storage | Stored on disk | Running in memory |
| Mutability | Immutable | Can be modified (ephemeral) |
| Analogy | Class (OOP) | Object/Instance (OOP) |
| Analogy | Recipe | Cake |
| Quantity | One image | Many containers from one image |
