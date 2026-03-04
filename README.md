# Home Assistant Helm Chart

This chart deploys Home Assistant using the shared dependency `lib-chart` (`0.0.11`).

## Installation

```bash
helm install home-assistant . --namespace home-automation
```

## Dependencies

- `lib-chart` (`0.0.11`) from `oci://ghcr.io/orhayoun-eevee`

Update dependencies from chart root:

```bash
helm dependency build
```

## Validation and Testing

This chart uses the reusable 5-layer validation pipeline from `build-workflow`:

1. Syntax and structure (`yamllint`, `helm lint --strict`)
2. Kubernetes schema validation (`kubeconform`) on rendered scenarios
3. Metadata and version checks (`ct lint` + version bump policy)
4. Unit and regression checks (`helm-unittest` + scenario snapshots)
5. Policy checks (`checkov`, `kube-linter`)

### CI Workflows

- Required gate: `.github/workflows/pr-required-checks.yaml`
- Release: `.github/workflows/on-tag.yaml`
- Renovate snapshot updates: `.github/workflows/renovate-snapshot-update.yaml`
- Renovate config validation: `.github/workflows/renovate-config.yaml`

Workflow references are pinned to `orhayoun-eevee/build-workflow@v0.1.15`.

### CI Trigger Matrix

- `pr-required-checks.yaml`: pull requests to `main`, merge queue (`merge_group`)
- `on-tag.yaml`: tag pushes matching `v*`
- `renovate-snapshot-update.yaml`: Renovate PRs to `main` when `values.yaml` changes
- `renovate-config.yaml`: pushes to `main` when Renovate config/workflow files change, plus manual dispatch

### Local Docker Validation

```bash
make docker-build
make deps
make snapshot-update
make ci
```

## Home Assistant Networking Note

This chart is intentionally Kubernetes-native (`ClusterIP`/`HTTPRoute`) and includes a best-effort discovery network policy profile (mDNS/SSDP egress).

Some Home Assistant integrations still require host-network behavior for full auto-discovery. This chart now supports host networking through `deployment.hostNetwork`.
Current default is `deployment.hostNetwork: true` for Home Assistant deployments.

Example:

```yaml
deployment:
  hostNetwork: true
  # Optional override. If omitted while hostNetwork=true, lib-chart renders ClusterFirstWithHostNet.
  dnsPolicy: ClusterFirstWithHostNet
```

## Metrics Note

Prometheus support is via Home Assistant's built-in `/api/prometheus` endpoint.
`metrics.enabled` is `false` by default; enable it only after configuring your auth/scrape strategy.

## Values Overview

Default highlights from `values.yaml`:

- `global.namespace: home-automation`
- Main service and container port: `8123`
- Main image: `ghcr.io/home-assistant/home-assistant:2026.2.1@sha256:db9b3d73ef5ff04c972aa929c48c85842346546a4fa3fa0af83ea878742baea6`
- Image pull policy: `Always`
- Persistence PVC: `home-assistant-config` (`10Gi`, `ReadWriteOnce`, `longhorn`)
- `metrics.enabled: false` with `/api/prometheus` path preconfigured for ServiceMonitor
- Best-effort discovery egress policy includes:
  - mDNS (`224.0.0.251:5353/UDP`)
  - SSDP (`239.255.255.250:1900/UDP`)

## Security Defaults

- Explicit `securityContext` on app container with:
  - `runAsNonRoot: true`
  - `allowPrivilegeEscalation: false`
  - `readOnlyRootFilesystem: true`
  - `seccompProfile.type: RuntimeDefault`
- Workflow permissions follow least-privilege defaults.
- Renovate snapshot workflow is locked to same-repo Renovate bot PRs.

## Test Assets

- `tests/home_assistant_contract_test.yaml`
- `tests/home_assistant_service_overrides_test.yaml`
- `tests/home_assistant_workload_overrides_test.yaml`
- `tests/scenarios/full.yaml`
- `tests/scenarios/minimal.yaml`
- `tests/scenarios/overrides.yaml`
- `tests/snapshots/*.yaml`

## Version Bump Automation

```bash
make bump VERSION=x.y.z
```

## Known Limitations

- Full Home Assistant auto-discovery parity may require host networking in some environments.
- Host networking reduces pod network isolation; enable only when needed.
- TODO: Host-network default is currently Home Assistant-specific. We still need a shared approach for multicast/mDNS and other pod-network-to-LAN discovery paths outside host networking.

## References

- https://www.home-assistant.io/
- https://www.home-assistant.io/integrations/prometheus/
