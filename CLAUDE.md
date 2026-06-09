# k3s cluster for learning

Single-node k3s homelab (`k3s-staging`, Debian 13, 192.168.1.50) managed
entirely by **Flux GitOps** — never `kubectl apply` resources by hand;
change the YAML, commit, push, let Flux reconcile.

## Repo layout

- `clusters/staging/` — Flux entrypoints (Kustomizations pointing at the dirs below)
- `infrastructure/controllers/` — operators: cert-manager, cnpg, ARC controller
- `infrastructure/services/` — workloads: nexus, arc-runner-set, cloudflare, renovate
- `apps/`, `monitoring/` — application and monitoring tiers
- `documentations/` — numbered guides; `04-ci-runners-cache.md` covers the CI
  stack, `05-alerting.md` covers Telegram deploy-failure notifications
- Each tier uses `base/` + `staging/` (+ `production/`) kustomize overlays

## Conventions

- Secrets: SOPS-encrypted (`*.enc.yaml`, age key, see `.sops.yaml`). Never
  commit plaintext secrets.
- Chart versions are pinned; Renovate bumps them. The two ARC charts
  (`gha-runner-scale-set-controller` and `gha-runner-scale-set`) must stay
  on the same version.
- Storage class is `local-path`: no volume expansion, and StatefulSet
  `volumeClaimTemplates` are immutable — changing a PVC size requires
  deleting the StatefulSet + PVC (see doc 04 troubleshooting).

## CI stack (summary — full detail in documentations/04-ci-runners-cache.md)

- ARC runner scale set `self-hosted-arc` runs `Eliorion/asp` workflows,
  0→5 ephemeral pods, one job per pod.
- Runner pods use a **manual dind template** (not `containerMode: dind`)
  so dockerd gets `--insecure-registry` for the HTTP-only Nexus connectors.
- Nexus repos: `pypi-proxy` (8081), `docker-hub` proxy (5000), `ghcr`
  proxy (5001), `docker-cache` hosted (5002). Connector `httpPort`s must be
  unique or the config Job 400s and the port never opens.
- In-cluster registry host: `nexus.nexus.svc.cluster.local:500x`.

## Verify changes

```bash
kubectl kustomize infrastructure/services/staging   # render check before commit
flux get kustomizations
flux get helmreleases -A
```
