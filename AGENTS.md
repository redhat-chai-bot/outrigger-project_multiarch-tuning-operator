# Multiarch Tuning Operator (MTO)

**Repository**: `openshift/multiarch-tuning-operator` · **Default branch**: `main` · **Go**: 1.26.7 · **Framework**: controller-runtime + Kubebuilder

Architecture-aware pod scheduling for multi-arch OpenShift clusters. Adds scheduling gates to pods, inspects container images, and sets `kubernetes.io/arch` nodeAffinity.

## Critical Warnings

1. **NEVER** enable more than one mode flag (`--enable-operator`, `--enable-ppc-controllers`, `--enable-ppc-webhook`, `--enable-enoexec-event-controllers`) — they are mutually exclusive (`cmd/main.go:339`)
2. **NEVER** hand-edit `zz_generated.deepcopy.go` files — run `make generate`
3. **NEVER** assume `openshift-*`/`hypershift-*` namespaces are excluded — only `kube-*` and the operator namespace are hardcoded; other exclusions require `namespaceSelector` on ClusterPodPlacementConfig
4. **ALWAYS** run `make manifests` after editing API types in `api/` — CRDs, RBAC, and webhook configs are generated
5. **ALWAYS** use `make vendor` after dependency changes — project uses `GOFLAGS=-mod=vendor`

## Architecture at a Glance

| Mode | Controller | Key File |
|------|-----------|----------|
| Operator | `ClusterPodPlacementConfigReconciler` | `internal/controller/operator/clusterpodplacementconfig_controller.go` |
| PPC Controllers | `PodReconciler` + `GlobalPullSecretSyncer` | `internal/controller/podplacement/pod_reconciler.go` |
| PPC Webhook | `PodSchedulingGateMutatingWebHook` | `internal/controller/podplacement/scheduling_gate_mutating_webhook.go` |
| ENoExec | `ENoExecEvent handler` + eBPF daemon | `internal/controller/enoexecevent/handler/` |

**Singleton CR**: `ClusterPodPlacementConfig` must be named `"cluster"` (`api/common/const.go`). **API versions**: v1alpha1 (with conversion) → v1beta1 (storage version).

## Documentation

| Need | Start Here |
|------|-----------|
| Internals, integrations, contracts | [ai-docs/ARCHITECTURE.md](ai-docs/ARCHITECTURE.md) |
| Build, test, deploy, common tasks | [ai-docs/DEVELOPMENT.md](ai-docs/DEVELOPMENT.md) |
| Test suites and patterns | [ai-docs/TESTING.md](ai-docs/TESTING.md) |
| Enhancement proposals | [ai-docs/ENHANCEMENTS.md](ai-docs/ENHANCEMENTS.md) |
| Metrics reference | [docs/metrics.md](docs/metrics.md) |
| Alert runbooks | [docs/alerts/](docs/alerts/) |
| Review guidelines | [REVIEW.md](REVIEW.md) |

## Key Files

| File | Purpose |
|------|---------|
| `cmd/main.go` | Entrypoint, flag binding, mode dispatch |
| `api/v1beta1/clusterpodplacementconfig_types.go` | Primary CRD types |
| `api/v1beta1/podplacementconfig_types.go` | Namespace-scoped PodPlacementConfig CRD |
| `internal/controller/podplacement/pod_model.go` | Core pod processing logic, `shouldIgnorePod` |
| `pkg/image/inspector.go` | Container image architecture inspection |
| `pkg/utils/const.go` | Labels, annotations, scheduling gate name |
| `Makefile` | Build, test, lint, deploy targets |

## External References

- [Enhancement: MTO-0001](docs/enhancements/MTO-0001.md) · [KEP-3521: Pod Scheduling Readiness](https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/3521-pod-scheduling-readiness/README.md)
- [Platform conventions](https://github.com/openshift/enhancements) (`dev-guide/`, `guidelines/`, `CONVENTIONS.md`)
