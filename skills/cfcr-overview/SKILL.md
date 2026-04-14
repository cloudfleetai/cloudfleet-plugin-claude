---
name: cfcr-overview
description: This skill should be used when the user asks about "container registry", "CFCR", "push image", "pull image", "registry.cloudfleet.ai", or needs help with container images on Cloudfleet.
user-invocable: false
---

# CFCR — Cloudfleet Container Registry

When users ask about CFCR or need help with container images, use this reference.

## Overview

CFCR (Cloudfleet Container Registry) is a managed OCI-compatible container registry. It provides a secure, private registry for storing and distributing container images alongside your CFKE clusters.

## Key Details

- **Registry domain**: `registry.cloudfleet.ai`
- **Image format**: `registry.cloudfleet.ai/<organization>/<repository>:<tag>`
- **Authentication**: Uses Cloudfleet CLI credentials

## Common Operations

**Login to registry:**

```bash
cloudfleet auth configure-docker
```

**Push an image:**

```bash
docker tag myapp:latest registry.cloudfleet.ai/myorg/myapp:latest
docker push registry.cloudfleet.ai/myorg/myapp:latest
```

**Pull an image:**

```bash
docker pull registry.cloudfleet.ai/myorg/myapp:latest
```

## Using CFCR images in CFKE

CFKE clusters have built-in authentication to CFCR within the same organization. No `imagePullSecrets` are needed — just reference the image directly:

```yaml
containers:
    - name: myapp
      image: registry.cloudfleet.ai/myorg/myapp:latest
```
