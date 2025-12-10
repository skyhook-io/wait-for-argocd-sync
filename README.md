# Wait for ArgoCD Sync

Wait for ArgoCD-managed workloads to sync and become ready by monitoring resources directly in the target cluster.

## Features

- **Cluster-level monitoring** - Works regardless of where ArgoCD Application CRD lives
- **Version tracking** - Wait for specific `app.kubernetes.io/version` label
- **Deployment ID tracking** - Wait for specific `deployment-id` annotation (GitHub run ID)
- **No false positives** - Matches on expected values before checking rollout status
- **Flexible selectors** - Use app name or custom label selectors

## Usage

```yaml
- name: Wait for ArgoCD sync
  uses: skyhook-io/wait-for-argocd-sync@v1
  with:
    app_name: my-app
    namespace: production
    expected_version: ${{ inputs.tag }}
    expected_deployment_id: ${{ github.run_id }}
    timeout: 5m
```

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Wait for expected metadata                              │
│   - Poll deployments with label app.kubernetes.io/name=<app>   │
│   - Wait until version OR deployment-id matches expected value │
│   - This confirms ArgoCD has synced the new manifests          │
├─────────────────────────────────────────────────────────────────┤
│ Step 2: Wait for rollouts                                       │
│   - kubectl rollout status for each workload                   │
│   - Track successes and failures                                │
├─────────────────────────────────────────────────────────────────┤
│ Step 3: Report results                                          │
│   - Output status and match info                                │
│   - Generate GitHub Actions summary                             │
└─────────────────────────────────────────────────────────────────┘
```

## Why Both Version AND Deployment ID?

| Change Type | Version Label | Deployment ID |
|-------------|---------------|---------------|
| Image change | ✅ Changes | ✅ Changes |
| Config-only change | ❌ Same | ✅ Changes |
| Env vars change | ❌ Same | ✅ Changes |

By checking **either** matches, we catch all deployment types:
- Image updates → version label changes
- Config updates → deployment-id annotation changes
- Both → both change

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `app_name` | Application name (label selector) | Yes | - |
| `namespace` | Target namespace | Yes | - |
| `expected_version` | Expected `app.kubernetes.io/version` value | No | - |
| `expected_deployment_id` | Expected `deployment-id` annotation | No | - |
| `selector` | Custom label selector (overrides app_name) | No | - |
| `timeout` | Total timeout (e.g., 300s, 5m) | No | `300s` |
| `workload_types` | Workload types to wait for | No | `deployment,statefulset,daemonset` |

## Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `workloads_found` | Number of workloads matching selector | `2` |
| `workloads_ready` | Number ready after rollout | `2` |
| `version_matched` | Version match result | `true`, `false`, `skipped` |
| `deployment_id_matched` | Deployment ID match result | `true`, `false`, `skipped` |
| `status` | Final status | `Ready`, `Failed`, `Timeout` |
| `message` | Status message | `All 2 workloads are ready` |

## Examples

### Standard Deployment (Recommended)

```yaml
- name: Deploy
  uses: skyhook-io/kustomize-deploy@v1
  id: deploy
  with:
    overlay_dir: deploy/overlays/${{ inputs.environment }}
    tag: ${{ inputs.tag }}

- name: Wait for ArgoCD sync
  uses: skyhook-io/wait-for-argocd-sync@v1
  with:
    app_name: my-service
    namespace: ${{ steps.deploy.outputs.namespace }}
    expected_version: ${{ inputs.tag }}
    expected_deployment_id: ${{ github.run_id }}
    timeout: 5m
```

### Image Change Only

```yaml
- name: Wait for new image
  uses: skyhook-io/wait-for-argocd-sync@v1
  with:
    app_name: backend-api
    namespace: production
    expected_version: main_2025-12-10_00
```

### Config Change Only

```yaml
- name: Wait for config update
  uses: skyhook-io/wait-for-argocd-sync@v1
  with:
    app_name: backend-api
    namespace: production
    expected_deployment_id: ${{ github.run_id }}
```

### No Expected Values (Legacy Mode)

If no expected values provided, falls back to finding workloads and waiting for rollout:

```yaml
- name: Wait for any rollout
  uses: skyhook-io/wait-for-argocd-sync@v1
  with:
    app_name: backend-api
    namespace: production
```

### Custom Label Selector

```yaml
- name: Wait by custom labels
  uses: skyhook-io/wait-for-argocd-sync@v1
  with:
    app_name: my-app
    namespace: production
    selector: "team=platform,component=api"
    expected_deployment_id: ${{ github.run_id }}
```

### Complete CI/CD Flow

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.build.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
      - name: Build and push image
        id: build
        run: |
          TAG="main_$(date +%Y-%m-%d)_${{ github.run_number }}"
          docker build -t myrepo/myapp:$TAG .
          docker push myrepo/myapp:$TAG
          echo "tag=$TAG" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup kubectl
        uses: skyhook-io/cloud-login@v1
        with:
          provider: gcp
          cluster: production

      - name: Deploy via GitOps
        uses: skyhook-io/kustomize-deploy@v1
        with:
          overlay_dir: deploy/overlays/prod
          tag: ${{ needs.build.outputs.tag }}

      - name: Wait for ArgoCD sync
        uses: skyhook-io/wait-for-argocd-sync@v1
        with:
          app_name: myapp
          namespace: prod
          expected_version: ${{ needs.build.outputs.tag }}
          expected_deployment_id: ${{ github.run_id }}
          timeout: 10m

      - name: Run smoke tests
        run: ./scripts/smoke-test.sh
```

## Label Discovery

The action finds workloads using the standard Kubernetes label:

```yaml
app.kubernetes.io/name: <app_name>
```

And checks for:
```yaml
# Label (set by kustomize-edit)
app.kubernetes.io/version: <expected_version>

# Annotation (set by kustomize-deploy)
deployment-id: <expected_deployment_id>
```

## Prerequisites

- Kubernetes cluster access (configured kubectl)
- ArgoCD managing the target workloads
- Workloads labeled with `app.kubernetes.io/name`
- For version matching: `app.kubernetes.io/version` label set by your deploy pipeline
- For deployment-id matching: `deployment-id` annotation set by your deploy pipeline

## Error Handling

| Scenario | Behavior |
|----------|----------|
| No workloads found | Warning, exits 0 |
| Expected values never match | Timeout, exits 1 |
| Rollout fails | Reports failed workloads, exits 1 |
| Overall timeout | Reports current state, exits 1 |

## Comparison with wait-for-deployment

| Feature | wait-for-deployment | wait-for-argocd-sync |
|---------|---------------------|----------------------|
| Input format | JSON workloads array | Label selector |
| ArgoCD awareness | None (race condition) | Version/ID matching |
| Use case | Direct kubectl apply | GitOps deployments |
