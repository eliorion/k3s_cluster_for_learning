# Barman Cloud Plugin (CNPG-I)

Standalone Flux unit. Installed via its **own** Flux Kustomization
(`infra-cnpg-plugin`, defined in `clusters/<env>/infrastructure.yaml`), separate
from the CNPG operator. This is deliberate: the plugin installs the
`ObjectStore` **CRD**, and the `ObjectStore` **CRs** (keycloak/asp backups) live
in the `infrastructure-controllers` and `apps` Kustomizations. Those consumers
`dependsOn: infra-cnpg-plugin`, so the CRD always exists before any CR is
applied — avoiding the Flux atomic-apply deadlock ("no matches for kind
ObjectStore").

- Chart `plugin-barman-cloud` (appVersion v0.12.0) pulled from the OCI registry
  `oci://ghcr.io/cloudnative-pg/charts` via its own `cnpg-plugin` HelmRepository,
  so it shares nothing with the operator's `cnpg` HelmRepository.
- Requires cert-manager → `infra-cnpg-plugin` `dependsOn: infra-certmanager`.
- Deployed into `cnpg-system`.
