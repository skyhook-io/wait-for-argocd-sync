# Wait for ArgoCD Sync

Wait for ArgoCD-managed workloads to sync and become ready by monitoring resources directly in the target cluster.

## Features

- **Cluster-level monitoring** - Works regardless of where ArgoCD Application CRD lives
- **Multiple detection strategies** - deployment-id, version label, or generation change
- **Fallback handling** - Detects if sync already completed before action started
- **No false positives** - Confirms actual change before proceeding

## Usage

```yaml
# Recommended: with expected deployment-id (most reliable)
- name: Wait for ArgoCD sync
  uses: skyhook-io/wait-for-argocd-sync@v1
  with:
    app_name: my-app
    namespace: production
    expected_deployment_id: ${{ github.run_id }}
    timeout: 5m

# Alternative: with expected version
- name: Wait for ArgoCD sync
  uses: skyhook-io/wait-for-argocd-sync@v1
  with:
    app_name: my-app
    namespace: production
    expected_version: ${{ inputs.tag }}
    timeout: 5m
```

## Detection Strategies

The action uses different strategies based on what inputs are provided:

| Inputs Provided | Strategy | How It Works |
|-----------------|----------|--------------|
| `expected_deployment_id` | **Exact match** | Wait for deployment-id annotation to equal expected value |
| `expected_version` (differs from current) | **Version match** | Wait for version label to equal expected value + confirm change |
| `expected_version` (same as current) | **Change detection** | Wait for deployment-id or generation to change |
| None | **Fallback** | Wait for change, or after 180s check if already synced |

### Why deployment-id is Most Reliable

```
Scenario: Config-only change (version stays same)

Without deployment-id:
  1. CI commits config change
  2. ArgoCD syncs (version unchanged)
  3. Action can't tell if sync happened → waits 180s fallback

With deployment-id:
  1. CI commits config change with deployment-id=12345
  2. ArgoCD syncs (deployment-id=12345)
  3. Action sees match → proceeds immediately ✅
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `app_name` | Application name (label selector) | Yes | - |
| `namespace` | Target namespace | Yes | - |
| `expected_deployment_id` | Expected `deployment-id` annotation | No | - |
| `expected_version` | Expected `app.kubernetes.io/version` label | No | - |
| `selector` | Custom label selector (overrides app_name) | No | - |
| `timeout` | Total timeout (e.g., 300s, 5m) | No | `300s` |
| `fallback_timeout` | Time before fallback check (e.g., 180s, 3m) | No | `180s` |
| `workload_types` | Workload types to wait for | No | `deployment,statefulset,daemonset` |

## Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `workloads_found` | Number of workloads matching selector | `2` |
| `workloads_ready` | Number ready after rollout | `2` |
| `version_matched` | Version match result | `true`, `false`, `skipped` |
| `deployment_id_matched` | Deployment ID match result | `true`, `false`, `skipped` |
| `sync_detection_method` | How sync was detected | `expected_match`, `change_detected`, `already_synced` |
| `status` | Final status | `Ready`, `Failed`, `Timeout` |
| `message` | Status message | `All 2 workloads are ready` |

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 0: Capture initial state                                   │
│   - Record deployment-id, generation, version for each workload│
│   - Determine detection strategy based on inputs               │
├─────────────────────────────────────────────────────────────────┤
│ Step 1: Wait for sync indication                                │
│                                                                 │
│   Strategy: expected_deployment_id                              │
│     → Wait for deployment-id == expected (immediate if match)  │
│                                                                 │
│   Strategy: expected_version                                    │
│     → Wait for version == expected + confirm change            │
│                                                                 │
│   Strategy: change detection                                    │
│     → Wait for deployment-id or generation to change           │
│                                                                 │
│   Strategy: fallback (no expected values)                       │
│     → Wait for change, OR after 180s if all rollouts complete  │
│       assume sync already happened                              │
├─────────────────────────────────────────────────────────────────┤
│ Step 2: Wait for rollouts                                       │
│   - kubectl rollout status for each workload                   │
└─────────────────────────────────────────────────────────────────┘
```

## Examples

### With Skyhook Deploy Actions (Recommended)

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

### Without Expected Values (Fallback Mode)

Waits up to 180s for changes, then checks if rollouts are complete:

```yaml
- name: Wait for sync
  uses: skyhook-io/wait-for-argocd-sync@v1
  with:
    app_name: backend-api
    namespace: production
    timeout: 10m
    fallback_timeout: 3m  # ArgoCD default sync interval
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
          expected_deployment_id: ${{ github.run_id }}
          expected_version: ${{ needs.build.outputs.tag }}
          timeout: 10m

      - name: Run smoke tests
        run: ./scripts/smoke-test.sh
```

## Labels and Annotations

The action looks for these on workloads:

**Labels:**
- `app.kubernetes.io/name` - Used for workload discovery (selector)
- `app.kubernetes.io/version` - Compared against `expected_version`

**Annotations:**
- `deployment-id` - Compared against `expected_deployment_id`

These are set automatically by `skyhook-io/kustomize-deploy` and `skyhook-io/kustomize-edit`.

## Prerequisites

- Kubernetes cluster access (configured kubectl)
- ArgoCD managing the target workloads
- Workloads labeled with `app.kubernetes.io/name`

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
| ArgoCD awareness | None (race condition possible) | Full (deployment-id, version, fallback) |
| Use case | Direct kubectl apply | GitOps deployments |
