# Node Feature Discovery (NFD) Configuration

**CRITICAL: NFD MUST BE DEPLOYED FIRST**

This is the first component deployed in the baremetal environment. NFD discovers hardware features and labels nodes, which all other operators depend on.

## Why NFD is First

NFD must be deployed before:
- **NVIDIA Network Operator** - Needs `pci-15b3.present` label
- **NVIDIA GPU Operator** - Needs `pci-10de.present` label
- **SR-IOV Operator** - Needs `pci-15b3.sriov.capable` label
- **RoCE Discovery** - Needs `pci-15b3.present` and `rdma.capable` labels

## Hardware Whitelist

This NodeFeatureRule explicitly discovers and labels:

### NVIDIA GPUs (Vendor: 10de)
- **Label**: `feature.node.kubernetes.io/pci-10de.present: "true"`
- **Purpose**: GPU Operator uses this to identify nodes with NVIDIA GPUs

### Mellanox/NVIDIA NICs (Vendor: 15b3)
- **Label**: `feature.node.kubernetes.io/pci-15b3.present: "true"`
- **Purpose**: Network Operator and SR-IOV Operator use this to identify nodes with Mellanox NICs

### SR-IOV Capability
- **Label**: `feature.node.kubernetes.io/pci-15b3.sriov.capable: "true"`
- **Purpose**: Identifies NICs that support SR-IOV (Virtual Functions)

### RDMA Capability
- **Label**: `feature.node.kubernetes.io/rdma.capable: "true"`
- **Purpose**: Identifies nodes with RDMA-capable network devices

### RDMA Available
- **Label**: `feature.node.kubernetes.io/rdma.available: "true"`
- **Purpose**: Identifies nodes where RDMA kernel modules are loaded

## Deployment Order

```
1. NFD Operator subscription (via apps/20-operators)
2. NFD Configuration (this - apps/15-nfd)
3. Wait for node labeling
4. NVIDIA Network Operator
5. NVIDIA GPU Operator
6. SR-IOV Operator
7. RoCE Discovery
```

## Verification

After NFD is running, verify nodes are labeled:

```bash
# Check NFD is running
oc get pods -n openshift-nfd

# Check if NodeFeatureRule is applied
oc get nodefeaturerule -n openshift-nfd

# Verify node labels
oc get nodes --show-labels | grep feature.node

# Check specific labels
oc get nodes -L feature.node.kubernetes.io/pci-10de.present
oc get nodes -L feature.node.kubernetes.io/pci-15b3.present
oc get nodes -L feature.node.kubernetes.io/pci-15b3.sriov.capable
oc get nodes -L feature.node.kubernetes.io/rdma.capable
```

## Troubleshooting

If nodes are not being labeled:

```bash
# Check NFD worker pods
oc get pods -n openshift-nfd -l app=nfd-worker

# Check NFD worker logs
oc logs -n openshift-nfd -l app=nfd-worker

# Check NFD master logs
oc logs -n openshift-nfd -l app=nfd-master

# Verify hardware is present
oc debug node/<node-name>
chroot /host
lspci | grep -i nvidia
lspci | grep -i mellanox
```

## Adding New Hardware

To whitelist additional hardware, add new rules to the NodeFeatureRule:

```yaml
- name: "custom-hardware"
  labels:
    "feature.node.kubernetes.io/custom.present": "true"
  matchFeatures:
    - feature: pci.device
      matchExpressions:
        vendor: {op: In, value: ["XXXX"]}  # Vendor ID
```

Vendor IDs can be found with:
```bash
lspci -nn | grep -i <device-name>
```
