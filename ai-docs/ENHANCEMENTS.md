# Enhancements & Design Documents

## Local Design Documents

| ID | Title | Status | Link |
|----|-------|--------|------|
| MTO-0001 | Multiarch Manager Operator | Implemented | [docs/enhancements/MTO-0001.md](../docs/enhancements/MTO-0001.md) |
| MTO-0002 | Namespace-scoped PodPlacementConfig | Implemented | [docs/enhancements/MTO-0002-local-pod-placement.md](../docs/enhancements/MTO-0002-local-pod-placement.md) |
| MTO-0003 | Support for non-OKD K8S/CRI-O clusters | Implemented | [docs/enhancements/MTO-0003-support-non-ocp-clusters.md](../docs/enhancements/MTO-0003-support-non-ocp-clusters.md) |
| MTO-0004 | eBPF-based ENOEXEC monitoring | Implemented | [docs/enhancements/MTO-0004-enoexec-monitoring.md](../docs/enhancements/MTO-0004-enoexec-monitoring.md) |
| MTO-0005 | CEL Architecture Placement Plugin | Implemented | [docs/enhancements/MTO-0005-architecture-rules-plugin.md](../docs/enhancements/MTO-0005-architecture-rules-plugin.md) |

## Upstream OpenShift Enhancement

| Title | Status | Link |
|-------|--------|------|
| Multiarch Manager Operator | Implemented | [openshift/enhancements: multiarch-manager-operator.md](https://github.com/openshift/enhancements/blob/master/enhancements/multi-arch/multiarch-manager-operator.md) |

## Related Upstream KEPs

| KEP | Title | Relevance |
|-----|-------|-----------|
| KEP-3521 | Pod Scheduling Readiness | Core mechanism — scheduling gates that MTO uses to gate pods before image inspection |
| KEP-3838 | Pod Mutable Scheduling Directives | Enables MTO to mutate nodeAffinity on gated pods |

## Other Documentation

| Document | Description |
|----------|-------------|
| [docs/metrics.md](../docs/metrics.md) | Prometheus metrics reference and example queries |
| [docs/ocp-release.md](../docs/ocp-release.md) | OCP release process notes |
| [docs/support-non-okd.md](../docs/support-non-okd.md) | Non-OKD cluster support documentation |
| [docs/alerts/](../docs/alerts/) | Alert runbook documentation for all components |
