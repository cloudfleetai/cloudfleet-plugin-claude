---
name: cluster-info
description: This skill should be used when the user asks to "list clusters", "show clusters", "cluster status", "what clusters do I have", or wants a Cloudfleet cluster overview.
user-invocable: true
argument-hint: "[cluster-name-or-id]"
---

# Cluster Info

Retrieve and display information about your Cloudfleet clusters.

## Important Notes

- `kubectl` is a separate tool from the Cloudfleet CLI — there is no `cloudfleet kubectl` command.
- If the user specifies a `--profile`, pass it to all `cloudfleet` CLI commands.

## Execution Steps

### 1. List or fetch clusters

If `$ARGUMENTS` contains a specific cluster name or ID:

```bash
cloudfleet clusters describe <cluster-id>
```

Otherwise, list all clusters:

```bash
cloudfleet clusters list
```

Add `--profile <name>` if the user specified one.

### 2. Present information

Display the results in a clear table format including:

- Cluster name and ID
- Kubernetes version
- Cloud provider and region
- Node count
- Status (ready, provisioning, suspended, etc.)
- Tier

### 3. Offer next steps

Based on the cluster status, suggest relevant actions:

- If a cluster has issues → offer to debug with `/cloudfleet:debug-cluster`
- If the user wants to query resources → offer to help with kubectl commands
- If they need to scale → point to fleet configuration
