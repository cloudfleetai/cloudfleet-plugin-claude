---
name: cfcr-overview
description: This skill should be used when the user asks about "container registry", "CFCR", "push image", "pull image", "registry.cloudfleet.dev", or needs help with container images on Cloudfleet.
user-invocable: false
---

# CFCR — Cloudfleet Container Registry

When users ask about CFCR or need help with container images, use this reference.

## Overview

CFCR (Cloudfleet Container Registry) is a managed OCI-compliant container registry, included with every Cloudfleet organization. Nothing has to be provisioned. It stores Docker images, multi-architecture manifest lists, and Helm charts as OCI artifacts.

## Registry URL

The organization ID and region are part of the **hostname**, not the path:

```
{organization-id}.{region}.registry.cloudfleet.dev/{repository}:{tag}
```

- `{organization-id}` — the UUID of the user's organization. `cloudfleet auth list-profiles` reports it as `organization_id` for each configured profile; it is also in the console under Billing → Payment.
- `{region}` — one of `europe`, `northamerica`, `apac`. Pick the one closest to where the workloads run, or the one data residency requires. `europe` is served entirely on EU infrastructure.
- `{repository}` — any path, e.g. `my-app` or `backend/api`. Do NOT add the organization as a path segment; it is already in the hostname.

Example: `dc78c04e-6651-4e5d-9c04-079f6532989b.europe.registry.cloudfleet.dev/backend/api:v1.2.3`

Each organization gets an isolated namespace. Credentials from one organization cannot reach another's images.

## Common Operations

**Authenticate Docker** (installs the CLI as a credential helper in `~/.docker/config.json`, so no `docker login` is needed afterwards):

```bash
cloudfleet auth configure-docker
```

**Push an image:**

```bash
docker tag myapp:latest <org-id>.europe.registry.cloudfleet.dev/myapp:latest
docker push <org-id>.europe.registry.cloudfleet.dev/myapp:latest
```

**Pull an image:**

```bash
docker pull <org-id>.europe.registry.cloudfleet.dev/myapp:latest
```

**Inspect the registry:**

```bash
cloudfleet repositories list                              # repositories across all regions
cloudfleet repositories tags list <region> <repository>   # tags with size and metadata
cloudfleet repositories tags describe <region> <repository> <tag>
cloudfleet repositories tags delete <region> <repository> <tag>
```

## Using CFCR images in CFKE

CFKE clusters authenticate to CFCR in the same organization automatically. No `imagePullSecrets`, service accounts, or credentials:

```yaml
containers:
    - name: myapp
      image: <org-id>.europe.registry.cloudfleet.dev/myapp:latest
```

## Access Control

CFCR uses the same roles as the rest of Cloudfleet: Administrators push and pull, Users pull only, and CFKE clusters have implicit pull access.
