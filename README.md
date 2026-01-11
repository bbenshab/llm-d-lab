# 🚀 LLM-D Benchmarking Lab

Accelerate reproducible inference experiments for large language models with [LLM-D](https://github.com/llm-d)! This lab automates the setup of a complete evaluation environment on OpenShift/OKD: GPU worker pools, core operators, observability, traffic control, and ready-to-run example workloads. 

> ⚠️ **Experimental Project:** This is a Work in progress repository. Breaking changes may occur. This project is not meant for production use.

<img src="docs/diagram.svg" alt="Architecture Diagram" width="800"/>

---

## 👤 Who is this for?
- 🛠️ Performance engineers running LLM-D & OpenShift AI benchmarks
- 🏗️ Platform engineers & SREs building scalable LLM serving
- 🧩 Solution architects prototyping LLM-backed solutions
- 🧑‍🔬 Researchers validating distributed inference engines and orchestration strategies

---

## ✨ Key Capabilities
- ☁️ AWS & IBM Cloud support
- ⚡ Automated infrastructure: MachineSets, autoscaling, ClusterAutoscaler
- 🧩 Operators & platform services: NVIDIA GPU Operator, Node Feature Discovery, Descheduler, KEDA, Gateway API, Kuadrant, Authorino, cert-manager
- 🕵️ Observability: NetObserv, LokiStack, Grafana dashboards (WIP)
- 🔬 Example manifests for KServe LLMInferenceService & KV-cache routing
- 🧪 Precise-prefix cache-aware experiments
- 🔄 GitOps App-of-Apps via Argo CD (WIP)

## 💡 Philosophy

- 🔁 GitOps-first: Everything is managed via ArgoCD applications
- 📄 Avoid local script execution: Prefer declarative manifests and Kubernetes control loops over imperative scripts run locally
- ⚡ Low friction and minimal dependencies on the user's workstation tooling
- 🧩 Modular & extensible: Fork and Customize via Kustomize overlays
- ☸️ Cloud-native: Leverage the full potential of Kubernetes, OpenShift, and the Operators pattern
- 🔁 Reproducible: Version-controlled manifests for consistent setups
- 🔬 Experiment-focused: Ready-to-run LLM-D workloads & experiments

------

## 🛠️ Cluster Prerequisites
- OpenShift 4.20+
- Cluster-admin permissions

## ⚡ Quickstart on AWS

1. Clone this repo.
2. Fill the GitOps Root Application in [overlays/aws/root-app.yaml](./overlays/aws/root-app.yaml) (See [app-of-apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)):
  - The minimum required values are the ClusterApi identifier, the cluster's region and available zones, and the routes for the Gateway API. You can also fork and replace the repo URL with your own. This is recommended because using the main repo URL directly binds your cluster to the current state of this repo. Further documentation is planned for customizing the environments.
3. Fill in the secrets in `overlays/aws/99-*.example.yaml` and save as `overlays/aws/99-*.yaml`.
4. Deploy with `oc apply -k overlays/aws/`.
5. Wait for all ArgoCD applications to become ready: you can find them in the OpenShift WebUI or via `oc get applications -n openshift-gitops`.
6. From here on any changes to the repo will be automatically applied to the cluster by ArgoCD, and ArgoCD will continuously ensure the cluster state matches the desired state defined in the Git repository.

NOTE: The initial setup will take longer, especially if the cluster requires scaling out worker nodes. The applications will report progressing and degraded states until all dependencies are met and the cluster converges to the desired state.

## ⚡ Quickstart on IBMCloud

To be tested and documented.

---

## 📦 What Gets Deployed

- **Infrastructure:**  
  MachineSet, MachineAutoscaler, ClusterAutoscaler
- **Core Operators:**  
  GPU Operator, Node Feature Discovery, Descheduler, KEDA
- **Networking & API Gateway:**  
  Gateway API, Kuadrant, Authorino, cert-manager
- **Observability:**  
  Grafana, NetObserv, LokiStack
- **GPU & System Tuning:**  
  NVIDIA GPU Operator, NFD, Descheduler
- **LLM-D & RHOAI Scaffolding:**  
  Upstream & downstream pre-reqs

---

## 🔄 Self-Healing & Automation

The deployment includes **automated self-healing** via an ArgoCD health monitor that runs every 2 minutes to detect and fix common GitOps deployment issues:

### What It Fixes Automatically

1. **Stuck ArgoCD Sync Operations** (>3 minutes)
   - Clears operations waiting for deleted resources
   - Unblocks wave-based deployments from progressing

2. **Stuck ArgoCD Hook Jobs**
   - Removes finalizers from PreSync/PostSync Jobs stuck in Terminating state
   - Prevents namespace deletion hangs

3. **Failed PreSync Hook Detection**
   - Detects Applications with SyncFailed resources
   - Automatically retriggers sync to retry failed hooks
   - **Example:** NVIDIA Network Operator's version discovery Job

4. **Stuck Pod Detection**
   - Monitors pods in Pending state for >3 minutes
   - Logs scheduling failure reasons for investigation
   - Alerts for manual intervention when needed

### How It Works

- **CronJob:** Runs every 2 minutes in `openshift-gitops` namespace
- **Proactive:** Detects issues before they block deployments
- **Transparent:** All actions logged in Job output
- **Safe:** Only clears stuck states, doesn't modify resources

### Viewing Health Monitor Logs

```bash
# See latest health check
oc logs -n openshift-gitops job/argocd-health-monitor-<latest> --tail=100

# Watch CronJob schedule
oc get cronjob argocd-health-monitor -n openshift-gitops
```

**Location:** `manifests/27-gitops/argocd-health-monitor.yaml`

---

## 🔧 Troubleshooting

### OLM CSV Stuck in Pending State

**Symptom:** After cleanup or fresh deployment, operator ClusterServiceVersion (CSV) remains in `Pending` phase for several minutes with status `RequirementsNotMet`.

**Example:**
```bash
oc get csv -n openshift-operators
# NAME                                DISPLAY                  VERSION   REPLACES   PHASE
# openshift-gitops-operator.v1.19.0   Red Hat OpenShift GitOps 1.19.0              Pending
```

**Root Cause:** This occurs when OLM recreates a CSV before its required CRDs are fully deleted from the cluster. The timing issue typically happens during cleanup when:
1. Cleanup Job deletes operator Subscription → OLM starts CSV cleanup
2. OLM has already recreated the CSV
3. Cleanup Job deletes CRDs 10 seconds later
4. CSV is now stuck waiting for CRDs that won't be created because InstallPlan already shows "Complete"

**Diagnostic Commands:**

```bash
# 1. Check CSV status
oc get csv <csv-name> -n <namespace> -o jsonpath='{.status.phase}'
# Output: Pending

# 2. Check detailed CSV conditions
oc describe csv <csv-name> -n <namespace>
# Look for:
#   Message: one or more requirements couldn't be found
#   Reason: RequirementsNotMet

# 3. Check missing CRDs
oc get csv <csv-name> -n <namespace> -o yaml | grep -A 30 "status:"
# Look for CRDs with status: NotPresent

# 4. Check InstallPlan status
oc get installplan -n <namespace>
# If InstallPlan shows "Complete" but CRDs are missing, CSV is stuck

# Example for ArgoCD operator:
oc get csv openshift-gitops-operator.v1.19.0 -n openshift-operators -o jsonpath='{.status.phase}'
oc describe csv openshift-gitops-operator.v1.19.0 -n openshift-operators
oc get installplan -n openshift-operators
```

**Resolution:**

Delete the stuck CSV to force OLM to start fresh with a new InstallPlan:

```bash
# Delete the stuck CSV
oc delete csv <csv-name> -n <namespace>

# Example for ArgoCD operator:
oc delete csv openshift-gitops-operator.v1.19.0 -n openshift-operators

# Wait 30-60 seconds, then verify:
oc get csv -n <namespace>
# Phase should progress: Installing → Succeeded
```

**Prevention:** The auto-approval Job (`overlays/<env>/00-prerequisites.yaml`) includes retry logic to handle this automatically during initial deployment. For manual cleanup scenarios, ensure sufficient time between CSV and CRD deletion steps.

---

## 🧹 Cluster Cleanup

Before redeploying or when you need to start fresh, use the automated cleanup job to remove all operator resources.

### Quick Cleanup

```bash
# Apply the cleanup Job directly
oc apply -k manifests/00-cleanup

# Watch the cleanup process
oc logs -n openshift-operators job/cleanup-operators -f

# Wait for completion (auto-deletes after 5 minutes)
oc wait --for=condition=complete job/cleanup-operators -n openshift-operators --timeout=600s
```

### What Gets Cleaned Up

The cleanup job removes:
- **Operators:** GPU Operator, NVIDIA Network Operator, NFD, SR-IOV, GitOps
- **CSVs:** From ALL namespaces (solves the 91-namespace CSV issue)
- **Resources:** Subscriptions, InstallPlans, ClusterPolicies, NicClusterPolicies
- **Namespaces:** nvidia-gpu-operator, nvidia-network-operator, openshift-nfd, openshift-sriov-network-operator, openshift-gitops
- **CRDs:** All operator-related custom resource definitions
- **RBAC:** ClusterRoles and ClusterRoleBindings

### Verify Cleanup

```bash
# Check no operator CSVs remain
oc get csv -A | grep -E "(gitops|gpu|nvidia|nfd|sriov)" || echo "Clean ✓"

# Check no operator namespaces remain
oc get ns | grep -E "(nvidia|nfd|sriov|gitops)" || echo "Clean ✓"

# Check no operator CRDs remain
oc get crd | grep -E "(nvidia|mellanox|nfd|sriov|argoproj)" || echo "Clean ✓"
```

### Via ArgoCD (Alternative Method)

1. Edit `envs/baremetal-lab/kustomization.yaml`
2. Uncomment the cleanup resource:
   ```yaml
   resources:
     - ../../manifests/00-cleanup  # <-- Uncomment this line
   ```
3. Sync via ArgoCD: `argocd app sync root-app`
4. Wait for completion, then re-comment before deploying the stack

**Note:** The cleanup job is safe to run multiple times (idempotent) and auto-deletes after 5 minutes. See [manifests/00-cleanup/README.md](manifests/00-cleanup/README.md) for details.

---

## 🏃 Running Example Workloads

The `examples/` directory contains sample manifests and Helm charts intended for deploying LLM-D workloads.

## 🧪 Example experiment: Upstream LLM-D w/ Workload Variant Autoscaler

Refer to [experiments/workload-variant-autoscaler](experiments/workload-variant-autoscaler) for a full example of deploying Upstream LLM-D with Workload Variant Autoscaling and gathering metrics for analysis.

## ⚠️ Limitations and Notes

- When deploying the complete env in `env/lab`, MachineSets and Cluster Autoscaling are configured alongside the operators' installation. In SNO clusters, the master is configured not to host user workloads, but the MachineSets and Cluster Autoscaler continue to progress. The cluster will eventually converge to the desired state, but it's recommended to have some worker nodes available in advance to speed up initial setup and improve reliability during this phase.
- The uninstallation of this stack is not yet fully supported. For example, operators managed via OLM will not be removed automatically; manual cleanup may be required. Still, once MachineSets and Cluster Autoscaler are removed, ensure you provision some worker nodes to enable rescheduling of the remaining workloads.

## 📝 Backlog

- [ ] Add IBMCloud overlay
- [ ] RWX Storage Class
- [ ] CertManager configuration
- [ ] Kuadrant configuration
- [ ] Authorino configuration
- [ ] Other Grafana dashboards
- [ ] Pin artifacts (operators, helm charts, ...) to specific versions/commits for reproducibility and enable renovate bot
- [ ] Grafana Authentication should be backed by OpenShift OAuth
- [ ] Review RBACs, resource requests/limits, and other manifests refinements
- [ ] Multi-tenancy and concurrent deployments and experiment jobs management, e.g., via Tekton Pipelines, Kueue, and other tools for workload orchestration
- [ ] Most operators are configured in their simplest form and in the global manifests. Further tuning and customizations may be needed for specific use cases, e.g., configuring the NVIDIA GPU Operator and networking. Such a configuration will be unlocked from the global manifests and moved to dedicated environment overlays or experiments, given the assessment of the requirements to enable multi-tenancy and concurrent experiments on the same cluster (see previous point)
- [ ] HyperShift hosted clusters support/Multi-Cluster management
- [ ] More example workloads and experiments
- [ ] Documentation


## 📁 Structure of the repo

```shell
apps/                  # ArgoCD Applications manifests. Each folder should refer to an Helm or Kustomize project, defined in /manifests if not external.
envs/                  # Kustomize base for different environments (e.g., lab, demo, AWS, IBMCloud)
examples/              # Example Helmfiles and manifests to deploy LLM-D workloads on top of the deployed platform, either Upstream or via RHOAI/ODH.
experiments/           # Example experiments leveraging the deployed platform
manifests/             # Helm charts and Kustomize bases for operators and platform services
overlays/              # Kustomize overlays for different environments. 
```

`/overlays/` in this repo only contains examples at the time of writing.
Clusters-specific `overlays/` should not have secrets and might stay in private forks to control several clusters with different configurations (cloud providers, hostnames, secrets, ...), fully leveraging the GitOps App of Apps pattern.

The examples inherently violate the pattern as they change the children applications with information about the managed cluster we prefer not to disclose publicly, e.g., the base domain of the managed clusters. This is today a practical compromise for experimentation purposes to avoid over-complicating secrets management and delivery.

## 🔧 Customizing the environments

To be documented
