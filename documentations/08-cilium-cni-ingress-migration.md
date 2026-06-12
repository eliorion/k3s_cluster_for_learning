# Cilium CNI + Gateway migration: Flannel → Cilium (kube-proxy-free)

Goal: replace the default Talos CNI (Flannel) + kube-proxy on the live
3-node cluster with **Cilium in kube-proxy-replacement mode**, then expose
workloads (nexus first) through a bare-metal **LB-IPAM** external IP and
**Cilium Gateway API**. No cluster wipe.

Talos `v1.13.4`, Kubernetes `v1.36.1`, Cilium **`1.19.4`**. Pod CIDR
`10.244.0.0/16`, service CIDR `10.96.0.0/12` (unchanged — Cilium reuses them).

> ⚠️ This migration degrades the **pod network cluster-wide** during cutover
> (see §2). The **control plane stays up** the whole time. Do all three nodes
> in one sitting; never leave the cluster half-migrated.

> ✅ **APPLIED 2026-06-12.** Live state: Cilium 1.19.4 kube-proxy-free,
> `nexus-lb` = `192.168.1.110`, Gateway `cilium-gw` = `192.168.1.111`,
> kube-proxy/flannel gone, Longhorn healthy. The cutover hit a **deadlock**
> (Flux installed Cilium before the Talos prerequisites) — see the Incident
> section at the bottom. The runbook below is corrected for that ordering.

Config is **talhelper-managed**: edit `bootstraping/talconfig.yaml`, render,
apply. Commands assume repo root and:

```bash
export TALOSCONFIG=bootstraping/talosconfig
export KUBECONFIG=bootstraping/kubeconfig
export SOPS_AGE_KEY_FILE=clusters/staging/age.agekey   # talhelper decrypts talsecret
```

---

## 1. Architecture — who owns what

**Flux manages everything; Talos installs no CNI** (`cniConfig.name: none`).

| Layer | Owner | Where |
|---|---|---|
| **Cilium CNI** (agent, operator, datapath, Hubble) | **Flux `HelmRelease`** | `infrastructure/controllers/base/cilium/release.yaml` (chart `1.19.4`, Renovate-pinned) |
| Gateway API CRDs (**v1.4.1**) | **Talos `extraManifests`** (URLs) | `bootstraping/talconfig.yaml`; CRDs carry no secrets |
| LB-IPAM pool + L2 announcement policy | **Flux** | `…/base/cilium/config/pool.yaml` |
| Gateway + `HTTPRoute` (nexus) | **Flux** | `…/base/cilium/config/gateway.yaml` |

Two Flux Kustomizations (`clusters/staging/infrastructure.yaml`):
`infra-cilium` (the chart) and `infra-cilium-config` (the CRs, `dependsOn:
infra-cilium` so the CRDs the chart installs exist first).

### Why Flux-managed (not Talos inlineManifest)

The chicken-and-egg — no CNI → no Flux → no CNI — only bites on a **full
cold-start** (all nodes down, Cilium never applied). It does not bite here:
this is a live migration (Flannel still routing while Flux installs Cilium),
and node *rebuilds* self-heal (the rebuilt node joins NotReady, the
cluster-wide Cilium DaemonSet tolerates NotReady, schedules onto it, installs
the CNI → Ready). Only an all-nodes-cold disaster needs one manual
`cilium install`. Bought back: clean GitOps, Renovate upgrades, **zero key
material in git** (the inlineManifest alternative embeds a Hubble CA in a
~2300-line committed blob — rejected).

---

## 2. Why "one node at a time" is not an isolated rolling swap

CNI is a **cluster-wide datapath**, not a per-node knob. Flannel (VXLAN) and
Cilium (eBPF) datapaths do not interop. While nodes are mixed:

- same-node pod ↔ pod: works
- **cross-node pod ↔ pod across the Flannel/Cilium boundary: broken**
- Services: inconsistent (kube-proxy present on some nodes, gone on others)

The whole pod network is degraded until all three converge on Cilium.

**Safety net:** the control plane is **host-network**, not CNI: `etcd`,
`kube-apiserver`, `talosd`, `kubelet`, and the VIP `192.168.1.100` keep
working. `talosctl` and `kubectl` (via the VIP) stay alive throughout — the
recovery anchor, and why this is drivable node-by-node.

