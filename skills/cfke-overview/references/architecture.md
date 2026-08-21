# CFKE Architecture — Detailed Reference

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────┐
│  Cloudfleet Managed Control Plane                        │
│  ┌──────────────┐ ┌─────────────┐ ┌────────────┐        │
│  │kube-apiserver│ │ controller- │ │   kube-    │        │
│  │              │ │ manager     │ │  scheduler │        │
│  └──────────────┘ └─────────────┘ └────────────┘        │
│  ┌─────────────┐ ┌──────────────────────────────┐        │
│  │    etcd      │ │ cloud-controller-manager     │        │
│  └─────────────┘ └──────────────────────────────┘        │
└──────────────────────────────────────────────────────────┘
                          │ WireGuard VPN
                          ▼
┌──────────────────────────────────────────────────────────┐
│  Your Infrastructure (any supported provider)            │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Worker Nodes                                      │  │
│  │  - kubelet, containerd                             │  │
│  │  - Cilium (CNI)                                    │  │
│  │  - CoreDNS                                         │  │
│  │  - Konnectivity Agent                              │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

## Key Concepts (Detailed)

### Single Cluster, Multiple Clouds

CFKE's core differentiator — unlike traditional multi-cloud solutions requiring separate Kubernetes clusters per cloud/region, CFKE provides a single managed cluster that simultaneously manages compute across different cloud providers and on-premise environments. Workloads on AWS, Hetzner, and self-managed infrastructure exist in the same cluster and communicate seamlessly.

### Secure Global Networking

CFKE uses WireGuard encrypted tunnels to create a secure overlay network connecting all nodes. Powered by Cilium CNI with zero-trust security model. Workloads across clouds/regions communicate in the same IP space, and NetworkPolicies work across the multi-cloud cluster.

### Cilium CNI

CFKE uses Cilium for container networking, providing eBPF-based networking, network policies, and observability.

### Just-in-Time Infrastructure

CFKE auto-provisions nodes through fleets. It provisions optimal instance types automatically, optimizes costs by selecting cheapest suitable nodes, scales dynamically, and supports spot instances, ARM, and GPU workloads.

### Self-managed Nodes

Any Linux server can join a CFKE cluster with a single command, enabling hybrid cloud, edge computing, and unsupported cloud providers. Nodes with `cfke.io/provider: self-managed` indicate hybrid/edge deployments.

### Konnectivity

Enables `kubectl exec`, `kubectl logs`, and webhook traffic from the control plane to reach pods on data plane nodes through the VPN tunnel.

## Auto-Provisioning Fleets

Fleets are configured via the Cloudfleet API or CLI (`cloudfleet clusters fleets`). Users define which cloud providers to use, resource limits (max vCPUs), and regions. CFKE handles the underlying node provisioning automatically — users do not manage NodePool or NodeClass resources directly.

## Load Balancers

CFKE's cloud-controller-manager creates load balancers natively on each cloud provider when you create a `Service` of type `LoadBalancer`. Configure behavior via annotations specific to each provider.

**Important**: Always use `externalTrafficPolicy: Local` — see best-practices for details.
