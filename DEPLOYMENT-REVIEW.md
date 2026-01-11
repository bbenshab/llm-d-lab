# Deployment Review - Manual Intervention Analysis

**Date**: 2026-01-09
**Deployment Attempt**: Full stack deployment from scratch
**Goal**: Completely hands-off GitOps deployment with ZERO manual intervention

---

## Complete List of Manual Interventions (Chronological)

### 0. Pre-Deployment Setup

**Manual Steps**:
1. Edit `envs/baremetal-lab/kustomization.yaml` - uncomment cleanup resource
2. Apply: `oc apply -k envs/baremetal-lab` or sync via ArgoCD
3. Monitor cleanup Job: `oc logs -n openshift-operators job/cleanup-operators -f`
4. Edit kustomization again - re-comment cleanup resource
5. Commit changes

**Why Manual**: No automatic cleanup triggering mechanism exists

---

### 1. Initial Deployment Trigger

**Manual Steps**:
1. Apply ArgoCD prerequisites: `oc apply -k overlays/baremetal`
2. Monitor auto-approval Job: `oc logs -n openshift-operators job/approve-gitops-installplan -f`
3. Wait for ArgoCD operator CSV to reach Succeeded state
4. Apply root GitOps Application: `oc apply -k envs/baremetal-lab`

**Why Manual**: No bootstrap mechanism to auto-deploy prerequisites and root app

---

### 2. NicClusterPolicy Version Validation Error

**Issue**: ArgoCD sync failed with admission webhook error:
```
admission webhook "vnicclusterpolicy.kb.io" denied the request:
NicClusterPolicy.mellanox.com "nic-cluster-policy" is invalid:
spec.ofedDriver.version: Invalid value: "": invalid OFED version
```

**Root Cause**:
- `manifests/28-nvidia-network-operator/kustomization.yaml` had the wrong resource enabled
- Was using static `nicclusterpolicy.yaml` with empty version field
- Dynamic version discovery job `00-version-discovery-job.yaml` was commented out

**Manual Interventions Required**:
1. Edit `manifests/28-nvidia-network-operator/kustomization.yaml`
2. Switch from `nicclusterpolicy.yaml` to `00-version-discovery-job.yaml`
3. Commit: `git add -A && git commit -m "..."`
4. Push: `git push origin master`
5. Manually apply Job: `oc apply -f manifests/28-nvidia-network-operator/00-version-discovery-job.yaml`
6. Monitor Job logs: `oc logs -n nvidia-network-operator job/discover-ofed-version -f`
7. Manually trigger ArgoCD refresh: `oc annotate application nvidia-network-operator ...`
8. Clear Application operation state: `oc patch application nvidia-network-operator ...`

**Why Manual**: Configuration error - wrong resource enabled in kustomization

**Fix Applied** (Commit: e257d39):
```yaml
# manifests/28-nvidia-network-operator/kustomization.yaml
resources:
  # Dynamic version discovery - PreSync hook creates NicClusterPolicy with operator default version
  - 00-version-discovery-job.yaml
  # Removed: - nicclusterpolicy.yaml
```

---

### 3. GPU PreSync Hook Missing RBAC Permissions

**Issue**: PreSync hook Job stuck in loop:
```
Waiting for NicClusterPolicy... (245 seconds left)
Waiting for NicClusterPolicy... (240 seconds left)
...
```

**Root Cause**:
- ClusterRole `gpu-mofed-waiter` was missing permissions to check for NicClusterPolicy
- Script could not execute: `oc get nicclusterpolicy nic-cluster-policy`
- Lacked API group `mellanox.com` and resource `nicclusterpolicies`