**Cilium grabs CNI globally once its DaemonSet lands.** Cilium's default
`cni.exclusive: true` makes each agent seize `/etc/cni/net.d` on its node,
evicting the Flannel conf there. You do not get fine-grained "node X Flannel,
node Y Cilium" control — the DaemonSet rollout decides. Plan for fast
convergence.

---

## 3. The per-node files — what changes, what doesn't

talhelper renders the 3 machine configs from one `talconfig.yaml`. Per-node
hardware differences are `nodes[]` fields; **the migration touches none of
them** — they live alongside the shared cluster settings, and talhelper
guarantees the shared block can't drift across nodes.

| Field | node-1 (AMD) | node-2 (Intel) | node-3 (Intel) |
|---|---|---|---|
| `ipAddress` | `192.168.1.101` | `192.168.1.102` | `192.168.1.103` |
| `installDisk` | `/dev/nvme0n1` | `/dev/sda` | `/dev/sda` |
| `talosImageURL` schematic | `…65cf8364…` (amd-ucode) | `…36cd6536…` (intel-ucode) | `…36cd6536…` |
| `hostname` | `staging-controlplane-1` | `staging-controlplane-2` | `staging-controlplane-3` |

> Node names are **hyphenated** (k8s rejects underscores; Talos sanitized the
> old `HostnameConfig` underscores). talhelper enforces this.

### The migration edits in `talconfig.yaml` (shared, render once)

```yaml
cniConfig:
  name: none                 # Talos installs no CNI; Cilium comes from Flux
# in the cluster patch:
cluster:
  proxy:
    disabled: true           # Cilium does kube-proxy replacement (KubePrism :7445)
  extraManifests:
    - …gateway-api…/v1.4.1/standard-install.yaml      # Gateway/HTTPRoute/… CRDs
    - …gateway-api…/v1.4.1/…/tlsroutes.yaml           # experimental TLSRoute CRD
```

KubePrism is already on (`machine.features.kubePrism.port: 7445`) — exactly
what the chart's `k8sServiceHost: localhost` / `k8sServicePort: 7445` need.

Render + validate:

```bash
cd bootstraping
SOPS_AGE_KEY_FILE=../clusters/staging/age.agekey talhelper genconfig --offline-mode
for n in 1 2 3; do talosctl validate \
  --config clusterconfig/Homelab_staging-staging-controlplane-$n.yaml --mode metal; done
```

---

## 4. Cilium chart values

Live in `infrastructure/controllers/base/cilium/release.yaml` (`spec.values`).
Key settings and why:

| Value | Why |
|---|---|
| `ipam.mode: kubernetes` | reuse Talos per-node podCIDR (`10.244.0.0/16`) |
| `kubeProxyReplacement: true` + `k8sServiceHost: localhost`, `k8sServicePort: 7445` | eBPF service LB via KubePrism |
| `securityContext.capabilities.{ciliumAgent,cleanCiliumState}` | Talos requires explicit caps (Sidero docs) |
| `cgroup.autoMount.enabled: false`, `hostRoot: /sys/fs/cgroup` | Talos manages cgroups |
| **`bpf.masquerade: false`** | `hostDNS.forwardKubeDNSToHost: true` + bpf.masquerade breaks CoreDNS |
| `routingMode: tunnel` / `vxlan` | safe default; switch to native later (§8) |
| `gatewayAPI.enabled: true` | L7 ingress (CRDs from Talos extraManifests) |
| `l2announcements.enabled: true`, `externalIPs.enabled: true` | bare-metal LB-IPAM |
| `hubble.tls.auto.method: cronJob` | in-cluster cert-gen — no cert churn on Flux reconcile, no key material in git |
| `operator.replicas: 2`, `rollOutCiliumPods`, `k8sClientRateLimit` | HA + lease headroom for L2 |

Optional later: `encryption.type: wireguard` (untrusted nets), Hubble
metrics → monitoring tier, native routing (§8).

---

## 5. Migration — execute

> Order matters: Talos drops kube-proxy/Flannel from its managed set but does
> **not** delete the running DaemonSets. Cilium installs alongside Flannel,
> seizes CNI, then you delete the leftovers and reboot to clear iptables.

