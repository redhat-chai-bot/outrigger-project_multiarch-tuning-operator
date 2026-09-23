# Architecture

## Repository Layout

```text
api/
├── common/              # Shared constants (SingletonResourceObjectName), plugin definitions
│   └── plugins/         # Plugin system: NodeAffinityScoring, CelArchitecturePlacement, ExecFormatErrorMonitor
├── v1alpha1/            # Alpha API — DO NOT add new fields here; conversion hub is v1beta1
└── v1beta1/             # Storage version — ClusterPodPlacementConfig, PodPlacementConfig, ENoExecEvent types

cmd/
├── main.go              # Single binary entrypoint — mode dispatch via flags
└── enoexec-daemon/      # Separate binary: eBPF-based exec format error monitor daemon

internal/controller/
├── operator/            # Operator mode: reconciles ClusterPodPlacementConfig, manages operand deployments
├── podplacement/        # Operand mode: PodReconciler, webhook, GlobalPullSecretSyncer, CEL evaluator
│   └── metrics/         # Prometheus metrics definitions per component
├── podplacementconfig/  # PodPlacementConfig webhook validation
└── enoexecevent/        # ENoExecEvent handler controller + eBPF daemon
    ├── handler/         # Kubernetes controller processing ENoExecEvent CRs
    └── daemon/          # eBPF tracepoint attacher, event storage

pkg/
├── image/               # Container image inspection: registry auth, manifest parsing, caching
├── informers/           # ClusterPodPlacementConfig singleton informer (in-memory cache)
├── models/              # Pod model abstraction — scheduling gate helpers, node affinity setters
├── testing/             # Test builders (fluent K8s object builders), framework utils, fake registry
└── utils/               # Constants (labels, gate names), helper functions, runtime utilities

config/                  # Kustomize: CRDs, RBAC, webhooks, manager, prometheus, samples, standalone
hack/                    # Build scripts, upgrade automation
deploy/                  # Additional deployment manifests
bundle/                  # OLM bundle (CSV, CRDs, metadata)
```

## Key Domain Concepts

The Multiarch Tuning Operator solves one problem: **ensuring pods land on nodes with compatible CPU architectures**. It uses Kubernetes scheduling gates (KEP-3521) to pause pod scheduling, inspects container images to determine supported architectures, then sets `kubernetes.io/arch` nodeAffinity before releasing the gate.

**Primary workflow** (pod placement):
1. User creates `ClusterPodPlacementConfig` CR named `"cluster"` → operator deploys pod-placement-controller + webhook
2. Webhook intercepts pod creation → adds `multiarch.openshift.io/scheduling-gate` → pod stays Pending
3. PodReconciler watches Pending pods with the gate → inspects container images via registry API
4. Determines supported architectures → sets required nodeAffinity (`kubernetes.io/arch in [amd64, arm64, ...]`)
5. Optionally sets preferred nodeAffinity via NodeAffinityScoring plugin
6. Removes scheduling gate → pod becomes schedulable

**Secondary workflow** (ENOEXEC monitoring):
1. eBPF daemon on each node monitors exec format errors via tracepoint
2. Creates `ENoExecEvent` CRs when architecture mismatches are detected at runtime
3. Handler controller processes events and publishes Kubernetes events on affected pods

**Two-level configuration**:
- `ClusterPodPlacementConfig` (cluster-scoped, singleton named `"cluster"`) — global configuration
- `PodPlacementConfig` (namespace-scoped, priority-based) — per-namespace overrides with CEL rules

## Component/Controller Details

**Framework**: All controllers use `controller-runtime` (sigs.k8s.io/controller-runtime). The operator controller additionally uses `library-go` for the event recorder (`github.com/openshift/library-go/pkg/operator/events`).

**Startup**: `cmd/main.go` → `bindFlags()` → `validateFlags()` (ensures exactly one mode) → builds leader election ID per mode → creates `ctrl.Manager` → dispatches to `Run*()` functions.

