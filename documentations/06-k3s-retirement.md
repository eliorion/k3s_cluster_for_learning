# Retiring the k3s staging cluster

Migration checklist: move everything from the k3s node (`192.168.1.50`,
Debian, Flux path `clusters/staging`) to the Talos node
(`staging-controlplane-1`, `192.168.1.41`, Flux path `clusters/talos-staging`).
Longhorn replaces `local-path` as the storage class on Talos (single replica,
no volume-expansion limitation, snapshots/backups built in).

Work top to bottom. Stateless controllers (steps 1–2) are **copied** to
`clusters/talos-staging/` and stay on k3s until decommission — the live k3s
databases still need cert-manager and the CNPG operator for backups. Stateful
or singleton workloads (steps 3+) are genuine moves.

⚠️ **Operational constraint (discovered 2026-06-10): only ONE cluster can run
at a time** — k3s and Talos never coexist. R2 (and local dump files) are the
only data channels between them. Singleton concerns (duplicate ARC scale
sets, duplicate tunnel) are moot; cutover = "boot the other node". Each k3s
boot is purely a data-extraction window.

## Prerequisites (before moving anything)

- [ ] Longhorn verified on Talos (PVC bind, write, reboot survival — done
      during bootstrap).
- [ ] Recreate the SOPS age secret on Talos — required by any Kustomization
      with `decryption.provider: sops`:
      ```bash
      export KUBECONFIG=bootstraping/kubeconfig
      kubectl create secret generic sops-age -n flux-system \
        --from-file=age.agekey=clusters/staging/age.agekey
      ```
- [ ] Configure Longhorn `backupTarget` → Cloudflare R2 (same bucket family as
      CNPG barman). Longhorn data lives on the Talos EPHEMERAL partition via
      the `/var/lib/longhorn` bind mount: it survives reboots and upgrades but
      NOT `talosctl reset` / wipe-reinstall, and replica count is 1 — backups
      are the only redundancy.
      ⚠️ Never add a Talos `UserVolumeConfig` for Longhorn while the
      `/var/lib/longhorn` kubelet bind exists (mount masking,
      siderolabs/talos#13069).

## Order of moves

Dependency-ordered; each block is one commit/PR.

1. **cert-manager** (`infra-certmanager`) — no state, copy to talos-staging
   (do NOT remove from k3s: pruning deletes its CRDs and breaks the barman
   plugin certificates there). ✅ done 2026-06-10
2. **CNPG plugin + operator + ARC controller** (`infra-cnpg-plugin`,
   `infrastructure-controllers`, `infra-arc-controller`) — stateless, copy;
   needs `sops-age` (decryption) on Talos first. ✅ done 2026-06-10
   **Monitoring moved here too** (originally step 8): cnpg's chart needs the
   PodMonitor CRD + monitoring namespace from kube-prometheus-stack. The
   monitoring namespace needs privileged PSA labels on Talos (node-exporter).
   ✅ done 2026-06-10
3. **Databases (the only real data migration)** ✅ done 2026-06-10 —
   asp-db seeded on Talos via barman recovery from the k3s archive
   (`serverName: asp-db`), now archiving to its own path
   (`serverName: asp-db-talos`), first backup completed; row counts verified
   (listings=22800). fbref-db had zero rows on k3s → fresh initdb on Talos,
   nothing migrated. Gotcha hit: after freezing writers, the end-of-backup
   checkpoint WAL segment was never archived (no traffic → no segment
   switch) — recovery fails with "could not locate required checkpoint
   record" until you run `CHECKPOINT; SELECT pg_switch_wal();` on the source.
   Original per-cluster steps, kept for production reference:
   - [ ] Trigger + verify a fresh barman-cloud backup to R2 on k3s
         (`kubectl get backups.postgresql.cnpg.io`, check R2 object timestamps).
   - [ ] Scale app writers down on k3s (avoid writes after the backup).
   - [ ] Deploy the CNPG cluster on Talos with `bootstrap.recovery` from the
         R2 backup (see doc 03 for barman object store layout). PVCs land on
         Longhorn automatically (default class).
   - [ ] Verify row counts / latest timestamps match before deleting the k3s
         cluster resources.
4. **Apps** (`apps`, `db-migrations` + GitRepository sources,
   ghcr pull secrets) — after their databases are on Talos.
5. **Nexus** (`infrastructure-services` part 1) — 80Gi `local-path` PVC holds
   proxy caches only (pypi, docker-hub, ghcr, docker-cache). Don't migrate
   data: deploy fresh on Talos/Longhorn and let caches re-fill. CI will be
   slow on first runs.
   - [ ] Update any hardcoded `nexus.nexus.svc.cluster.local:500x` references
         if the namespace/ports change (they shouldn't).
6. **ARC** (`infra-arc-controller` + runner set) — stateless; GitHub app
   secret is SOPS-encrypted, moves with the YAML. Runner pods need the
   insecure-registry dind template to still resolve the Nexus connectors.
   - [ ] Suspend the k3s runner set first so two scale sets don't register
         for `Eliorion/asp` simultaneously.
7. **Cloudflare tunnel** — cutover moment for external traffic.
   - [ ] Deploy cloudflared on Talos (same SOPS credentials), verify it
         connects, then remove the k3s deployment. Tunnel tolerates two
         replicas briefly; keep the overlap window short.
8. **Monitoring** — already copied at step 2 (CRD dependency). Both clusters
   alert to the same Telegram group until k3s is decommissioned.
9. **Renovate** — stateless, plain move.

## Decommission k3s

- [ ] All stateful/singleton entrypoints moved; only `flux-system/` and the
      copied controller entrypoints remain in `clusters/staging/` — safe to
      delete with the cluster.
- [ ] Final fresh R2 backups exist for both CNPG clusters (belt & braces).
- [ ] Power off the k3s node.
- [ ] GitHub repo → Settings → Deploy keys: delete the staging key
      (`flux-system-main-flux-system-./clusters/staging`).
- [ ] Delete `clusters/staging/` from the repo.
- [ ] Remove the k3s entry from local kubeconfigs.

## Afterwards (separate task)

Rename `clusters/talos-staging` → `clusters/staging`: re-run
`flux bootstrap github` with the new `--path` (new deploy key, new
`gotk-sync.yaml` path) and delete the old path + key. Cosmetic — do it only
once everything is stable.