- [ ] **0. Pre-flight gates**

      ```bash
      talosctl -n 192.168.1.101 etcd members        # 3 members
      talosctl -n 192.168.1.101 health
      talosctl -n 192.168.1.101 etcd snapshot bootstraping/etcd-pre-cilium.snapshot
      kubectl get nodes -o wide                      # 3 Ready
      ```

> 🔴 **ORDER IS LOAD-BEARING.** Apply the Talos config (Gateway CRDs +
> `cni:none` + `proxy.disabled`) and confirm the CRDs exist **before** Flux
> reconciles the cilium HelmRelease. Flux auto-installs on `git push`, so
> either (a) apply the Talos config *before* pushing the cilium manifests, or
> (b) push with `infra-cilium` **suspended** (`flux suspend kustomization
> infra-cilium infra-cilium-config`) and resume after step 1. Letting Flux
> install Cilium first — before the Gateway CRDs and before kube-proxy is
> gone — **deadlocked the live cluster on 2026-06-12** (see Incident).

- [ ] **1. Apply the Talos config FIRST** (adds Gateway CRDs via
      extraManifests, sets `cni:none` + `proxy.disabled`). Flannel + kube-proxy
      keep running — Talos stops *managing* them but doesn't delete them — so
      the cluster stays up.

      ```bash
      for n in 1 2 3; do talosctl apply-config -n 192.168.1.10$n \
        --file bootstraping/clusterconfig/Homelab_staging-staging-controlplane-$n.yaml; done
      # GATE: all 5 Gateway CRDs must exist before step 2
      kubectl get crd gatewayclasses.gateway.networking.k8s.io
      ```

- [ ] **2. Now let Flux install Cilium** (CRDs present + KubePrism in the chart
      values → install succeeds; if you pushed earlier with it suspended,
      `flux resume kustomization infra-cilium`):

      ```bash
      git add -A && git commit && git push           # if not already pushed
      flux reconcile kustomization infra-cilium --with-source
      kubectl -n kube-system rollout status ds/cilium --timeout=5m
      # sanity: KubePrism must be set, or kube-proxy removal (step 3) strands cilium
      kubectl -n kube-system get cm cilium-config \
        -o jsonpath='{.data.k8s-service-host}:{.data.k8s-service-port}{"\n"}'  # localhost:7445
      ```

- [ ] **3. Delete the leftovers Talos won't remove:**

      ```bash
      kubectl -n kube-system delete ds kube-flannel kube-proxy --ignore-not-found
      kubectl -n kube-system delete cm kube-proxy             --ignore-not-found
      ```

- [ ] **4. Clear stale kube-proxy iptables — rolling reboot, ONE node at a
      time, watch etcd quorum** (never two CPs down; doc 07 discipline).
      **Reboot the node hosting `flux-system` + CoreDNS first** — clearing its
      stale rules is what brings DNS/Flux back fastest (it was `.102` on
      2026-06-12; check with `kubectl -n flux-system get pods -o wide`):

      ```bash
      talosctl -n 192.168.1.102 reboot          # flux/coredns home — wait Ready
      kubectl get nodes
      talosctl -n 192.168.1.101 etcd members    # still 3
      # repeat .103, then .101
      ```

- [ ] **5. Restart workload pods** so they re-attach to Cilium veths:

      ```bash
      kubectl rollout restart deploy,statefulset,daemonset -A
      ```

- [ ] **6. Verify the datapath:**

      ```bash
      cilium status                          # KubeProxyReplacement: True; all green
      kubectl create ns cilium-test
      kubectl label ns cilium-test pod-security.kubernetes.io/enforce=privileged
      cilium connectivity test --test-namespace cilium-test
      kubectl delete ns cilium-test
      ```

- [ ] **7. Apply the networking CRs** (pool/Gateway) — auto once `infra-cilium`
      is healthy (`infra-cilium-config` dependsOn it), or force:

      ```bash
      flux reconcile kustomization infra-cilium-config --with-source
      ```

---

## 6. The nexus fix (and Gateway)

