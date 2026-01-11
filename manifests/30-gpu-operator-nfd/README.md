# NVIDIA GPU Operator Configuration

This directory contains the ClusterPolicy configuration for the NVIDIA GPU Operator.

## Operator Defaults

The `clusterpolicy.yaml` intentionally does NOT specify driver versions - it uses the operator's built-in defaults to ensure compatibility with the OpenShift cluster.

The GPU Operator automatically:
- Selects the appropriate GPU driver version for your kernel
- Uses OpenShift Driver Toolkit (DTK) to build drivers when needed
- Manages driver upgrades through its `upgradePolicy`

## MOFED Dependency Enforcement

The GPU operator requires MOFED drivers to be fully loaded for proper RDMA functionality. This directory includes a **PreSync hook** (`00-wait-for-mofed-presync.yaml`) that enforces the dependency:

### How It Works

1. **ArgoCD Sync Wave 4** - This Application is configured to deploy in Wave 4
2. **PreSync Hook Execution** - Before applying any manifests, ArgoCD executes the hook Job:
   - Waits for NicClusterPolicy to exist
   - Waits for MOFED DaemonSet creation
   - Waits for MOFED pods to be Running
   - **Waits for all MOFED pods to be fully Ready (2/2 containers)**
3. **ClusterPolicy Application** - Only after the hook succeeds does ArgoCD apply the ClusterPolicy

### Benefits

- ✅ **Guaranteed Ordering** - GPU driver NEVER deploys before MOFED is ready
- ✅ **Hands-Off Automation** - No manual intervention required
- ✅ **Failure Handling** - Hook retries up to 30 times with backoff
- ✅ **Clear Feedback** - Job logs show detailed progress of MOFED readiness

### Troubleshooting

If the PreSync hook fails:
```bash
# Check hook Job status
oc get jobs -n nvidia-gpu-operator wait-for-mofed-before-gpu

# View hook logs
oc logs -n nvidia-gpu-operator job/wait-for-mofed-before-gpu

# Check MOFED pod status
oc get pods -n nvidia-network-operator -l nvidia.com/ofed-driver
```

## Configuration

Key settings in `clusterpolicy.yaml`:
- **RDMA Support**: `driver.rdma.enabled: true` with `useHostMofed: false` (uses NVIDIA Network Operator's OFED)
- **Auto Upgrade**: Enabled with rolling updates
- **OCP Driver Toolkit**: Enabled for automatic driver compilation
- **Node Affinity**: Prevents driver deployment on master/control-plane nodes
- **Tolerations**: Configured for GPU nodes with `benchmark.llm-d.ai/test-gpu-amd64` taint

**IMPORTANT:** Do not pin driver versions unless absolutely necessary. Let the operator manage versions.
