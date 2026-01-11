# NVIDIA Network Operator Configuration

This directory contains the dynamic NicClusterPolicy configuration for the NVIDIA Network Operator.

## How It Works: Dynamic Version Discovery + User Customization

This implementation provides the **best of both worlds**:

1. ✅ **Automatic version discovery** - OFED driver version is pulled from the operator's defaults
2. ✅ **User customization** - All other settings can be modified via a template

### Architecture

```
┌─────────────────────────────────────┐
│  User Edits Template (ConfigMap)   │
│  - nodeAffinity                     │
│  - env vars                         │
│  - probes                           │
│  - upgrade policy                   │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  PreSync Hook Job (Wave -1)         │
│  1. Discover OFED version from      │
│     operator deployment             │
│  2. Read user template from         │
│     ConfigMap                       │
│  3. Inject version into template    │
│  4. Apply NicClusterPolicy          │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  NicClusterPolicy Applied           │
│  - User settings                    │
│  - Auto-discovered version          │
└─────────────────────────────────────┘
```

## User Customization

### Option 1: Edit Before Deployment (Recommended)

Edit the ConfigMap template in the git repository before deploying:

**File:** `manifests/28-nvidia-network-operator/00-version-discovery-job.yaml`

Find the ConfigMap section (lines 1-65) and modify:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nicclusterpolicy-template
data:
  template.yaml: |
    spec:
      ofedDriver:
        # Add custom environment variables
        env:
          - name: UNLOAD_STORAGE_MODULES
            value: 'true'
          - name: MY_CUSTOM_VAR      # ← Add here
            value: 'my_value'

        # Customize probe timings
        livenessProbe:
          initialDelaySeconds: 60   # ← Change here
          periodSeconds: 30

        # Customize upgrade behavior
        upgradePolicy:
          maxParallelUpgrades: 1    # ← Change here
```

Then commit and deploy:
```bash
git add manifests/28-nvidia-network-operator/00-version-discovery-job.yaml
git commit -m "Customize NicClusterPolicy for our cluster"
git push origin master
oc apply -k overlays/baremetal/
```

### Option 2: Edit After Deployment (Live Changes)

Edit the ConfigMap directly in the cluster:

```bash
# Edit the template
oc edit configmap nicclusterpolicy-template -n nvidia-network-operator

# Make your changes, save and exit

# Force ArgoCD to re-run the PreSync hook
oc delete job discover-ofed-version -n nvidia-network-operator
oc patch application nvidia-network-operator -n openshift-gitops --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{}}}'
```

## What Gets Auto-Discovered vs User-Controlled

| Setting | Source | Editable? |
|---------|--------|-----------|
| `ofedDriver.version` | Auto-discovered from operator | ❌ No (automatic) |
| `ofedDriver.image` | User template | ✅ Yes |
| `ofedDriver.repository` | User template | ✅ Yes |
| `ofedDriver.env[]` | User template | ✅ Yes |
| `ofedDriver.livenessProbe` | User template | ✅ Yes |
| `ofedDriver.readinessProbe` | User template | ✅ Yes |
| `ofedDriver.startupProbe` | User template | ✅ Yes |
| `ofedDriver.upgradePolicy` | User template | ✅ Yes |
| `spec.nodeAffinity` | User template | ✅ Yes |

## Example Customizations

### Add Custom Environment Variable

```yaml
env:
  - name: UNLOAD_STORAGE_MODULES
    value: 'true'
  - name: CREATE_IFNAMES_UDEV
    value: 'true'
```

### Change Upgrade Behavior

```yaml
upgradePolicy:
  autoUpgrade: false              # Disable auto-upgrade
  maxParallelUpgrades: 1          # Only upgrade 1 node at a time
  drain:
    enable: false                  # Don't drain nodes
```

### Adjust Health Probes

```yaml
livenessProbe:
  initialDelaySeconds: 60         # Wait longer before first probe
  periodSeconds: 60               # Check less frequently
readinessProbe:
  initialDelaySeconds: 20
  periodSeconds: 10
```

### Change Node Selection

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchExpressions:
      # Only deploy on nodes with specific label
      - key: network.llm-d-lab/roce-enabled
        operator: Exists
```

## Viewing Current Configuration

To see the final applied NicClusterPolicy (with injected version):

```bash
oc get nicclusterpolicy nic-cluster-policy -o yaml
```

To see what version was discovered:

```bash
oc get nicclusterpolicy nic-cluster-policy -o jsonpath='{.spec.ofedDriver.version}'
```

To view the PreSync Job logs:

```bash
oc logs -n nvidia-network-operator job/discover-ofed-version
```

## Troubleshooting

### Version not being injected

Check the Job logs:
```bash
oc logs -n nvidia-network-operator job/discover-ofed-version
```

### Template changes not applying

1. Delete the Job to force recreation:
   ```bash
   oc delete job discover-ofed-version -n nvidia-network-operator
   ```

2. Re-sync the ArgoCD application:
   ```bash
   oc patch application nvidia-network-operator -n openshift-gitops --type merge -p '{"operation":{"sync":{}}}'
   ```

### MOFED Pods CrashLoopBackOff - irdma Module Conflict

**Symptom:** MOFED driver pods crash with error:
```
rmmod: ERROR: Module ib_uverbs is in use by: irdma
Command "/etc/init.d/openibd restart" failed with exit code: 1
```

**Cause:** Intel RDMA (irdma) driver conflicts with NVIDIA MOFED driver.

**Check if you have this issue:**
```bash
oc debug node/<worker-node-name> -- chroot /host lsmod | grep irdma
```

If `irdma` module is loaded, apply the workaround in `overlays/baremetal/README.md`.

## Benefits of This Approach

✅ **No version management** - Operator handles version selection automatically
✅ **User customization** - All settings except version can be modified
✅ **Git-tracked changes** - Template is version controlled
✅ **Works for everyone** - No git write access needed
✅ **Cluster-specific configs** - Different clusters can use different templates
✅ **Upgrade safe** - Version updates automatically when operator upgrades

## Alternative: Pin Specific Version

If you need to pin a specific OFED version (not recommended):

1. Remove the dynamic discovery Job
2. Create a static `nicclusterpolicy.yaml` with a pinned version
3. Manually update the version when needed

This loses automatic version discovery but gives full control.