**Manual Interventions Required**:
1. Edit `manifests/30-gpu-operator-nfd/00-wait-for-mofed-presync.yaml`
2. Add mellanox.com API group permissions to ClusterRole
3. Commit: `git add -A && git commit -m "..."`
4. Push: `git push origin master`
5. Apply updated RBAC: `oc apply -f manifests/30-gpu-operator-nfd/00-wait-for-mofed-presync.yaml`
6. Delete stuck Job: `oc delete job wait-for-mofed-before-gpu -n nvidia-gpu-operator`
7. Monitor new Job: `oc logs -n nvidia-gpu-operator job/wait-for-mofed-before-gpu --tail=30`

**Why Manual**: Code defect - missing RBAC permissions in PreSync hook

**Fix Applied** (Commit: 730fda7):
```yaml
# manifests/30-gpu-operator-nfd/00-wait-for-mofed-presync.yaml
rules:
  # ... existing rules ...
  - apiGroups: ["mellanox.com"]
    resources: ["nicclusterpolicies"]
    verbs: ["get", "list"]
```

---

### 4. ArgoCD PreSync Hook Stuck in Finalizer State

**Issue**: ArgoCD Application stuck "OutOfSync" with message:
```
Running: waiting for completion of hook batch/Job/wait-for-mofed-before-gpu
```

**But the Job had actually completed successfully**:
- Pod status: `Succeeded`
- Exit code: `0`
- All MOFED checks passed
- Job completion logs showed success

**Root Cause**:
- `argocd.argoproj.io/hook-delete-policy: BeforeHookCreation`
- This keeps hook resources around after completion
- ArgoCD adds finalizer: `argocd.argoproj.io/hook-finalizer`
- ArgoCD sync engine got stuck waiting for hook to complete even though it already did
- Known ArgoCD state machine issue with hook lifecycle

**Manual Interventions Required**:
1. Attempt to trigger ArgoCD sync: `oc patch application gpu-operator-nfd ...`
2. Check Job status repeatedly: `oc get job wait-for-mofed-before-gpu -n nvidia-gpu-operator`
3. Check pod exit code: `oc get pod ... -o jsonpath='{.status.containerStatuses[0].state.terminated.exitCode}'`
4. Remove Job finalizers: `oc patch job wait-for-mofed-before-gpu -p '{"metadata":{"finalizers":[]}}' --type=merge`
5. Clear Application operation state: `oc patch application gpu-operator-nfd --type=json -p '[{"op":"remove","path":"/operation"}]'`
6. Trigger multiple sync attempts: `oc patch application gpu-operator-nfd ...` (repeated)
7. Clear operation state again: `kubectl patch application gpu-operator-nfd --type=json -p '[{"op":"remove","path":"/status/operationState"}]'`
8. **Eventually bypass ArgoCD entirely**: `oc apply -k manifests/30-gpu-operator-nfd/`

**Why Manual**: ArgoCD bug/limitation - finalizers on completed hooks prevent sync progression

