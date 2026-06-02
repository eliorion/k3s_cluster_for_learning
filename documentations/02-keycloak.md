# Keycloak — Identity & Scaling

Keycloak is the identity platform: SSO for the internal IT team and customer
accounts (CIAM) for the apps. Chosen over edge-only auth (Cloudflare Access)
because Access gates *reach*, it does not *manage* customer identities.

## Deployment shape

```
CNPG Cluster keycloak-db ──auto──▶ Secret keycloak-db-app (random password)
                                          │ secretKeyRef (username/password)
                                          ▼
                                   Keycloak StatefulSet (start, prod mode)
```

- **DB:** `infrastructure/controllers/base/keycloak/database/cluster.yaml` — CNPG
  `Cluster`. CNPG generates the `keycloak-db-app` secret (random password); no
  credential lives in git. Keycloak connects to the `keycloak-db-rw` service.
- **App:** `infrastructure/controllers/base/keycloak/app/` — `start` (production)
  mode. DB creds from the CNPG secret, initial admin from the SOPS-encrypted
  `keycloak-secret`. Namespace `identity`.
- **Single replica** today. Keycloak itself is stateless (all durable state in
  Postgres), so the pod has no PVC.

### Why StatefulSet at 1 replica

Keycloak is stateless, so a Deployment would work today. StatefulSet is kept
deliberately: scaling to 2+ replicas needs StatefulSet semantics (stable network
identity + ordered rolling updates) for Infinispan/JGroups session clustering.
`kind` cannot be mutated in place, so starting as a StatefulSet avoids a
delete/recreate migration later. The `keycloak-discovery` headless service is the
clustering hook — keep it.

## Scaling path (in order)

1. **Vertical first.** One well-resourced replica handles thousands of
   users/logins. Raise CPU/memory before adding pods — cheapest, no clustering
   risk.
2. **Horizontal (replicas 2+)** for HA + throughput. Bumping `replicas` alone
   gives a broken cluster (split-brain sessions). You must also add:
   - `KC_CACHE: ispn`
   - `KC_CACHE_STACK: kubernetes`
   - `JAVA_OPTS_APPEND: -Djgroups.dns.query=keycloak-discovery.identity.svc.cluster.local`
   - keep the `keycloak-discovery` headless service.
   At very large scale, offload sessions to an external Infinispan cluster.
3. **Database:** raise CNPG `instances: 1 → 3` for HA (see `asp-db` for the
   pattern).
4. **Identity model** (independent of replica count): realm-per-tenant or shared
   realm + groups, federate the IT team (LDAP / Google Workspace via SCIM),
   self-registration + social login for clients.

## Operational notes

- **Bootstrap admin** comes from the SOPS `keycloak-secret` and is only the
  *initial* admin. After first login: create a personal admin, then disable the
  bootstrap account.
- **Hostname:** not publicly exposed yet (no Cloudflare tunnel route). When
  exposing, add a `cloudflared` ingress entry and set a fixed `KC_HOSTNAME`
  (e.g. `https://keycloak.eliorion.fr`), then drop `KC_HOSTNAME_STRICT: "false"`.
- **Flux ordering:** the CNPG `Cluster` depends on the CNPG operator being
  healthy. First reconcile may error, then self-heal on retry.
- **Backups:** `keycloak-db` is backed up to Cloudflare R2 (PITR, daily base +
  continuous WAL, 7d retention). See `03-backups.md`.
