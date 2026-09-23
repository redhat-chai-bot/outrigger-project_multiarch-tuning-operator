# Development Guide

## Prerequisites

- **Go**: 1.26.7 (see `go.mod`)
- **CGO**: Required (`CGO_ENABLED=1`) — image inspection uses `containers/image` which depends on `gpgme`
- **System packages**: `gpgme-devel` (RHEL/Fedora) or `libgpgme-dev` (Debian/Ubuntu)
- **Container runtime**: Podman (preferred) or Docker
- **For multi-arch builds**: `qemu-user-static` package

## Build & Test

```bash
# Build binary locally (CGO required)
make build

# Run all checks + unit tests (fmt, vet, goimports, gosec, lint, unit tests)
make test

# Unit tests only
make unit

# Lint only
make lint

# Security scanning
make gosec
```

**Containerized vs local execution**: By default, builds and tests run inside a container image (`BUILD_IMAGE`). To run locally:
```bash
NO_DOCKER=1 make test
# Or add to .env file (see dotenv.example):
echo "NO_DOCKER=1" >> .env
```

**Run a single test**:
```bash
GINKGO_ARGS="-v --focus='your test pattern'" make unit
```

## Common Tasks

### Add a New Plugin

1. Define plugin struct in `api/common/plugins/` — embed `BasePlugin`, implement `Name()` method
2. Add plugin constant to `api/common/const.go`
3. Register in `api/common/plugins/base_plugin.go` plugin registry
4. Add plugin handling to `internal/controller/podplacement/pod_model.go` and reconciler
5. Run `make generate && make manifests` to regenerate deepcopy and CRDs
6. Add validation to `internal/controller/podplacementconfig/podplacementconfig_webhook.go`

### Modify the ClusterPodPlacementConfig API

1. Edit types in `api/v1beta1/clusterpodplacementconfig_types.go`
2. Update conversion functions in both `api/v1alpha1/` and `api/v1beta1/`
3. Run `make generate` (deepcopy) then `make manifests` (CRDs, RBAC, webhooks)
4. Update `internal/controller/operator/clusterpodplacementconfig_controller.go` if the field affects operand behavior
5. Run `make bundle VERSION=<version>` to update the OLM bundle
6. Run `make verify-diff` to ensure no uncommitted generated changes

### Add Operand RBAC Permissions

Operand RBAC is built **programmatically** — do NOT use kubebuilder RBAC markers on operand controllers:
- Pod placement controller: `internal/controller/operator/podplacement_objects.go` → `buildClusterRoleController()`
- Pod placement webhook: `internal/controller/operator/podplacement_objects.go` → `buildClusterRoleWebhook()`
- ENoExec handler: `internal/controller/operator/enoexecevent_objects.go`

Only the **operator** controller (manager) uses kubebuilder RBAC markers → `config/rbac/role.yaml`.

### Deploy to a Cluster

```bash
# Build and push image
make docker-build IMG=<registry>/multiarch-tuning-operator:tag
make docker-push IMG=<registry>/multiarch-tuning-operator:tag

# Install CRDs and deploy
make install
make deploy IMG=<registry>/multiarch-tuning-operator:tag

# Enable the operand (creates the singleton CR)
kubectl apply -f config/samples/

# Undeploy
make undeploy
make uninstall
```

### OLM Bundle Operations

```bash
# Generate bundle
make bundle VERSION=<version>

# Verify bundle is deterministic (no diff after regeneration)
make bundle-verify

# Build and push bundle + catalog images
make bundle-build BUNDLE_IMG=<registry>/multiarch-tuning-operator-bundle:<version>
make bundle-push BUNDLE_IMG=<registry>/multiarch-tuning-operator-bundle:<version>
make catalog-build CATALOG_IMG=<registry>/multiarch-tuning-operator-catalog:<version>
make catalog-push CATALOG_IMG=<registry>/multiarch-tuning-operator-catalog:<version>
```

### Multi-Architecture Image Build

```bash
# Requires qemu-user-static
make docker-buildx IMG=<registry>/multiarch-tuning-operator:tag

# Prevent buildx instance cleanup between builds
touch .persistent-buildx
```

## Environment Variables

Configure via `.env` file at repo root (see `dotenv.example`):

| Variable | Default | Purpose |
|----------|---------|---------|
| `NO_DOCKER` | unset | Set `1` to run builds/tests locally instead of containerized |
| `FORCE_DOCKER` | unset | Set `1` to force Docker over Podman |
| `BUILD_IMAGE` | (from Makefile) | Override builder container image |
| `RUNTIME_IMAGE` | (from Makefile) | Override runtime base image |
| `ARTIFACT_DIR` | `./_output` | Test output directory |
| `KUBECONFIG` | (from env) | Cluster kubeconfig for E2E tests |
| `NAMESPACE` | `default` (code), typically `openshift-multiarch-tuning-operator` for E2E | Operator namespace |
| `GINKGO_ARGS` | unset | Extra Ginkgo arguments (e.g., `--focus`, `--skip`) |

## Vendoring

This project uses Go vendoring (`GOFLAGS=-mod=vendor`). After modifying dependencies:
```bash
make vendor
make verify-diff  # Ensure no uncommitted changes
```

## Common Mistakes

1. **Adding kubebuilder RBAC markers to operand controllers** — operand RBAC is programmatic in `internal/controller/operator/`. Markers on `pod_reconciler.go` would over-provision the operator ServiceAccount.
2. **Forgetting `make manifests` after API changes** — CRDs, webhook configs, and RBAC roles are all generated. Tests will pass but deployment will fail with stale CRDs.
3. **Editing generated files** — `zz_generated.deepcopy.go`, CRD YAMLs, and bundle manifests are all generated. Use `make generate`, `make manifests`, `make bundle`.
4. **Running without CGO** — `CGO_ENABLED=0` builds will fail at link time due to `gpgme` dependency from `containers/image`.
5. **Testing without the scheduling gate** — unit tests in `podplacement/` use envtest; the scheduling gate must be present on test pods. Use `pkg/testing/builder/` fluent builders which handle this.
6. **Assuming namespace exclusions** — only `kube-*` and the operator namespace are hardcoded. Test scenarios must configure `namespaceSelector` for OpenShift namespaces.

## Platform Conventions

For generic OpenShift development patterns, refer to [openshift/enhancements](https://github.com/openshift/enhancements):
- `dev-guide/` — development conventions
- `CONVENTIONS.md` — coding standards
