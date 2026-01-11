# MOFED Readiness Gate (Wave 3)

This directory contains a Job that waits for NVIDIA MOFED drivers to be fully ready before allowing the GPU Operator (Wave 4) to deploy.

## Purpose

The GPU Operator requires MOFED drivers to be fully loaded for proper RDMA functionality. This Job ensures:
1. NicClusterPolicy exists
2. MOFED DaemonSet is created
3. All MOFED pods are Running and Ready (2/2 containers)

## Known Limitation

**ArgoCD Sync Wave Timing Issue:**

ArgoCD sync waves control the ORDER of applying resources, but Applications with `automated: selfHeal: true` may start syncing in parallel. This can cause Wave 4 (GPU Operator) to begin deployment while Wave 3 (MOFED readiness) is still in progress.

**Why this happens:**
- The root Application creates all child Applications simultaneously
- Each Application syncs according to its wave number
- ArgoCD proceeds to the next wave once resources are "Synced" (applied), not necessarily "Healthy" (ready)
- Jobs are considered "Synced" immediately upon creation, before they complete

**Current behavior:**
- Wave 2: NicClusterPolicy applied
- Wave 3: wait-for-mofed Job created (considered "Synced")
- Wave 4: GPU Operator Application triggers (doesn't wait for Wave 3 Job completion)
- **BUT:** Wave 4 has its own PreSync hook that waits for MOFED readiness

**Mitigation - IMPLEMENTED:**
✅ The GPU Operator Application (`30-gpu-operator-nfd`) now includes a **PreSync hook** (`00-wait-for-mofed-presync.yaml`) that:
- Runs before any GPU operator manifests are applied
- Waits for all MOFED pods to be fully Ready (2/2 containers)
- Ensures the ClusterPolicy is NEVER applied until MOFED is ready
- Provides clear progress feedback and error handling

This solves the race condition while maintaining hands-off automation.

**How the two Jobs work together:**
1. **Wave 3 Job** (`wait-for-mofed.yaml` - this directory):
   - General readiness gate for MOFED drivers
   - Validates MOFED before Wave 4 Applications are created
   - ArgoCD considers Wave 3 "Synced" when Job is created (not completed)

2. **Wave 4 PreSync Hook** (`30-gpu-operator-nfd/00-wait-for-mofed-presync.yaml`):
   - **ArgoCD-enforced dependency** - runs before ClusterPolicy is applied
   - Guarantees GPU driver never deploys before MOFED ready
   - Blocks until all MOFED pods are 2/2 Running and Ready

**Recommendation:**
The PreSync hook in Wave 4 provides guaranteed ordering. No manual intervention required.
