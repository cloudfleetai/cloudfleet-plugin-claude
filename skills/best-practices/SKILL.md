---
name: best-practices
description: This skill should be used when the user creates deployments, services, or asks about resource requests, load balancers, node placement, GPU workloads, or production readiness on CFKE.
user-invocable: false
---

# CFKE Best Practices

Critical guidance for deploying and operating workloads on Cloudfleet Kubernetes Engine. For detailed examples and YAML snippets, see [references/best-practices.md](references/best-practices.md).

## 1. Resource Requests (CRITICAL)

Always set CPU and memory requests on ALL containers, including sidecar init containers. CFKE's auto-provisioner depends on these for node sizing and scaling.

**Do NOT set CPU limits.** CPU limits cause unnecessary CFS throttling. Set CPU requests for scheduling only.

CFKE ships a `ValidatingAdmissionPolicy` named `resource-requests-are-not-set` that names the containers missing requests. It only warns, so the object is still admitted and the warning is easy to lose in Helm output. Treat it as the signal it is.

Set requests in chart values before the first install of any third-party chart. Upstream charts almost always ship `resources: {}`.

## 2. Load Balancer Services (CRITICAL)

Always use `externalTrafficPolicy: Local`. The default 'Cluster' mode creates LBs in every cloud/region where nodes exist, causing extra costs and inefficient routing.

## 3. Node Placement Constraints

Without constraints, the auto-provisioner picks the cheapest node from ANY configured provider. Use CFKE node labels to control placement:

- `cfke.io/provider` — cloud provider
- `topology.kubernetes.io/zone` — availability zone
- `cfke.io/accelerator-name` — GPU type

See [references/best-practices.md](references/best-practices.md) for nodeSelector, affinity, and topologySpreadConstraints examples.

## 4. Quick Reference

- **PodDisruptionBudgets**: Define PDBs for critical workloads
- **Health Checks**: Configure liveness/readiness probes on all containers
- **Security**: No privileged containers, versioned image tags, NetworkPolicies, TLS on Ingresses
- **GPU/ML**: Use `cfke.io/accelerator-name` label, set `nvidia.com/gpu` requests
- **Container Registries**: CFCR, AWS ECR, and GCP Artifact Registry work without imagePullSecrets
