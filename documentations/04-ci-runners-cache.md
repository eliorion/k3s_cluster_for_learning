# CI: Self-hosted GitHub runners (ARC) + Nexus dependency cache

Staging-only. Two components:

- **ARC** (actions-runner-controller, `gha-runner-scale-set` mode) — ephemeral
  GitHub Actions runners for `Eliorion/asp`, scaling 0→2 pods on demand.
- **Nexus Repository OSS 3** — in-cluster PyPI proxy so pip downloads are
  cached locally between CI runs. Deployed with the `stevehipwell/nexus3`
  Helm chart (StatefulSet), which also provisions the `pypi-proxy` repo
  declaratively via its config Job — no manual Nexus setup.

## Layout

| Component | Path | Flux Kustomization |
|---|---|---|
| ARC controller | `infrastructure/controllers/base/arc/` | `infra-arc-controller` |
| Runner scale set (asp) | `infrastructure/services/staging/arc-runner-set/` | `infrastructure-services` (SOPS) |
| Nexus | `infrastructure/services/base/nexus/` + `staging/nexus/` | `infrastructure-services` (SOPS) |

The two ARC Helm charts (`gha-runner-scale-set-controller` and
`gha-runner-scale-set`) must stay on the **same version** — Renovate bumps
both, keep them aligned when merging.

## One-time setup

### 1. GitHub PAT for runner registration

Create a PAT (classic) with `repo` scope on `Eliorion/asp`, then:

```bash
# Edit infrastructure/services/staging/arc-runner-set/github-pat.enc.yaml
# Replace REPLACE_ME with the PAT, then encrypt before committing:
sops --encrypt --in-place infrastructure/services/staging/arc-runner-set/github-pat.enc.yaml
```

Never commit the file unencrypted.

### 2. Nexus root password

The chart's config Job needs the admin password as a Secret; it sets the
password and provisions the `pypi-proxy` repo + anonymous read access
automatically:

```bash
# Edit infrastructure/services/staging/nexus/nexus-root-password.enc.yaml
# Replace REPLACE_ME with a strong password, then encrypt before committing:
sops --encrypt --in-place infrastructure/services/staging/nexus/nexus-root-password.enc.yaml
```

No manual UI/REST setup — repositories live in
`infrastructure/services/base/nexus/release.yaml` under `values.config.repos`.

## Using the runners from `Eliorion/asp` workflows

Target the scale set with `runs-on: asp-runners`.

### Python (pip via Nexus proxy)

```yaml
jobs:
  build:
    runs-on: asp-runners
    env:
      PIP_INDEX_URL: http://nexus.nexus.svc:8081/repository/pypi-proxy/simple
      PIP_TRUSTED_HOST: nexus.nexus.svc
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt   # first run warms the cache
```

### Flutter (pub via actions/cache)

Nexus OSS cannot proxy pub.dev (no Dart format). Use GitHub's cache action —
it works on self-hosted runners:

```yaml
      - uses: actions/cache@v4
        with:
          path: ~/.pub-cache
          key: pub-${{ hashFiles('**/pubspec.lock') }}
```

### Container builds

The scale set runs in `dind` mode, so `docker build`/`docker push` work
inside jobs.

## Verification

```bash
flux get kustomizations            # infra-arc-controller, infrastructure-services Ready
flux get helmreleases -A           # nexus, arc-runner-set-asp Ready
kubectl -n arc-systems get pods    # controller Running
kubectl -n arc-runners get pods,autoscalingrunnerset
kubectl -n nexus get pods,pvc      # nexus-0 Running / Bound, config job Completed

# GitHub: asp repo > Settings > Actions > Runners > "asp-runners" online
# Push a workflow with runs-on: asp-runners and watch a runner pod spawn:
kubectl -n arc-runners get pods -w

# Test the pip proxy from inside the cluster:
kubectl run pip-test --rm -it --image=python:3.12 -- \
  pip install --index-url http://nexus.nexus.svc:8081/repository/pypi-proxy/simple \
  --trusted-host nexus.nexus.svc requests
```

## Sizing notes (single-node)

- Nexus: 1–2g JVM heap, 2.5Gi memory limit, 10Gi PVC. Raise
  `install4jAddVmParams` and the limit together if it OOMs.
- Runners: max 2 concurrent pods, 1Gi limit each. Bump `maxRunners` in
  `arc-runner-set/release.yaml` when the node has headroom.