**Root cause nexus is unreachable:** `nexus-lb` is a `type: LoadBalancer`
Service that was served by **k3s ServiceLB (Klipper)**. Talos has no ServiceLB,
so it's stuck `<pending>` with no external IP. Cilium **LB-IPAM assigns it an
IP** from the pool — that alone fixes nexus.

LB pool — `infrastructure/controllers/base/cilium/config/pool.yaml`:

```yaml
apiVersion: cilium.io/v2            # NOTE: v2 in Cilium 1.19 (was v2alpha1)
kind: CiliumLoadBalancerIPPool
spec:
  blocks:
    - start: "192.168.1.110"        # clear of VIP .100 and nodes .101-.103;
      stop:  "192.168.1.130"        # confirm outside the DHCP pool
---
apiVersion: cilium.io/v2alpha1      # L2 policy still alpha in 1.19
kind: CiliumL2AnnouncementPolicy
spec:
  loadBalancerIPs: true
  interfaces: [^en.*]
  nodeSelector: { matchLabels: { kubernetes.io/os: linux } }
```

> 🔴 VIP `.100` (Talos built-in, ARP) and Cilium L2 (ARP) must not overlap.
> Pool `.110–.130` avoids it. Keep it outside DHCP.

Gateway + nexus L7 route — `…/config/gateway.yaml`: a `cilium`
`GatewayClass` Gateway on `:80` (gets its own pool IP) and an `HTTPRoute` in
`nexus` for hostname `nexus.staging.lan` → `nexus-lb:8081`. The docker
connector ports (5000–5002, TCP) stay on the `nexus-lb` LoadBalancer directly
(in-cluster CI keeps using `nexus.nexus.svc:500x`).

**Set before relying on it:** the `hostnames:` in `gateway.yaml` and point LAN
DNS at the Gateway's IP.

Verify:

```bash
kubectl -n nexus get svc nexus-lb                       # EXTERNAL-IP from the pool
kubectl -n kube-system get gateway cilium-gw            # PROGRAMMED=True, ADDRESS set
curl http://<nexus-lb-ip>:8081/                          # nexus UI (L4 path)
curl -H 'Host: nexus.staging.lan' http://<gateway-ip>/   # nexus via Gateway (L7)
```

---

## 7. What "production-grade" means here

1. **Secrets handled.** Cluster PKI is in `bootstraping/talsecret.sops.yaml`
   (SOPS, staging age key). The plaintext `controlplane-*.yaml`,
   `talsecret.yaml`, `talosconfig`, and `clusterconfig/` are gitignored.
   talhelper decrypts via `SOPS_AGE_KEY_FILE`. ✅
2. **No config drift.** One `talconfig.yaml` → 3 rendered configs; the shared
   cluster block cannot diverge. ✅
3. **CNI lifecycle.** Flux `HelmRelease` + Renovate-pinned `1.19.4` = automated
   upgrades, no hand-rendered blob. Caveat: all-nodes-cold DR needs one manual
   `cilium install` (§1).
4. **HA** — 3 control planes, `operator.replicas: 2`.
5. **Native routing** (post-cutover optimization, single L2 segment):

   ```yaml
   routingMode: native
   autoDirectNodeRoutes: true
   ipv4NativeRoutingCIDR: 10.244.0.0/16
   ```

6. **WireGuard** node encryption if the LAN isn't trusted; Hubble metrics →
   monitoring tier (doc 05 alerts).

---

## 8. Rollback

Control plane never depends on the CNI, so `talosctl`/`kubectl` stay up —
rollback is always drivable.

- Revert `cniConfig.name: none` and the `proxy.disabled` patch in
  `talconfig.yaml`, re-render, `apply-config` all three, `talosctl upgrade-k8s`
  → Flannel + kube-proxy return. Suspend/remove the cilium HelmRelease
  (`flux suspend hr cilium -n flux-system`; delete the DaemonSet).
- Pods already on Cilium veths need a restart to re-attach to Flannel.
- Worst case: restore `bootstraping/etcd-pre-cilium.snapshot`.

---

## 9. Gotchas

- **`forwardKubeDNSToHost: true` + `bpf.masquerade: true` breaks CoreDNS.**
  Keep `bpf.masquerade: false`. To enable eBPF masquerade later, first set
  `machine.features.hostDNS.forwardKubeDNSToHost: false`.
