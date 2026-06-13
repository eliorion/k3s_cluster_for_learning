# Tailscale operator

Exposes the **asp** and **fbref** admin UIs on the tailnet — authed, reachable
off-LAN, **never internet-exposed** (no Cloudflare hostname, no public ingress).
The operator runs a per-Service proxy and publishes each annotated Service as a
tailnet device (`<name>.<your-tailnet>.ts.net`).

## Why this over a LoadBalancer / ingress

The admin UIs front a **no-auth control surface** (asp orchestrator has no auth;
fbref BFF queries the DB directly). A LAN LoadBalancer (`192.168.1.50:<port>`)
would let anyone on the home network drive them. Tailscale gates access behind
tailnet identity and never touches the public internet.

## One-time setup (manual — Flux can't do these for you)

1. **ACL** — in the Tailscale admin console (Access Controls) define the tags and
   let the operator own its proxy tag:
   ```jsonc
   "tagOwners": {
     "tag:k8s-operator": [],
     "tag:k8s":          ["tag:k8s-operator"],
   },
   ```
   Make sure your own user/devices are allowed to reach `tag:k8s` devices
   (default allow-all ACL already does; tighten as you like).

2. **OAuth client** — Settings → OAuth clients → Generate. Scopes:
   `Devices Core = write` and `Auth Keys = write`. Tag it `tag:k8s-operator`.
   Copy the client id + secret.

3. **Secret** — create the SOPS-encrypted `operator-oauth` Secret:
   ```bash
   cd infrastructure/controllers/staging/tailscale-operator
   cp operator-oauth.enc.yaml.example operator-oauth.enc.yaml
   # paste the real client id/secret into the copy, then:
   sops --encrypt --in-place operator-oauth.enc.yaml
   ```

4. **Confirm the chart version** in `release.yaml` against
   `https://pkgs.tailscale.com/helmcharts` (the pinned value is a guess until you
   run `helm search repo tailscale`). Renovate keeps it current afterward.

5. **Wire it in** — uncomment `- tailscale-operator/` in
   `../kustomization.yaml`, render-check, commit:
   ```bash
   kubectl kustomize infrastructure/controllers/staging   # must build cleanly
   git add -A && git commit && git push                   # Flux reconciles
   ```

## Reaching the UIs

Once the operator is up and the app HelmReleases carry the
`tailscale.com/expose` annotations (already set in `apps/staging/{asp,fbref}/release.yaml`):

- asp:   `https://asp-admin-ui.<your-tailnet>.ts.net`
- fbref: `https://fbref-admin-ui.<your-tailnet>.ts.net`

(Approve the new devices in the admin console if your tailnet requires it.)

## Egress proxies (cluster → tailnet) — `egress-proxies.yaml`

The reverse direction: reach a device ON the tailnet FROM the cluster via a
**local Service**. The operator runs a proxy pod (tailnet member) and points an
`ExternalName` Service at it; cluster apps dial the local name, traffic exits to
the tailnet target, port preserved.

`egress-proxies.yaml` defines the central proxy pool:

- `tailscale-proxy-00` → residential exit `100.100.98.5:8888`
  Consume at `http://tailscale-proxy-00.tailscale.svc.cluster.local:8888`.

Add more exits (`-01`, `-02`, …) by copying the block + bumping the IP. Any
namespace can consume them (the operator's proxy lives in `tailscale`).

### Scraper cutover (do LAST — after the operator + egress proxy are healthy)

Today the asp scraper dials the raw tailnet IP. Once
`tailscale-proxy-00.tailscale.svc.cluster.local:8888` resolves, repoint it —
**one line** in `apps/staging/asp/release.yaml` under `scraper.proxies`:

```yaml
      proxies:
        - name: tailscale
          # was: http://100.100.98.5:8888
          url: "http://tailscale-proxy-00.tailscale.svc.cluster.local:8888"
          sites: [leboncoin]
```

Left unchanged for now on purpose: flipping it before the egress proxy is up
would break the live scraper. Verify first:
```bash
kubectl -n tailscale get svc tailscale-proxy-00          # externalName populated by operator
kubectl -n asp run nettest --rm -it --image=curlimages/curl --restart=Never -- \
  -x http://tailscale-proxy-00.tailscale.svc.cluster.local:8888 https://api.ipify.org   # → residential IP
```

## Notes

- The admin-ui chart NetworkPolicy is egress-only, so the operator's proxy pod
  can reach the Service with no policy change.
- The `tailscale.com/expose` annotation on the app Services is **inert** until
  this operator is installed — adding it early is harmless.
- Egress proxies (`egress-proxies.yaml`) ALSO need the operator running; they
  live in the same gated dir, so they activate together.