| Controller | Mode Flag | Leader Election ID | Key Behavior |
|-----------|-----------|-------------------|-------------|
| `ClusterPodPlacementConfigReconciler` | `--enable-operator` | `operator-208d7abd.multiarch.openshift.io` | Manages operand Deployments, RBAC, webhooks, ServiceMonitor |
| `PodReconciler` | `--enable-ppc-controllers` | `ppc-controllers-208d7abd.multiarch.openshift.io` | Reconciles gated pods; `MaxConcurrentReconciles = NumCPU * 4` |
| `PodSchedulingGateMutatingWebHook` | `--enable-ppc-webhook` | N/A (webhook, not controller) | Adds scheduling gate; uses ants worker pool (16×16) for events |
| `GlobalPullSecretSyncer` | `--enable-ppc-controllers` | (same as PodReconciler) | Syncs global pull secret for image inspection |
| `ENoExecEvent handler` | `--enable-enoexec-event-controllers` | `enoexecevent-controllers-208d7abd.multiarch.openshift.io` | Processes ENoExecEvent CRs, publishes pod events |
| `PodPlacementConfig webhook` | `--enable-operator` | (same as operator) | Validates PodPlacementConfig create/update/delete |

## Resource Management

| Controller | Apply Method | Code Reference |
|-----------|-------------|---------------|
| `ClusterPodPlacementConfigReconciler` | library-go `resourceapply` functions via `utils.ApplyResources()` / `utils.ApplyResource()` | `pkg/utils/resource.go:69–120`, `internal/controller/operator/clusterpodplacementconfig_controller.go` |
| `PodReconciler` | `client.Update` for pod spec (nodeAffinity + gate removal) | `internal/controller/podplacement/pod_reconciler.go` |
| `PodSchedulingGateMutatingWebHook` | JSON patch response via `admission.PatchResponseFromRaw` | `internal/controller/podplacement/scheduling_gate_mutating_webhook.go` |
| Operand Deployments/DaemonSets | `resourceapply.ApplyDeployment()` / `resourceapply.ApplyDaemonSet()` | `pkg/utils/resource.go:170–190` |
| Operand RBAC/Services | `resourceapply.ApplyClusterRole()`, `ApplyServiceAccount()`, etc. | `pkg/utils/resource.go:80–110` |

The operator uses a `resourceapply.ResourceCache` (`pkg/utils/resource.go:35`) to track previously applied resources and avoid unnecessary updates.

## Plugins System

Plugins extend pod placement behavior. Defined in `api/common/plugins/`:

| Plugin | Scope | Purpose | Key File |
|--------|-------|---------|----------|
| `NodeAffinityScoring` | Cluster + Namespace | Adds preferred (soft) nodeAffinity with architecture weights (1–100) | `nodeaffinityscoring_plugin.go` |
| `CelArchitecturePlacement` | Namespace only | CEL-based architecture rules; replaces image inspection with rule-based placement | `celarchitectureplacement_plugin.go` |
| `ExecFormatErrorMonitor` | Cluster | Enables eBPF ENOEXEC monitoring | `execFormatErrorMonitor_plugin.go` |

**CEL plugin details**: Expressions evaluate against `self.metadata.*` only (name, namespace, generateName, labels, annotations). References to `self.spec.*` or `self.status.*` are rejected at admission time by `assertMetadataOnly()` AST walker (`celarchitectureplacement_plugin.go`). Max 1000 rules per config.

## Feature Gates

This operator does **not** use OpenShift feature gates. Plugin enablement is controlled via the `plugins` field in `ClusterPodPlacementConfig` and `PodPlacementConfig` CRDs. The CEL plugin's typed validation environment is cached at the package level via `sync.Once` to avoid rebuilding on each webhook call.

## Error Classification

| Error Type | Behavior | Code Reference |
|-----------|----------|---------------|
| Image inspection failure | Retries up to `MaxRetryCount` (5); if `fallbackArchitecture` configured, uses fallback | `pod_model.go` |
| `requeueAfterError` | Custom error type causing requeue with delay | `clusterpodplacementconfig_controller.go` |
| Webhook decode failure | Returns `admission.Errored(http.StatusBadRequest, ...)` | `scheduling_gate_mutating_webhook.go` |
| Pod already scheduled (`spec.nodeName` set) | Skipped silently | `shouldIgnorePod()` in `pod_model.go` |
| Namespace excluded | Skipped silently via namespaceSelector | `shouldIgnorePod()` |
| DaemonSet-owned pod | Skipped — DaemonSets target specific nodes | `shouldIgnorePod()` |

