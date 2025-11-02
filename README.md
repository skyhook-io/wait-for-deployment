# Kubernetes Wait for Deployment

Waits for Kubernetes workloads to become ready with automatic Argo Rollout delegation.

## Features

- ⏳ **Universal workload support** - Deployments, StatefulSets, DaemonSets, and Rollouts
- 🔄 **Automatic Rollout delegation** - Seamlessly handles Argo Rollouts via `wait-for-rollout`
- 📊 **Batch processing** - Wait for multiple workloads from kustomize-inspect output
- 🎯 **Smart namespace handling** - Automatic namespace resolution
- 📝 **Status reporting** - Detailed workload status after waiting

## Usage

```yaml
- name: Wait for deployments
  uses: skyhook-io/wait-for-deployment@v1
  with:
    namespace: production
    workloads_json: '[{"kind":"Deployment","name":"api"},{"kind":"StatefulSet","name":"db"}]'
    timeout: 5m
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `namespace` | Default namespace for workloads | ✅ | - |
| `workloads_json` | JSON array of workloads: `[{kind,name,namespace?}]` | ✅ | - |
| `timeout` | Wait timeout (e.g., 300s, 5m) | ❌ | `300s` |
| `show_status` | Show kubectl status after waiting | ❌ | `true` |
| `verify_only` | Pass-through for Rollout verification | ❌ | `false` |

## Supported Workload Types

- **Deployment** - Standard Kubernetes deployments
- **StatefulSet** - Stateful applications
- **DaemonSet** - Node-level workloads
- **Rollout** - Argo Rollouts (delegated to `wait-for-rollout@v1`)

## Examples

### Basic Deployment Wait
```yaml
- name: Deploy and wait
  run: kubectl apply -f manifests/

- name: Wait for deployment
  uses: skyhook-io/wait-for-deployment@v1
  with:
    namespace: production
    workloads_json: '[{"kind":"Deployment","name":"frontend"}]'
```

### Multiple Workloads
```yaml
- name: Wait for all services
  uses: skyhook-io/wait-for-deployment@v1
  with:
    namespace: default
    workloads_json: |
      [
        {"kind":"Deployment","name":"api"},
        {"kind":"Deployment","name":"worker"},
        {"kind":"StatefulSet","name":"database"}
      ]
    timeout: 10m
```

### With Kustomize Inspect
```yaml
- name: Inspect manifests
  id: inspect
  uses: skyhook-io/kustomize-inspect@v1
  with:
    overlay_path: overlays/production

- name: Wait for workloads
  uses: skyhook-io/wait-for-deployment@v1
  with:
    namespace: production
    workloads_json: ${{ steps.inspect.outputs.workloads }}
```

### Mixed Standard and Rollout Workloads
```yaml
- name: Wait for mixed workloads
  uses: skyhook-io/wait-for-deployment@v1
  with:
    namespace: production
    workloads_json: |
      [
        {"kind":"Deployment","name":"frontend"},
        {"kind":"Rollout","name":"api"},
        {"kind":"StatefulSet","name":"cache"}
      ]
```

### With Custom Namespaces
```yaml
- name: Wait across namespaces
  uses: skyhook-io/wait-for-deployment@v1
  with:
    namespace: default  # Fallback for workloads without namespace
    workloads_json: |
      [
        {"kind":"Deployment","name":"api","namespace":"backend"},
        {"kind":"Deployment","name":"web","namespace":"frontend"},
        {"kind":"StatefulSet","name":"db"}
      ]
```

## How It Works

1. **Parses workload list** - Processes the JSON array of workloads
2. **Separates by type** - Identifies standard K8s workloads vs Argo Rollouts
3. **Delegates Rollouts** - Automatically uses `wait-for-rollout@v1` for Rollout resources
4. **Waits for standard workloads** - Uses `kubectl rollout status` for Deployments, StatefulSets, DaemonSets
5. **Reports status** - Shows final status for all workloads

## Rollout Delegation

When a Rollout is detected in the workload list:
- Automatically delegates to `skyhook-io/wait-for-rollout@v1`
- Passes through timeout and verify_only parameters
- Supports only one Rollout per action invocation (use matrix for multiple)

## Error Handling

The action will fail if:
- Any Deployment fails to become ready
- Timeout is exceeded
- Multiple Rollouts are provided (use matrix strategy instead)
- kubectl connection fails

Note: StatefulSet and DaemonSet failures are non-blocking by default.

## Complete Deployment Flow

```yaml
jobs:
  deploy:
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: skyhook-io/cloud-login@v1
        with:
          provider: aws
          region: us-east-1
          cluster: production
      
      - name: Build manifests
        id: build
        run: |
          kustomize build overlays/production > manifests.yaml
          kubectl apply -f manifests.yaml
      
      - name: Get workloads
        id: inspect
        uses: skyhook-io/kustomize-inspect@v1
        with:
          overlay_path: overlays/production
      
      - name: Wait for all workloads
        uses: skyhook-io/wait-for-deployment@v1
        with:
          namespace: production
          workloads_json: ${{ steps.inspect.outputs.workloads }}
          timeout: 10m
      
      - name: Run tests
        if: success()
        run: ./scripts/smoke-test.sh
```

## Prerequisites

- Kubernetes cluster access (configured kubectl)
- For Rollouts: Argo Rollouts installed in cluster
- Appropriate RBAC permissions

## Notes

- Requires kubectl context to be configured
- Use with `cloud-login` action for authentication
- For multiple Rollouts, use matrix strategy or sequential calls
- Timeout applies to each workload wait operation