- **VIP vs LB-IPAM ARP collision** — pool clear of `.100`–`.103`.
- **`cni.exclusive: true`** (default) — once a Cilium agent starts on a node it
  owns `/etc/cni/net.d` there; Flannel can't coexist on that node. Expected.
- **CRD apiVersions** — `CiliumLoadBalancerIPPool` is `cilium.io/v2` in 1.19;
  `CiliumL2AnnouncementPolicy` is still `cilium.io/v2alpha1`. Gateway API
  **v1.4.1** (Cilium 1.19's required version — older v1.2 is wrong).
- **PodSecurity `enforce: baseline`** — `kube-system` is exempt (apiServer
  config), so Cilium installs cleanly; the connectivity-test namespace needs
  the `privileged` label.
- **Gateway CRDs before the chart** — `gatewayAPI.enabled: true` makes the
  chart create a `cilium` GatewayClass, which needs the CRDs. They ship via
  Talos `extraManifests` (applied at step 1), before Flux installs Cilium
  (step 2). Keep that order.
- **talhelper render uses the encrypted secret** — always pass
  `SOPS_AGE_KEY_FILE`; never regenerate `talsecret` (new PKI = dead cluster).

---

## 10. Incident & recovery (2026-06-12)

**What went wrong.** The Flux cilium manifests were pushed **before** the Talos
config was applied. Flux auto-reconciles on push, so it installed the cilium
HelmRelease while:

1. the Gateway API CRDs did not exist yet (Talos `extraManifests` not applied)
   → the chart's `cilium` GatewayClass could not be created → HelmRelease wedged
   in `Running 'install' action`;
2. an early `release.yaml` revision lacked `k8sServiceHost`/`k8sServicePort`, so
   cilium ran kube-proxy-replacement with **no KubePrism endpoint**;
3. `kube-proxy` + `kube-flannel` were still running alongside Cilium.

The KubePrism-less Cilium + coexisting kube-proxy left service VIPs unprogrammed.
The CoreDNS service `10.96.0.10:53` went `connection refused` → **cluster DNS
died** → every Flux controller `CrashLoopBackOff` → the source-controller could
not fetch git → **self-heal deadlock** (the fix was in git, but Flux couldn't
reach it). Control plane stayed up the whole time (host-network).

**Recovery order that worked** (each step is DNS/Flux-independent until the end):

1. **Restore Cilium's API path** — manual emergency override (Flux was dead):

   ```bash
   kubectl -n kube-system patch cm cilium-config --type merge \
     -p '{"data":{"k8s-service-host":"localhost","k8s-service-port":"7445"}}'
   kubectl -n kube-system rollout restart ds/cilium
   ```

   KubePrism `:7445` is host-network (always up), so agents reconnect
   independent of kube-proxy → services program → DNS recovers. (Flux later
   re-applied the identical value from git — no drift.)

2. **Apply the Talos config** (`talosctl`, host-network — works without cluster
   DNS): lands the Gateway CRDs (fetched via *node* DNS) + `cni:none` +
   `proxy.disabled`, unsticking the HelmRelease.

3. **Delete the legacy datapath:** `kubectl -n kube-system delete ds
   kube-flannel kube-proxy`.

4. **Rolling reboot, node hosting Flux/CoreDNS first** (`.102`), then `.103`,
   `.101` — clears stale kube-proxy iptables. After the first reboot the
   GitRepository went `Ready`, controllers recovered, the HelmRelease
   `UpgradeSucceeded`, and `nexus-lb`/`cilium-gw` got their pool IPs.

5. Post-reboot, outage-backlog pods were stuck in crashloop-backoff; deleting
   them (or waiting ~10 min) cleared it. Longhorn rebuilt replicas to `healthy`
   on its own.

**Prevention** (now baked into §5): apply the Talos config and verify the
Gateway CRDs **before** Flux installs Cilium, and ensure `release.yaml` carries
`k8sServiceHost: localhost` / `k8sServicePort: 7445` from the first commit.
Without KubePrism, removing kube-proxy strands Cilium with no API path — fix
`k8sServiceHost` *before* step 3, never after.
