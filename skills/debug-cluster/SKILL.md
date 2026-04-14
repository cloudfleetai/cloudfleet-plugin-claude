---
name: debug-cluster
description: This skill should be used when the user mentions "debug", "troubleshoot", "why is", "what's wrong", or describes CFKE cluster issues like node failures, pod crashes, or networking problems.
user-invocable: true
argument-hint: "[cluster-name-or-id]"
---

# Debug CFKE Cluster

Guided debugging workflow for Cloudfleet Kubernetes Engine clusters.

kubectl enabled: **${user_config.enable_kubectl}**

## Important Notes

- `kubectl` is a separate tool from the Cloudfleet CLI — there is no `cloudfleet kubectl` command.
- If the user specifies a `--profile`, pass it to all `cloudfleet` CLI commands.
- The kubeconfig context name format is `{cluster-id}/{profile-name}`. Do NOT guess — run `kubectl config get-contexts` if unsure.

## Prerequisites

- Cloudfleet CLI installed and authenticated

## Execution Steps

### 1. Identify the cluster

If a cluster name or ID was provided as `$ARGUMENTS`, use it. Otherwise, run `cloudfleet clusters list` (with `--profile` if specified) to show available clusters and ask the user which one to debug. Save the cluster ID as CLUSTER_ID.

### 2. Configure kubectl access (if kubectl enabled)

Use the `cfke-kubeconfig` helper to get an isolated kubeconfig:

```bash
KUBECONFIG=$(cfke-kubeconfig get $CLUSTER_ID)
```

Use `kubectl --kubeconfig $KUBECONFIG` for ALL subsequent kubectl commands. **Never use bare `kubectl`** — the user's active context may point to a different cluster.

### 3. Gather cluster info

**If kubectl is enabled:**

```bash
kubectl --kubeconfig $KUBECONFIG get nodes -o wide
kubectl --kubeconfig $KUBECONFIG get pods --all-namespaces --field-selector=status.phase!=Running
kubectl --kubeconfig $KUBECONFIG get events --all-namespaces --sort-by='.lastTimestamp' | tail -30
```

**If kubectl is not enabled**, use the Cloudfleet MCP `clusters_query` tool (read-only) to check:

- `/api/v1/nodes` — node status
- `/api/v1/pods?fieldSelector=status.phase!=Running` — failing pods
- `/api/v1/events?limit=30` — recent events

### 4. Systematic diagnosis

For comprehensive parallel diagnostics, consider delegating to the `cluster-diagnostics` agent.

**With kubectl:**

Nodes:

```bash
kubectl --kubeconfig $KUBECONFIG describe nodes | grep -A5 "Conditions:"
kubectl --kubeconfig $KUBECONFIG top nodes
```

System pods:

```bash
kubectl --kubeconfig $KUBECONFIG get pods -n kube-system -o wide
```

Resource usage:

```bash
kubectl --kubeconfig $KUBECONFIG top pods --all-namespaces --sort-by=memory | head -20
```

**With MCP only:** Query `/api/v1/namespaces/kube-system/pods` and `/api/v1/nodes` for status details. Note that `kubectl top` equivalents are not available via MCP.

### 5. Targeted investigation

Based on findings, drill into specific areas:

- **Pod failures**: Check logs with `kubectl logs` (requires kubectl), describe pod for events
- **Node issues**: Check node conditions, Cilium status
- **Networking**: Check Cilium pods, CoreDNS, Services/Endpoints
- **Scheduling**: Check resource requests vs available capacity, taints/tolerations

### 6. Summary

Present findings with:

- Root cause (or most likely candidates)
- Recommended remediation steps
- Any Cloudfleet-specific guidance (e.g., fleet adjustments, version upgrades)

If the issue appears to be on the platform side (e.g., control plane unreachable, API errors), remind the user to check https://status.cloudfleet.ai/ for ongoing incidents and, if needed, open a support ticket at https://console.cloudfleet.ai/support.