## Namespace Exclusion Logic

Hardcoded exclusions in `shouldIgnorePod()` (`internal/controller/podplacement/pod_model.go`):
- Operator's own namespace (`utils.Namespace()`)
- `kube-*` prefixed namespaces

Additionally, pods are always skipped if they:
- Have `spec.nodeName` set (already scheduled)
- Have a control-plane nodeSelector (`node-role.kubernetes.io/master` or `node-role.kubernetes.io/control-plane`)
- Are owned by a DaemonSet

**WARNING**: `openshift-*` and `hypershift-*` namespaces are **NOT** hardcoded exclusions. Without a properly configured `namespaceSelector`, the webhook will gate pods in these namespaces, which can block critical workloads if the pod-placement-controller is unavailable.

## OpenShift Integration Points

| Integration | How | Code Reference |
|------------|-----|---------------|
| OLM lifecycle | Bundle in `bundle/`, CSV with `replaces`/channel annotations | `bundle/manifests/` |
| Global pull secret | Synced from `openshift-config/pull-secret` | `global_pull_secret.go` |
| ServiceMonitor | Created by operator for Prometheus scraping | `internal/controller/operator/objects.go` |
| library-go events | Operator controller uses library-go event recorder | `cmd/main.go:228` |
| Image registry certs | ConfigMap name `image-registry-certificates` (flag `--registry-certificates-configmap-name`) | `cmd/main.go:355` |
| Monitoring namespace label | `openshift.io/cluster-monitoring: "true"` required for metrics | [docs/metrics.md](../docs/metrics.md) |

## Generated Code Inventory

| File/Pattern | Generator | Regenerate Command |
|-------------|-----------|-------------------|
| `api/*/zz_generated.deepcopy.go` | controller-gen | `make generate` |
| `config/crd/bases/*.yaml` | controller-gen | `make manifests` |
| `config/rbac/role.yaml` | controller-gen (from RBAC markers) | `make manifests` |
| `config/webhook/manifests.yaml` | controller-gen | `make manifests` |
| `bundle/` | operator-sdk | `make bundle VERSION=<version>` |

**NEVER** hand-edit generated files. Operand RBAC (ClusterRole for pod-placement-controller) is built programmatically in `internal/controller/operator/podplacement_objects.go:buildClusterRoleController()` — do NOT add kubebuilder RBAC markers to `pod_reconciler.go` for operand permissions.

## API Behavioral Contracts

### Singleton Constraint
`ClusterPodPlacementConfig` must be named `"cluster"` — defined as `common.SingletonResourceObjectName` in `api/common/const.go`. This constraint is documented in the CRD type definition (`api/v1beta1/clusterpodplacementconfig_types.go`) and enforced by convention; the operator reconciler expects the singleton name.

### API Versions & Conversion
- **v1alpha1**: Original version, has conversion webhook to v1beta1
- **v1beta1**: Storage version (hub). All new fields go here
- Conversion defined in `api/v1alpha1/clusterPodPlacementConfig_conversion.go` and `api/v1beta1/clusterPodPlacementConfig_conversion.go`

### Pod Processing Pipeline
1. Webhook adds scheduling gate + label `multiarch.openshift.io/scheduling-gate: gated`
2. PodReconciler filters: `status.phase=Pending` field selector on cache (`cmd/main.go`)
3. `shouldIgnorePod()` checks namespace exclusions, existing nodeAffinity, DaemonSet ownership
4. Image inspection via `pkg/image/` — supports OCI and Docker v2.2 manifests
5. Architecture intersection computed across all containers (init + regular + ephemeral)
6. Required nodeAffinity set via `SetNodeAffinityArchRequirement()`, preferred via `SetPreferredArchNodeAffinity()`
7. Labels applied: `multiarch.openshift.io/node-affinity: set`, arch-specific labels
8. Scheduling gate removed; label updated to `scheduling-gate: removed`

