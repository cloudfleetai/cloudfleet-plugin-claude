---
name: cfke-overview
description: This skill should be used when the user asks "how does CFKE work", "what is CFKE", "explain the architecture", or needs to understand Cloudfleet Kubernetes Engine concepts.
user-invocable: false
---

# CFKE Architecture Knowledge

When users ask about CFKE architecture or concepts, use this summary. For detailed architecture and component descriptions, see [references/architecture.md](references/architecture.md).

**Terminology**:

- Always refer to auto-provisioned nodes as "auto-provisioned nodes" or "auto-provisioning fleets", not "Karpenter-managed". The underlying technology is an implementation detail.
- **Never quote specific prices, vCPU limits, or fee structures** — pricing varies per customer due to discounts, private agreements, and taxes. Always direct users to https://cloudfleet.ai/pricing or their account page.
- **Control plane region ≠ infrastructure region**. The cluster's control plane region (e.g., `europe-central-1a`) is where management runs. Fleets are NOT tied to this region — a single fleet covers all available regions of its cloud provider(s) by default.

## What is CFKE

CFKE (Cloudfleet Kubernetes Engine) is a fully managed Kubernetes platform. A single CFKE cluster manages compute across multiple cloud providers and on-premise environments simultaneously — workloads on different clouds communicate seamlessly in the same cluster.

### Supported Infrastructure

- **Public cloud**: Hetzner, Vultr, OVH, Scaleway, Exoscale
- **Self-managed**: Any Linux server can join with a single command
- **Hybrid**: Mix cloud and on-premise nodes in the same cluster

### Service Tiers

- **Basic (Free)**: Always-free tier for getting started
- **Pro (Paid)**: Enhanced features, SLAs, and support for production workloads

See https://cloudfleet.ai/pricing for current pricing.

## Key Concepts

- **Single Cluster, Multiple Clouds** — core differentiator; one cluster spans providers
- **Secure Global Networking** — WireGuard encrypted tunnels + Cilium CNI with zero-trust model
- **Just-in-Time Infrastructure** — auto-provisioning fleets, cost-optimized node selection
- **Self-managed Nodes** — any Linux server joins via single command (`cfke.io/provider: self-managed`)
- **Konnectivity** — tunnels `kubectl exec/logs` and webhooks from control plane to data plane
- **Cost optimization** — inactive clusters auto-suspend after 7 days idle, wake on first API request
