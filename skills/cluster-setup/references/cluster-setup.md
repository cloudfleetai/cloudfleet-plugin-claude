# Cluster Setup — Detailed Reference

## Cluster Creation

### Fetch Available Options

Always fetch available regions, versions, and tiers from the user's organization before creating a cluster:

```bash
cloudfleet organization describe
```

The `quota` field contains `regions`, `versions` (with `id` and `label`), and `cluster_tiers`. Use these values — never hardcode them.

### CLI Command

```bash
cloudfleet clusters create <<EOF
{
  "name": "production-cluster",
  "tier": "<tier from quota.cluster_tiers>",
  "region": "<region from quota.regions>",
  "version_channel": "<version id from quota.versions>"
}
EOF
```

### Parameters

| Field             | Required | Description                                                                                      |
| ----------------- | -------- | ------------------------------------------------------------------------------------------------ |
| `name`            | Yes      | Cluster name (1-63 chars)                                                                        |
| `tier`            | Yes      | From `quota.cluster_tiers`                                                                       |
| `region`          | No       | Control plane region from `quota.regions`. Does NOT restrict where worker nodes are provisioned. |
| `version_channel` | No       | From `quota.versions` (use the `id` field)                                                       |

### Other Cluster Commands

```bash
cloudfleet clusters list                    # List all clusters
cloudfleet clusters describe <cluster-id>   # Get cluster details
cloudfleet clusters update <cluster-id>     # Update cluster
cloudfleet clusters delete <cluster-id>     # Delete cluster
```

## Fleet Creation

A fleet connects one or more cloud provider accounts to enable auto-provisioned nodes.

### Hetzner

```bash
cloudfleet clusters fleets create <cluster-id> <<EOF
{
  "id": "hetzner-fleet",
  "hetzner": {
    "apiKey": "<hetzner-cloud-api-token>"
  },
  "limits": { "cpu": 24 }
}
EOF
```

Generate API token with "Read & Write" permissions in Hetzner Cloud Console → Security → API Tokens.

### AWS

```bash
cloudfleet clusters fleets create <cluster-id> <<EOF
{
  "id": "aws-fleet",
  "aws": {
    "controllerRoleArn": "arn:aws:iam::123456789012:role/cloudfleet-controller-role"
  },
  "limits": { "cpu": 48 }
}
EOF
```

AWS uses IAM Workload Identity Federation (credential-less access). You need to:

1. Create an IAM role with the required trust policy and permissions
2. Cloudfleet provides a Terraform module to automate IAM and VPC setup
3. Pass the resulting role ARN as `controllerRoleArn`

### GCP

```bash
cloudfleet clusters fleets create <cluster-id> <<EOF
{
  "id": "gcp-fleet",
  "gcp": {
    "project": "my-gcp-project-id"
  },
  "limits": { "cpu": 48 }
}
EOF
```

Requires:

- Grant IAM roles to Cloudfleet-managed service account
- Default VPC with automatic subnet creation in the GCP project

### Multi-Provider Fleet

A single fleet can include multiple providers:

```bash
cloudfleet clusters fleets create <cluster-id> <<EOF
{
  "id": "multi-cloud",
  "hetzner": { "apiKey": "<token>" },
  "aws": { "controllerRoleArn": "arn:aws:iam::123456789012:role/cf-role" },
  "limits": { "cpu": 100 }
}
EOF
```

### Fleet Limits

The `limits.cpu` field sets the maximum total vCPU count across all nodes in the fleet. This prevents runaway costs.

### Other Fleet Commands

```bash
cloudfleet clusters fleets list <cluster-id>        # List fleets
cloudfleet clusters fleets describe <cluster-id> <fleet-id>  # Fleet details
cloudfleet clusters fleets update <cluster-id> <fleet-id>    # Update fleet
cloudfleet clusters fleets delete <cluster-id> <fleet-id>    # Delete fleet
```

## Self-Managed Nodes

### Requirements

- Ubuntu 22.04 or 24.04
- SSH access with root privileges
- Internet egress (TCP to _:443, UDP from :41641 to _._, UDP to _:3478)
- Can operate behind NAT without public IP

### CLI Command

```bash
cloudfleet clusters add-self-managed-node <cluster-id> \
  --host <ip-address> \
  --ssh-username root \
  --ssh-key ~/.ssh/id_rsa \
  --ssh-port 22 \
  --region datacenter-1 \
  --zone rack-a
```

| Flag                       | Default   | Description                 |
| -------------------------- | --------- | --------------------------- |
| `--host`                   | Required  | SSH host IP                 |
| `--ssh-username`           | `root`    | SSH user                    |
| `--ssh-key`                | SSH agent | Path to private key         |
| `--ssh-port`               | `22`      | SSH port                    |
| `--region`                 | Required  | Region label for scheduling |
| `--zone`                   | Required  | Zone label for scheduling   |
| `--install-nvidia-drivers` | false     | Install NVIDIA GPU drivers  |

The `--region` and `--zone` values become Kubernetes node labels (`topology.kubernetes.io/region` and `topology.kubernetes.io/zone`) for workload scheduling.

### Terraform (Cloud-init)

For automated provisioning of new VMs:

```hcl
resource "cloudfleet_cfke_node_join_information" "node" {
  cluster_id = data.cloudfleet_cfke_cluster.cluster.id
  region     = "datacenter-1"
  zone       = "rack-a"
  node_labels = {
    "cfke.io/provider" = "custom"
  }
}
```

### Terraform (SSH — existing machines)

```hcl
resource "cloudfleet_cfke_self_managed_node" "server" {
  cluster_id = cloudfleet_cfke_cluster.example.id
  region     = "datacenter-1"
  zone       = "rack-a"
  ssh {
    host             = "192.168.1.100"
    user             = "ubuntu"
    private_key_path = "~/.ssh/id_rsa"
  }
}
```

### GPU Nodes

Add `--install-nvidia-drivers` to install NVIDIA drivers from Ubuntu's repository and configure the NVIDIA container runtime. Add GPU labels for scheduling:

```
"cfke.io/accelerator-name" = "V100"
```

### Node Removal

On the target machine:

```bash
sudo apt remove -y kubelet
sudo rm -rf /etc/kubernetes/ /var/lib/kubelet/
```

From the cluster:

```bash
kubectl delete node <node-name>
```
