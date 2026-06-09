# Alerting: Telegram notifications on Flux deploy failures

Staging-only. Get a phone push when a Helm chart (or any Flux resource) fails
to reconcile. Two complementary paths, same Telegram bot:

- **Flux notification-controller** — sends Flux *events* to Telegram. The
  message carries the Helm error reason (`upgrade failed: timed out
  waiting...`), so it's the one you read while debugging.
- **kube-prometheus-stack Alertmanager** — fires on *metrics*. The message is
  label-only (which release, which namespace), but you get Grafana history,
  grouping and silences. Reuses the already-running stack.

Both deliver to the same Telegram group. WhatsApp was rejected — no native
Flux/Alertmanager provider, would need a Twilio middleman.

## Files

All under `monitoring/configs/staging/` (the `monitoring-configs`
Kustomization has SOPS decryption), plus one HelmRelease edit:

| Path | Purpose |
|---|---|
| `flux-alerts/provider.yaml` | `Provider` telegram, `channel` = chat id |
| `flux-alerts/alert.yaml` | `Alert` `helm-failures`, severity `error` |
| `flux-alerts/telegram-token.enc.yaml` | bot token, SOPS-encrypted, ns `flux-system` |
| `flux-am/podmonitor.yaml` | scrape flux-system metrics (port `http-prom`) |
| `flux-am/prometheusrule.yaml` | alerts `FluxHelmReleaseFailed` / `FluxKustomizationFailed` |
| `flux-am/telegram-am-secret.enc.yaml` | bot token, SOPS-encrypted, ns `monitoring` |
| `controllers/base/kube-prometheus-stack/release.yaml` | `alertmanager.config` + secret mount |

## How it works

```
HelmRelease fails
   |
   |-- notification-controller --(event w/ error text)--> Telegram  (instant)
   |
   `-- gotk_reconcile_condition{Ready=False}
          |
        PodMonitor scrapes flux-system :8080/metrics
          |
        Prometheus rule FluxHelmReleaseFailed (for: 5m)
          |
        Alertmanager telegram receiver -----------------> Telegram  (after 5m)
```

### Path A — notification-controller

`Provider` references secret `telegram-bot-token` (key `token`) and a
`channel` = numeric chat id. `Alert` watches `HelmRelease *` in `flux-system`,
`asp`, `monitoring` and `Kustomization *` in `flux-system`, severity `error`.

### Path B — Alertmanager

- **PodMonitor** carries label `release: kube-prometheus-stack` (required by
  Prometheus `podMonitorSelector`) and scrapes pods labelled
  `app.kubernetes.io/part-of: flux` in `flux-system` on port `http-prom`
  (8080). Without it Prometheus has zero `gotk_*` series.
- **PrometheusRule** (same `release` label, matches `ruleSelector`) fires on
  `gotk_reconcile_condition{type="Ready",status="False",kind="HelmRelease"} == 1`
  after `for: 5m`.
- **Alertmanager** uses the chart's global `alertmanager.config` (not an
  `AlertmanagerConfig` CRD — the cluster `matcherStrategy: OnNamespace` would
  scope a CRD route to a namespace label the metric alert may not carry). The
  bot token is mounted from secret `alertmanager-telegram` via
  `alertmanagerSpec.secrets` and referenced as
  `bot_token_file: /etc/alertmanager/secrets/alertmanager-telegram/token`, so
  the token never lands in Git in plaintext. `chat_id` is inline (low
  sensitivity). The default Watchdog alert is routed to a `blackhole` receiver.

## Setup (one-time secrets)

Token is never committed in plaintext. Two secret files hold the same bot
token, one per namespace.

1. **Create bot**: Telegram `@BotFather` → `/newbot` → copy token.
2. **Get chat id**: add the bot to your group (or DM it), send a message, then
   `curl -s "https://api.telegram.org/bot<TOKEN>/getUpdates"` and read
   `"chat":{"id":...}`. Group ids are negative (`-100...`); DM ids positive.
   Empty `result` usually = bot privacy mode (DM the bot, or send a `/command`
   / @-mention in the group), or a stale webhook (`getWebhookInfo` →
   `deleteWebhook`).
3. **Fill placeholders**: `REPLACE_BOT_TOKEN` (both `*.enc.yaml`),
   `REPLACE_CHAT_ID` (`provider.yaml` channel — quoted string;
   `release.yaml` chat_id — bare number).
4. **Encrypt** (path matches the staging `*.enc.yaml` SOPS rule):
   ```bash
   sops -e -i monitoring/configs/staging/flux-alerts/telegram-token.enc.yaml
   sops -e -i monitoring/configs/staging/flux-am/telegram-am-secret.enc.yaml
   ```
5. **Verify + commit**: `grep ENC` both files before `git add`. Plaintext token
   in Git = leaked → rotate via BotFather.

## Verify

```bash
flux reconcile kustomization monitoring-configs -n flux-system
flux get alerts,providers -n flux-system
# Path B scrape must return series, not empty:
kubectl -n monitoring exec sts/prometheus-kube-prometheus-stack-prometheus -c prometheus -- \
  promtool query instant http://localhost:9090 'gotk_reconcile_condition{kind="HelmRelease"}'
```

End to end: point a throwaway HelmRelease at a bad chart version, commit.
Expect an instant notification-controller message (with error text) and an
Alertmanager message after 5m. Revert.

## Troubleshooting

- **No Telegram message, Path A**: `kubectl logs deploy/notification-controller
  -n flux-system`. Wrong token/chat id → 400/403 from Telegram API. Bot must be
  a group member.
- **No Telegram message, Path B**: check the alert is firing in Prometheus
  (`gotk_reconcile_condition` series exist → PodMonitor working), then
  Alertmanager UI / `kubectl logs alertmanager-...-0`.
- **Blank namespace in Path B message**: the metric label is `namespace`, not
  `exported_namespace`. Swap it in `prometheusrule.yaml` annotations/`expr`.
- **PodMonitor ignored**: missing `release: kube-prometheus-stack` label, or
  port name mismatch (must be `http-prom`).
