# RoCE Discovery and SR-IOV Resource Generator

Automatically discovers RoCE-capable network interfaces and generates SR-IOV policies and networks.

## Quick Start

The generator automatically discovers all Mellanox/NVIDIA NICs with RoCE capability and creates SR-IOV resources. **No manual configuration needed for basic deployment.**

## Subnet Configuration Modes

You can configure how IP addresses are allocated to SR-IOV networks by setting `SUBNET_MODE` in `job-generator.yaml`.

### Mode 1: Separate Subnets (DEFAULT - Recommended)

**Each NIC gets its own isolated /24 subnet:**

```yaml
env:
  - name: SUBNET_MODE
    value: "separate"  # DEFAULT
  - name: IP_RANGE_BASE
    value: "10.0"
```

**Result:**
- `ens3f0np0` → `10.0.101.0/24` (254 IPs)
- `ens3f1np1` → `10.0.102.0/24` (254 IPs)
- `ens7f0np0` → `10.0.103.0/24` (254 IPs)

**Use when:**
- ✅ You need network isolation between NICs
- ✅ Running multi-tenant workloads
- ✅ Segregating traffic types (storage vs compute)
- ✅ Maximum scalability (254 IPs × number of NICs)

### Mode 2: Shared Subnet

**All NICs share the same /24 subnet:**

```yaml
env:
  - name: SUBNET_MODE
    value: "shared"
  - name: IP_RANGE_BASE
    value: "10.0"
```

**Result:**
- `ens3f0np0` → `10.0.100.0/24` (shared pool)
- `ens3f1np1` → `10.0.100.0/24` (shared pool)
- `ens7f0np0` → `10.0.100.0/24` (shared pool)
- Total: **254 IPs shared across all NICs**

**Use when:**
- ✅ Pods on different NICs need layer-2 connectivity
- ✅ Legacy applications expect flat network
- ✅ High-availability pairs require same subnet

**Limitations:**
- ⚠️  Only 254 IPs total (shared by all pods on all NICs)
- ⚠️  No network isolation
- ⚠️  Risk of IP exhaustion with many pods

## Configuration Examples

### Example 1: Default Configuration (Separate Subnets)

```yaml
# manifests/35-roce-discovery/job-generator.yaml
env:
  - name: NUM_VFS
    value: "1"              # 1 Virtual Function per NIC
  - name: MTU
    value: "9000"           # Jumbo frames for RDMA
  - name: SUBNET_MODE
    value: "separate"       # Each NIC gets own subnet
  - name: IP_RANGE_BASE
    value: "10.0"           # Use 10.0.x.x addresses
  - name: NETWORK_NAMESPACE
    value: "default"
  - name: ROUTE_DEST
    value: "192.168.75.0/24"
```

**Generates:**
```yaml
# SriovNetwork for ens3f0np0
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetwork
metadata:
  name: ens3f0np0-network
spec:
  ipam: |
    {
      "type": "whereabouts",
      "range": "10.0.101.0/24",
      "routes": [{"dst": "192.168.75.0/24"}]
    }
  resourceName: ens3f0np0rdma

# SriovNetwork for ens3f1np1
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetwork
metadata:
  name: ens3f1np1-network
spec:
  ipam: |
    {
      "type": "whereabouts",
      "range": "10.0.102.0/24",
      "routes": [{"dst": "192.168.75.0/24"}]
    }
  resourceName: ens3f1np1rdma
```

### Example 2: Shared Subnet with Custom IP Range

```yaml
# manifests/35-roce-discovery/job-generator.yaml
env:
  - name: SUBNET_MODE
    value: "shared"         # All NICs share subnet
  - name: IP_RANGE_BASE
    value: "192.168"        # Use 192.168.x.x addresses
```

**Generates:**
```yaml
# All SriovNetworks use same subnet
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetwork
metadata:
  name: ens3f0np0-network
spec:
  ipam: |
    {
      "type": "whereabouts",
      "range": "192.168.100.0/24",  # Shared with all NICs
      ...
    }
---
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetwork
metadata:
  name: ens3f1np1-network
spec:
  ipam: |
    {
      "type": "whereabouts",
      "range": "192.168.100.0/24",  # Same subnet
      ...
    }
```

### Example 3: Multiple VFs with 172.16.x.x Range

```yaml
env:
  - name: NUM_VFS
    value: "4"              # 4 VFs per NIC
  - name: SUBNET_MODE
    value: "separate"
  - name: IP_RANGE_BASE
    value: "172.16"
```

**Result:**
- `ens3f0np0` → `172.16.101.0/24` with 4 VFs
- `ens3f1np1` → `172.16.102.0/24` with 4 VFs

## How It Works

1. **Discovery Phase** (`job-discovery.yaml`)
   - DaemonSet runs on each worker node
   - Detects RoCE-capable NICs (Mellanox/NVIDIA)
   - Writes discovery results to hostPath volume

2. **Generator Phase** (`job-generator.yaml`)
   - Waits for all nodes to complete discovery
   - Collects results from all nodes
   - Deduplicates by (PF name, device ID)
   - Generates SriovNetworkNodePolicy and SriovNetwork for each unique NIC
   - Applies resources to cluster

3. **SR-IOV Operator** (automatic)
   - Creates Virtual Functions (VFs)
   - Configures device plugin
   - Makes resources available to pods

## Verification

```bash
# Check discovered NICs
oc get sriovnetworknodepolicy -n openshift-sriov-network-operator

# Check generated networks
oc get sriovnetwork -n openshift-sriov-network-operator

# Check allocatable resources per node
oc get nodes -o json | jq '.items[] | {
  node: .metadata.name,
  allocatable: .status.allocatable | with_entries(select(.key | test("rdma")))
}'

# View network IP ranges
oc get sriovnetwork -n openshift-sriov-network-operator \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.ipam}{"\n"}{end}' | \
  jq -r '. | "\(.metadata.name): \(.spec.ipam | fromjson | .range)"'
```

## Advanced Configuration

### Custom IP Ranges

Change the first two octets of IP addresses:

```yaml
- name: IP_RANGE_BASE
  value: "192.168"  # Use 192.168.x.x instead of 10.0.x.x
```

### Per-Namespace Networks

By default, SR-IOV networks are available in the `default` namespace. To make networks available in other namespaces, either:

**Option 1: Change default namespace**
```yaml
- name: NETWORK_NAMESPACE
  value: "my-workload-namespace"
```

**Option 2: Create additional SriovNetwork resources** for each namespace (recommended for multi-tenancy)

### Custom MTU

```yaml
- name: MTU
  value: "1500"  # Standard Ethernet (vs 9000 for jumbo frames)
```

## Troubleshooting

### No NICs discovered

```bash
# Check discovery pods
oc get pods -n openshift-sriov-network-operator -l app=roce-port-discovery

# View discovery logs
oc logs -n openshift-sriov-network-operator -l app=roce-port-discovery

# Check node labels
oc get nodes --show-labels | grep "pci-15b3.present"
```

### IP address conflicts

If using `SUBNET_MODE="shared"` and seeing IP conflicts:
- Switch to `SUBNET_MODE="separate"`
- Or increase subnet size (requires manual editing of generated resources)

### Networks not available in namespace

Check `NETWORK_NAMESPACE` setting or create additional SriovNetwork resources with different `networkNamespace` values.

## Related Documentation

- [SR-IOV Operator Configuration](../30-sriov-operator/README.md)
- [Baremetal Deployment Guide](../../overlays/baremetal/README.md)
- [OpenShift SR-IOV Network Operator Docs](https://docs.openshift.com/container-platform/latest/networking/hardware_networks/configuring-sriov-operator.html)
