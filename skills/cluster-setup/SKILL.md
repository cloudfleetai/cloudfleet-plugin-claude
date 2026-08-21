---
name: cluster-setup
description: This skill should be used when the user wants to create a CFKE cluster, set up fleets, connect cloud providers (AWS, GCP, Hetzner), add self-managed nodes, or get a new Cloudfleet environment running.
user-invocable: false
---

# Cluster Setup

Guide users through the full process of getting a Cloudfleet cluster up and running — from creating the cluster through connecting cloud accounts to deploying a first workload. See [references/cluster-setup.md](references/cluster-setup.md) for detailed CLI commands and all provider examples.

## Setup Flow

1. **Create a cluster** — the managed Kubernetes control plane
2. **Create a fleet** — connect a cloud provider account so nodes can be auto-provisioned
3. **Add self-managed nodes** (optional) — join any Linux server to the cluster
4. **Configure kubectl** — set up local access
5. **Deploy a workload** — verify everything works end-to-end

## 1. Create a Cluster

First, fetch available options from the user's organization:

```bash
cloudfleet organization describe
```

This returns the available `regions`, `versions`, and `cluster_tiers` under the `quota` field. Use these values — do NOT hardcode regions or versions.

A payment method is required at signup, so every tier is billable from the start. If a tier the user asked for is not listed in `quota.cluster_tiers`, tell them rather than silently picking another one.

Then create the cluster:

```bash
cloudfleet clusters create --name <name> --tier <tier> --region <region>
```

Named flags cover top-level scalar fields. For nested fields (`features`, `networking`) pass a document instead, with `-f -` for stdin or `-f <file>`:

```bash
cloudfleet clusters create -f - <<EOF
name: <name>
tier: <tier from quota.cluster_tiers>
region: <region from quota.regions>
version_channel: <version id from quota.versions>
EOF
```

Optional at create time, all Pro/Enterprise only: `release_channel` (`rapid`/`stable`/`extended`, how fast the control plane adopts new versions), `features` (GPU sharing via time-slicing or MPS), and `networking` (custom pod/service CIDRs, IPv4/IPv6 dual-stack). `networking` is immutable once the cluster exists. See [references/cluster-setup.md](references/cluster-setup.md).

Control plane provisions in 2-3 minutes. The cluster starts with zero worker nodes — they provision on-demand when workloads are scheduled.

**Important**: The control plane region is where the management layer runs. It does NOT restrict where worker nodes can be provisioned — fleets cover all available regions of their cloud providers by default.

## 2. Create a Fleet

A fleet connects a cloud provider account so CFKE can auto-provision nodes. At least one provider must be configured for auto-provisioning to work.

Wait for the cluster to reach status `deployed` first. Creating a fleet while the cluster is still provisioning returns 409.

**Fleets are not region-scoped** — a single Hetzner fleet can provision nodes in any Hetzner region (nbg1, fsn1, hel1, ash, sin, etc.). The same applies to AWS and GCP. Use node labels (`topology.kubernetes.io/region`) and placement constraints to control where workloads land.

```bash
cloudfleet clusters fleets create <cluster-id> -f - <<EOF
{
  "id": "my-fleet",
  "hetzner": { "enabled": true, "apiKey": "<hetzner-api-token>" },
  "limits": { "cpu": 24 }
}
EOF
```

Every provider block needs `"enabled": true` alongside its credentials.

Auto-provisioning providers: Hetzner (API token), AWS (IAM role ARN), GCP (project ID). These are the only three in the public fleet schema; Upcloud, Exoscale, Scaleway, and OVH are in private preview and need Cloudfleet support to enable. For anything else, join nodes as self-managed. A single fleet can span multiple providers. See [references/cluster-setup.md](references/cluster-setup.md) for all provider examples.

**Updates replace the whole resource.** `clusters update` and `fleets update` are full PUTs: any field left out resets to its default, and a flags-only update is rejected client-side. Read the current state, edit it, pipe it back. See [references/cluster-setup.md](references/cluster-setup.md).

## 3. Add Self-Managed Nodes (Optional)

Any Ubuntu 22.04/24.04 Linux server can join the cluster — bare metal, VMs, edge devices:

```bash
cloudfleet clusters add-self-managed-node <cluster-id> \
  --host <ip-address> \
  --region <datacenter-region> \
  --zone <datacenter-zone>
```

For GPU nodes, add `--install-nvidia-drivers`.

## 4. Configure kubectl

```bash
cloudfleet clusters kubeconfig <cluster-id>
```

## 5. Deploy and Verify

Deploy a test workload to trigger node provisioning:

```bash
kubectl create deployment nginx --image=nginx:latest --replicas=2
kubectl get pods -w
```

Pods transition Pending → ContainerCreating → Running as the auto-provisioner creates nodes. This confirms the full setup — cluster, fleet, and auto-provisioning — is working.