### PodPlacementConfig Priority
Namespace-scoped `PodPlacementConfig` resources have a `priority` field (0–255). When multiple configs match, higher priority wins. Priority uniqueness is enforced per-namespace by the webhook.

### Concurrency Model
PodReconciler uses `MaxConcurrentReconciles = runtime.NumCPU() * 4` because reconciliation is I/O-bound (image inspection involves registry network calls). The webhook uses an `ants.MultiPool` with 16 pools × 16 workers for async event publishing.

### Image Inspection Cache
`pkg/image/facade.go` provides a singleton facade (`image.FacadeSingleton()`) wrapping the image inspector and cache, shared across reconciliations. Cached by image reference to minimize registry queries.

### Ordered Deletion
When `ClusterPodPlacementConfig` is deleted, the operator performs ordered teardown: removes the MutatingWebhookConfiguration first (to stop gating new pods), waits for the PodReconciler to process remaining gated pods, then removes operand deployments. The finalizer `finalizers.multiarch.openshift.io/pod-placement` controls this lifecycle.

## Labels and Annotations

| Label/Annotation | Purpose | Set By |
|-----------------|---------|--------|
| `multiarch.openshift.io/scheduling-gate: gated/removed` | Tracks scheduling gate state | Webhook / Controller |
| `multiarch.openshift.io/node-affinity: set/overriden/not-set` | Whether arch nodeAffinity was applied | Controller |
| `multiarch.openshift.io/single-arch` | Pod images support single architecture | Controller |
| `multiarch.openshift.io/multi-arch` | Pod images support multiple architectures | Controller |
| `multiarch.openshift.io/fallback-arch` | Fallback architecture was used | Controller |
| `multiarch.openshift.io/image-inspect-error` | Image inspection failed | Controller |
| `multiarch.openshift.io/preferred-affinity-sources` (annotation) | Audit trail of preferred affinity sources | Controller |
| `multiarch.openshift.io/operand` | Marks operator-managed resources | Operator |
| `multiarch.openshift.io/exec-format-error: true/false` | ENOEXEC detected on pod | ENoExec handler |

## Design References

### Scheduling Gate Pattern (KEP-3521)
**Decision**: Use scheduling gates instead of a standalone admission controller with retry logic.
**Rationale**: Scheduling gates provide a native Kubernetes mechanism to hold pods from scheduling. This avoids race conditions between the webhook and the scheduler, and allows the PodReconciler to take time for image inspection without risking premature scheduling.
**Consequence**: Requires Kubernetes 1.26+ (scheduling gates GA). Pods remain Pending until the gate is removed.

### Single-Binary Multi-Mode Architecture
**Decision**: One binary with mutually exclusive mode flags rather than separate binaries per component.
**Rationale**: Simplifies build pipeline, OLM packaging, and image management. Each mode has its own leader election ID for safe co-existence.
**Consequence**: The operator deploys itself as operand via programmatic resource construction (not static manifests), setting the appropriate flag per Deployment.

### CEL-Based Architecture Rules
**Decision**: Use CEL (Common Expression Language) for namespace-scoped architecture placement rules (MTO-0005).
**Rationale**: CEL provides a safe, sandboxed expression language already used in Kubernetes for validation. Only `self.metadata.*` is exposed to prevent information leakage from spec/status.
**Consequence**: Rules are validated at admission time with typed environments and AST walking. Max 1000 rules per config.

## Platform Documentation

For generic OpenShift development patterns, refer to the [openshift/enhancements](https://github.com/openshift/enhancements) repository:
- `dev-guide/` — development conventions and best practices
- `guidelines/` — enhancement proposal guidelines
- `CONVENTIONS.md` — coding standards

## SME Review Recommended

- Exact image inspection behavior with mirroring configurations (ICSP, IDMS, ITMS) — see `pkg/image/helpers_glob.go`
- eBPF tracepoint attachment details in `internal/controller/enoexecevent/daemon/`
- OLM upgrade path semantics (`spec.replaces`, channel strategy)
