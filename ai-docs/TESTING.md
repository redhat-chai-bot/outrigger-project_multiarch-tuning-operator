# Testing Guide

## Test Suites

| Suite | Location | Command | Framework |
|-------|----------|---------|-----------|
| Unit tests | `*_test.go` alongside source | `make unit` | Ginkgo/Gomega + envtest |
| E2E tests | `pkg/e2e/` | `make e2e` | Ginkgo/Gomega + real cluster |
| All checks + unit | — | `make test` | Includes fmt, vet, goimports, gosec, lint |

## Unit Tests

Unit tests use **Ginkgo/Gomega** with **envtest** (controller-runtime's test environment) providing a local API server.

**Run all unit tests**:
```bash
make unit
# Or locally:
NO_DOCKER=1 make unit
```

**Run specific tests**:
```bash
GINKGO_ARGS="-v --focus='should set node affinity'" make unit
```

**Test output**: Coverage reports generated in `test-unit-coverage.out`, artifacts in `ARTIFACT_DIR` (default `./_output`).

### Test Builders

The `pkg/testing/builder/` package provides **fluent builders** for Kubernetes objects used extensively in tests. Key builders:

| Builder | Creates | Key Methods |
|---------|---------|-------------|
| `NewPod()` | `corev1.Pod` | `.WithContainers()`, `.WithSchedulingGates()`, `.WithNodeSelector()` |
| `NewContainer()` | `corev1.Container` | `.WithImage()`, `.WithName()` |
| `NewClusterPodPlacementConfig()` | `ClusterPodPlacementConfig` | `.WithNamespaceSelector()`, `.WithPlugins()` |
| `NewPodPlacementConfig()` | `PodPlacementConfig` | `.WithLabelSelector()`, `.WithPlugins()`, `.WithPriority()` |
| `NewDeployment()` | `appsv1.Deployment` | `.WithReplicas()`, `.WithNamespace()` |
| `NewNodeBuilder()` | `corev1.Node` | `.WithLabels()`, `.WithName()` |
| `NewNodeAffinityBuilder()` | `corev1.NodeAffinity` | `.WithRequired()`, `.WithPreferred()` |
| `NewENoExecEvent()` | `ENoExecEvent` | `.WithNodeName()`, `.WithPodName()` |

Example usage pattern:
```go
pod := builder.NewPod().
    WithName("test-pod").
    WithNamespace("default").
    WithContainers(
        builder.NewContainer().WithImage("registry.example.com/app:latest").Build(),
    ).
    WithSchedulingGates(corev1.PodSchedulingGate{Name: utils.SchedulingGateName}).
    Build()
```

### Test Framework Utilities

`pkg/testing/framework/` provides shared test utilities:

| Utility | Purpose | File |
|---------|---------|------|
| `GetPodsWithLabel()` | Gets pods matching a label selector | `pod.go` |
| `GetPodLog()` | Retrieves container logs for a pod | `pod.go` |
| `EnsureNamespaces()` | Creates/ensures test namespaces exist | `utils.go` |
| `GetRandomNodeName()` | Gets a random node name with matching label | `node.go` |
| `ValidateCreation()` / `ValidateDeletion()` | Verifies CPPC operand lifecycle | `cluster_pod_placement_config.go` |
| `VerifyConditions()` | Asserts expected status conditions on CPPC | `cluster_pod_placement_config.go` |
| `HaveEquivalentNodeAffinity()` | Custom Gomega matcher for nodeAffinity | `node_affinity_equivalence_matcher.go` |
| `HaveEquivalentPreferredNodeAffinity()` | Custom Gomega matcher for preferred nodeAffinity | `node_affinity_equivalence_matcher.go` |
| `ApplyCRDs()` | Applies CRD manifests from kustomize path | `suites_utils.go` |

### Fake Image Infrastructure

`pkg/testing/image/fake/` provides mock implementations for image inspection:

- `fake.Inspector` — returns preconfigured architecture results
- `fake.Cache` — in-memory cache for tests
- `fake.Facade` — wraps inspector + cache
- `fake/registry/` — full mock Docker registry with auth support (htpasswd)

Tests replace the global `imageInspectionCache` in `pod_model.go` with fakes to avoid real registry calls.

## E2E Tests

E2E tests require a **running cluster** with the operator deployed.

```bash
KUBECONFIG=/path/to/kubeconfig NAMESPACE=openshift-multiarch-tuning-operator make e2e
```

**Structure**: `pkg/e2e/` contains separate suites:
- Operator lifecycle tests (CPPC creation, operand deployment)
- Pod placement tests (scheduling gate flow, image inspection, nodeAffinity)

**Prerequisites**:
- Operator deployed via `make deploy` or OLM
- ClusterPodPlacementConfig created
- Multi-arch cluster or nodes with appropriate `kubernetes.io/arch` labels

## Verify Commands

```bash
# Format, vet, imports check
make fmt
make vet
make goimports

# Lint (golangci-lint, config in .golangci.yaml)
make lint

# Security scanning (gosec)
make gosec

# Verify no uncommitted generated changes
make verify-diff

# Verify bundle determinism
make bundle-verify

# Verify snapshot determinism
make verify-snapshots
```

## CI Integration

CI pipelines run `make test` which executes all checks and unit tests. E2E tests run separately against deployed clusters. The `.ci-operator.yaml` file configures CI behavior for OpenShift CI.

## Platform Testing Conventions

For generic OpenShift testing patterns, refer to [openshift/enhancements](https://github.com/openshift/enhancements) `dev-guide/`.
