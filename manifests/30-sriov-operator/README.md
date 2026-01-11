# SR-IOV Network Operator Configuration

This directory contains the base SR-IOV Network Operator configuration.

## Configuration Options

### Parallel Node Updates

The `maxUnavailable` setting in `sriovoperatorconfig.yaml` controls how many nodes can be updated simultaneously during SR-IOV configuration.

**Default: 1** (serial updates - one node at a time)

```yaml
spec:
  maxUnavailable: 1  # Only one node updates at a time (safest)
```

**To update multiple nodes in parallel:**

```yaml
spec:
  maxUnavailable: 2  # Two nodes can update simultaneously
```

**Considerations:**

- **maxUnavailable: 1** (default)
  - ✅ Safest for production
  - ✅ Ensures cluster stability
  - ❌ Slowest: ~60 min per node sequentially
  - Example: 3 worker nodes = ~180 minutes total

- **maxUnavailable: 2**
  - ⚡ Faster: ~60 min for 2 nodes in parallel
  - ⚠️ Less safe: multiple nodes draining/rebooting simultaneously
  - Example: 3 worker nodes = ~120 minutes total

- **maxUnavailable: N** (where N = number of worker nodes)
  - ⚡⚡ Fastest: all nodes update in parallel
  - ⚠️⚠️ Risky: entire worker pool unavailable during updates
  - 🚫 NOT recommended for production
  - Example: 3 worker nodes = ~60 minutes total

**How to change:**

Edit `manifests/30-sriov-operator/sriovoperatorconfig.yaml` and modify the `maxUnavailable` value before deployment.

## References

See https://github.com/opendatahub-io/kserve/tree/release-v0.15/docs/samples/llmisvc