**Fix Applied** (Commit: c040139):
```yaml
# manifests/30-gpu-operator-nfd/00-wait-for-mofed-presync.yaml
# Changed ALL hook resources from:
argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
# To:
argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

**Why This Works**:
- `HookSucceeded` auto-deletes hook resources immediately after successful completion
- No lingering resources with finalizers
- ArgoCD state machine doesn't get stuck
- Same pattern used in Network Operator version discovery (which worked perfectly)

---

### 5. GPU ClusterPolicy Not Deploying

**Issue**: ClusterPolicy resource remained "Missing" in ArgoCD Application status

**Root Cause**:
- Cascading failure from Issue #4
- ArgoCD sync stuck on PreSync hook
- ClusterPolicy sync never started

**Manual Intervention Required**:
- Bypass ArgoCD entirely: `oc apply -k manifests/30-gpu-operator-nfd/`
- (Already counted in Issue #4, same command)

**Why Manual**: Cascading from hook finalizer issue

**Fix**: Resolved by fixing Issue #4 (hook delete policy)

---

## Summary of All Manual Interventions

| # | Issue | Manual Steps Count | Root Cause | Fix Status | Commit |
|---|-------|-------------------|------------|------------|--------|
| 0 | Cleanup workflow | 5 steps | No auto-cleanup mechanism | **UNFIXED** | N/A |
| 1 | Bootstrap deployment | 4 steps | No bootstrap automation | **UNFIXED** | N/A |
| 2 | NicClusterPolicy version error | 8 steps | Wrong resource in kustomization | FIXED | e257d39 |
| 3 | GPU PreSync RBAC | 7 steps | Missing API permissions | FIXED | 730fda7 |
| 4 | ArgoCD hook stuck | 8 steps | BeforeHookCreation policy | FIXED | c040139 |
| 5 | ClusterPolicy not deploying | (counted in #4) | Cascading from #4 | FIXED | c040139 |

**Total Manual Intervention Steps**: 32 steps across 5 categories

---

## Remaining Manual Interventions (UNFIXED)

### Issue #0: Cleanup Workflow

**Current Process** (5 manual steps):
1. Edit `envs/baremetal-lab/kustomization.yaml` - uncomment `- ../../manifests/00-cleanup`
2. Commit and push OR apply/sync
3. Wait for Job completion
4. Edit kustomization again - re-comment cleanup line
5. Commit changes (if using Git)

**Options to Eliminate**:

**Option A**: Accept cleanup as a documented manual pre-requisite step
- Document in README: "Run cleanup before fresh deployment"
- Keep current Job-based approach
- **Pros**: Simple, already works
- **Cons**: Still requires manual intervention

**Option B**: Separate cleanup script outside GitOps
- Shell script in `scripts/cleanup.sh`
- Runs `oc delete` commands directly
- **Pros**: No kustomization editing needed
- **Cons**: You previously rejected this approach ("I don't like bash scripts")

**Option C**: Make cleanup a deletable ArgoCD Application
- Create `apps/00-cleanup/application.yaml` with sync-wave: "-10"
- User deploys cleanup app when needed: `oc apply -f apps/00-cleanup/application.yaml`
- After cleanup completes, delete the app: `oc delete application cleanup -n openshift-gitops`
- **Pros**: Kubernetes-native, repeatable, no kustomization editing
- **Cons**: Still requires manual app creation/deletion

**Option D**: Cleanup on every deployment with idempotent logic
- Make cleanup Job part of permanent sync-wave -10
- Job checks if cleanup needed (detect existing operators)
- Only runs cleanup if old resources exist
- **Pros**: Fully automated, no intervention
- **Cons**: Runs on every sync (slower), more complex logic

**Recommended**: Option C or D

---

### Issue #1: Bootstrap Deployment

**Current Process** (4 manual steps):
1. Apply prerequisites: `oc apply -k overlays/baremetal`
2. Wait for auto-approval Job
3. Wait for ArgoCD operator CSV
4. Apply root app: `oc apply -k envs/baremetal-lab`

**Options to Eliminate**:

**Option A**: Single bootstrap script
- Shell script: `./deploy.sh`
- Applies prerequisites, waits for ArgoCD, deploys root app
- **Pros**: One command deployment
- **Cons**: Bash script (you may not like this)

**Option B**: Bootstrap Job in OLM subscription
- InstallPlan auto-approval Job also deploys root app
- Waits for ArgoCD instance to be ready
- Applies root GitOps Application
- **Pros**: Fully automated after `oc apply -k overlays/baremetal`
- **Cons**: Complex Job logic, tight coupling

**Option C**: ArgoCD App-of-Apps pattern with bootstrap
- Root app includes a bootstrap sub-app
- Bootstrap app has sync-wave -100, creates all other apps
- **Pros**: GitOps-native, self-contained
- **Cons**: Chicken-and-egg: how does root app get deployed?

**Option D**: Kustomize all-in-one
- Combine prerequisites and root app into single kustomization
- Single command: `oc apply -k bootstrap/`
- Includes wait logic in Job for ArgoCD readiness
- **Pros**: Single command, Kubernetes-native
- **Cons**: Still one manual `oc apply -k` needed

**Option E**: Accept manual bootstrap as documented step
- Document: "1. oc apply -k overlays/baremetal  2. oc apply -k envs/baremetal-lab"
- **Pros**: Simple, clear, already works
- **Cons**: Not zero-intervention

**Recommended**: Option D (single bootstrap kustomization) or Option E (document it)

**Note**: Achieving **truly** zero-intervention from a bare cluster requires some form of initial bootstrap trigger. Even managed GitOps platforms (Flux, ArgoCD Autopilot) require an initial `kubectl apply` or CLI command.

---

## Expected Behavior After Fixes

### Wave 2: Network Operator
1. ✅ Operator deploys (sync wave 2)
2. ✅ PreSync hook `discover-ofed-version` runs
3. ✅ Hook queries operator deployment for default OFED version
4. ✅ Hook creates NicClusterPolicy with discovered version
5. ✅ Hook completes and auto-deletes (HookSucceeded)
6. ✅ Application syncs remaining resources
7. ✅ MOFED driver DaemonSet deploys

### Wave 3: MOFED Readiness
1. ✅ Job `wait-for-mofed-ready` runs
2. ✅ Waits for all MOFED pods to be Ready (2/2)
3. ✅ Completes successfully
4. ✅ Application marked Synced/Healthy

### Wave 4: GPU Operator
1. ✅ PreSync hook `wait-for-mofed-before-gpu` runs
2. ✅ Checks for NicClusterPolicy (now has RBAC)
3. ✅ Checks for MOFED DaemonSet
4. ✅ Waits for all MOFED pods Ready
5. ✅ Hook completes and auto-deletes (HookSucceeded)
6. ✅ ArgoCD proceeds with sync (no stuck state)
7. ✅ ClusterPolicy created
8. ✅ GPU driver DaemonSet deploys

---

## Validation Steps for Next Deployment

1. Run cleanup: Uncomment `- ../../manifests/00-cleanup` in kustomization, sync, re-comment
2. Deploy prerequisites: `oc apply -k overlays/baremetal`
3. Wait for auto-approval Job to approve InstallPlan
4. Wait for ArgoCD operator CSV: Succeeded
5. Deploy root app: `oc apply -k envs/baremetal-lab`
6. **DO NOT INTERVENE** - Monitor only:
   - Wave -1: Node preparation
   - Wave 1: NFD, SR-IOV operators
   - Wave 2: Network Operator (version discovery → NicClusterPolicy → MOFED drivers)
   - Wave 3: MOFED readiness gate
   - Wave 4: GPU PreSync hook → ClusterPolicy → GPU drivers

Expected result: **ZERO manual interventions required**

---

## Lessons Learned

1. **ArgoCD Hook Delete Policies Matter**
   - `BeforeHookCreation`: Leaves resources around, can cause state issues
   - `HookSucceeded`: Auto-cleanup, prevents finalizer problems
   - **Always use HookSucceeded for PreSync/PostSync hooks**

2. **RBAC Must Be Complete Before Deployment**
   - PreSync hooks need all permissions they use in scripts
   - Missing permissions cause silent failures in loops

3. **Dynamic Version Discovery > Static Configuration**
   - Operator defaults change with updates
   - Version discovery Job ensures compatibility
   - Prevents admission webhook validation errors

4. **Test Hooks in Isolation**
   - Verify hook completion behavior
   - Check ArgoCD Application status after hook
   - Ensure auto-delete works correctly

---

## Related Issues

- ArgoCD hook finalizer behavior: https://github.com/argoproj/argo-cd/issues/7805
- OLM InstallPlan approval: Handled by auto-approval Job in prerequisites
- CSV namespace proliferation: Handled by cleanup Job in manifests/00-cleanup
