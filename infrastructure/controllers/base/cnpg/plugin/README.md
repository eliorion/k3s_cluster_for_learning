# Barman Cloud Plugin (CNPG-I)

Installed via Flux `HelmRelease` (`release.yaml`) from the existing `cnpg`
HelmRepository — chart `plugin-barman-cloud` (appVersion v0.12.0), into
`cnpg-system`. Provides the `ObjectStore` CRD and the backup sidecar injector.

Requirements (already satisfied by this repo):
- CNPG operator >= 1.26 (chart 0.28.2 → operator 1.29.1).
- cert-manager — added under `infrastructure/controllers/base/cert-manager`,
  health-gated by the `infra-certmanager` Flux Kustomization, which
  `infrastructure-controllers` depends on. The plugin creates a cert-manager
  `Certificate`, so cert-manager must be ready first.
