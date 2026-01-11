# Operator Cleanup Job

This directory contains a Kubernetes Job that performs comprehensive cleanup of all operators and their resources.

## Purpose

Run this Job **BEFORE** a fresh deployment to prevent conflicts from previous installations:
- Removes all operator CSVs from ALL namespaces (solves the 91-namespace CSV issue)
- Deletes operator subscriptions and install plans
- Removes operator namespaces
- Deletes CRDs and webhook configurations

## Usage

### Method 1: Temporary Uncomment in Kustomization (Recommended)

1. Edit `envs/baremetal-lab/kustomization.yaml`
2. Uncomment the cleanup resource:
   ```yaml
   resources:
     - namespace.yaml
     - ../../manifests/00-cleanup  # <-- Uncomment this line
     # - ../../apps/00-node-preparation
     # ... other resources
   ```
3. Apply via ArgoCD or kubectl:
   ```bash
   # Via ArgoCD (if already deployed)
   argocd app sync root-app

   # Or via kubectl
   kubectl apply -k envs/baremetal-lab
   ```
4. Wait for the cleanup Job to complete:
   ```bash
   oc logs -n openshift-operators job/cleanup-operators -f
   ```
5. Re-comment the cleanup resource before deploying the stack
6. Proceed with fresh deployment

### Method 2: Direct Application

```bash
# Apply the cleanup Job directly
oc apply -k manifests/00-cleanup

# Watch the cleanup process
oc logs -n openshift-operators job/cleanup-operators -f

# Wait for completion
oc wait --for=condition=complete job/cleanup-operators -n openshift-operators --timeout=600s

# Delete the Job (auto-deletes after 5 minutes anyway)
oc delete -k manifests/00-cleanup
```

## What Gets Cleaned Up

1. **ArgoCD/GitOps Operator**
   - Subscription, CSVs (all 91 namespaces), deployments

2. **GPU Operator**
   - Subscription, ClusterPolicy, CSVs

3. **Network Operator**
   - Subscription, NicClusterPolicy, CSVs

4. **NFD Operator**
   - Subscription, CSVs

5. **SR-IOV Operator**
   - Subscription, CSVs

6. **Namespaces**
   - nvidia-gpu-operator
   - nvidia-network-operator
   - openshift-nfd
   - openshift-sriov-network-operator
   - openshift-gitops

7. **CRDs and Webhooks**
   - All operator-related CRDs
   - Webhook configurations

## Verification

After cleanup completes, verify:

```bash
# No operator CSVs remain
oc get csv -A | grep -E "(gitops|gpu|nvidia|nfd|sriov)" || echo "Clean ✓"

# No operator namespaces remain
oc get ns | grep -E "(nvidia|nfd|sriov|gitops)" || echo "Clean ✓"
```

## Notes

- The Job runs in the `openshift-operators` namespace
- Auto-deletes after 5 minutes (`ttlSecondsAfterFinished: 300`)
- Safe to run multiple times (idempotent)
- Uses cluster-scoped RBAC with minimal required permissions
