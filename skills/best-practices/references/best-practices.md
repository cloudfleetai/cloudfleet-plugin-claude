# CFKE Best Practices — Detailed Reference

## Resource Requests

```yaml
resources:
    requests:
        cpu: "100m"
        memory: "128Mi"
    limits:
        memory: "512Mi"
```

Do NOT set CPU limits. CPU limits cause unnecessary throttling — the kernel CFS quota can throttle a container even when the node has idle CPU. Set CPU requests for scheduling, but let containers use available CPU freely.

Without resource requests:

- Clusters may fail to scale correctly
- Nodes can become saturated
- Auto-provisioner cannot make informed decisions about node sizing

## Load Balancer Services

```yaml
apiVersion: v1
kind: Service
metadata:
    name: my-service
spec:
    type: LoadBalancer
    externalTrafficPolicy: Local # Required for CFKE!
    ports:
        - port: 80
          targetPort: 8080
    selector:
        app: my-app
```

The default 'Cluster' mode causes serious problems on CFKE:

- Load Balancers provisioned in EVERY cloud/region where nodes exist
- Extra costs from unnecessary LB instances
- Traffic routed through encrypted in-cluster network with extra hops
- Inefficient routing where traffic may land on nodes without target pods

Benefits of Local policy:

- Load Balancers only provisioned where Pods actually run
- Direct traffic path to pods (no extra hops)
- Preserves client source IP
- Reduced costs and latency

## CFKE Node Labels

| Label                              | Description       | Example values                                                  |
| ---------------------------------- | ----------------- | --------------------------------------------------------------- |
| `cfke.io/provider`                 | Cloud provider    | aws, gcp, hetzner, vultr, ovh, scaleway, exoscale, self-managed |
| `cfke.io/accelerator-name`         | GPU type          | nvidia-a100, nvidia-v100                                        |
| `topology.kubernetes.io/zone`      | Availability zone | eu-central-1a, fsn1                                             |
| `topology.kubernetes.io/region`    | Region            | eu-central, us-east                                             |
| `node.kubernetes.io/instance-type` | Instance type     | cx32, m5.large                                                  |

## Node Selector Example

```yaml
spec:
    nodeSelector:
        cfke.io/provider: hetzner
        topology.kubernetes.io/region: eu-central
```

## Node Affinity (Multiple Providers)

```yaml
spec:
    affinity:
        nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                    - matchExpressions:
                          - key: cfke.io/provider
                            operator: In
                            values: ["aws", "hetzner"]
```

## Topology Spread (High Availability)

```yaml
spec:
    topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
              matchLabels:
                  app: my-app
        - maxSkew: 1
          topologyKey: cfke.io/provider
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
              matchLabels:
                  app: my-app
```

## Use Cases for Placement Constraints

1. **GPU workloads**: Select nodes with specific GPU hardware using `cfke.io/accelerator-name`
2. **Data locality**: Place pods near data sources in specific regions
3. **Compliance**: Keep workloads in specific regions for regulatory requirements
4. **Cost optimization**: Use specific instance types or providers
5. **High availability**: Spread across zones and providers

## Production Readiness

- **PodDisruptionBudgets**: Define PDBs for critical workloads to ensure availability during node maintenance
- **Health Checks**: Configure liveness and readiness probes for all containers
- **Resource-intensive workloads**: Lock CPU-bound pods to dedicated instances (check node labels)
- **Pod distribution**: Use topologySpreadConstraints for high availability across zones/providers

## Security

- Avoid privileged containers and running as root when possible
- Use versioned image tags instead of `:latest`
- Implement NetworkPolicies for zero-trust security
- Enable TLS on all Ingresses for public-facing services

## GPU/ML Workloads

- Use `cfke.io/accelerator-name` label to select specific GPU types
- Set appropriate resource requests for GPU (`nvidia.com/gpu`)
- Consider spot instances for batch ML training jobs

## Private Container Registries

CFKE can pull from AWS ECR and GCP Artifact Registry without needing imagePullSecrets, using Cloudfleet's built-in authentication. CFCR (Cloudfleet Container Registry) images also work without pull secrets within the same organization.
