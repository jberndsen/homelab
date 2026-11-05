# HA-prometheus-connection

## Mission and teaching agreement

Connect the existing Home Assistant VM to the existing Prometheus/Grafana stack
while learning how the configuration, authentication and deployment fit together.
The owner requested this separate, manual learning plan on 2026-10-23.

Use the `teach` skill **in conversation** when this plan is resumed. The owner
executes all live steps: UI actions, commands, edits, credential creation,
installation, restarts and deployment. The assistant explains, reviews and helps
interpret results. Do not autonomously install File editor or execute this plan.
The earlier installation approval question was superseded by this learning plan;
it is not an approval to install anything.

Teach one small step at a time, usually 5–10 minutes:

1. Explain the one concept needed for the next action in plain language.
2. Ask a short prediction or recall question; adapt to the learner's answer.
3. Give one action or a short, closely related group of actions.
4. Have the owner report the result, without credentials or private file contents.
5. Compare the result with the expected evidence, explain differences, and stop
   to diagnose before adding another change.
6. Update this file's checkpoint and learning notes before the next live change.

Prefer conversation over generated HTML lessons/reference sites for this task.
Do not create a separate teaching workspace unless the owner asks. Use this
mission and the notes below to ground the `teach` session; no mission interview
is necessary unless the owner's goal changes. At each new session, ask what the
owner remembers and resume at the first incomplete step. Do not dump every
command or the whole configuration at once.

## Authoritative restart checkpoint

**Status: prepared; no monitoring implementation recorded. Next lesson: step 1.**
Baseline evidence below is dated October 23, 2026; recheck it when resuming.

- Prometheus integration is not known to be enabled in HA. No metrics account,
  token, Secret, scrape job or HA dashboard has been created by this work.
- No File editor/SSH access configured, owner-confirmed October 23. No app was
  installed. The owner must choose and perform the file-access setup later.
- Prepared reference YAML:
  [home-assistant/prometheus.yaml](../infrastructure/node-main/home-assistant/prometheus.yaml).
  It is not installed or referenced by a Kubernetes Kustomization.
- HomeWizard authentication migration completed October 23 ~20:36; smoke test
  passed. Physical Hue control/feedback passed. October 23 solar agrees with
  Enphase at display precision. Preserve working configuration and entity IDs.
- Backup history is in the [stability plan](home-assistant-stability.md#recovery-evidence--dated-verify-availability-before-use).
  Native `09bb68ea` predates P1 migration; native `ac35f78b` and the successful
  October 23 21:03 VM102 NAS backup contain the post-migration state. They are
  historical recovery points, not substitutes for a fresh backup before this
  lesson's changes. Restore remains untested; the key is in the password manager.
- Final October 23 inspection: Core 2026.10.2, Supervisor 2026.10.3 and HAOS
  18.3 confirmed after reboot. The earlier HAOS deferral is superseded.
  This plan does not update software or move devices between VLANs.
- K3s node Ready; monitoring Argo Application Synced/Healthy; five existing
  scrape jobs up=1 at inspection. Recheck current state when execution begins.
- Inspect `git status` and stage only the lesson's intended changes.

| Checkpoint field | Current value |
| --- | --- |
| Last completed step | Preparation only |
| Next single action | Explain the scrape path in step 1 |
| Live change in progress | None |
| Recovery point for the next live change | Fresh backup required; historical copies recorded in stability plan |
| File-access method | Undecided; no editor/SSH configured |
| Transport decision | Undecided; HA currently uses HTTP on the trusted LAN |
| Metrics user/token | Not created |
| Sealing certificate / verified backup | Not yet checked for this task |
| Git revision deployed | None |
| Outstanding validation | All implementation steps below |

After each step, replace these fields with exact status, evidence, next action,
rollback state and any open UI workflow. Record backup IDs, timestamps and Git
revisions, never tokens, passwords or encryption keys. A new conversation must
be able to continue from this file without the old browser/session.

## Context and boundaries

Read [repository instructions](../AGENTS.md),
[network facts](../infrastructure/network/system.md),
[node facts](../infrastructure/node-main/system.md),
[HA decisions](../infrastructure/node-main/home-assistant/system.md),
[HA operations](../infrastructure/node-main/home-assistant/README.md),
[monitoring operations](../infrastructure/node-main/ns-monitoring/README.md), and
[Argo/sealing operations](../infrastructure/node-main/ns-argo/SETUP.md)
before changing the systems they govern.

The [stability plan](home-assistant-stability.md) owns device stability, solar
history, TV troubleshooting, P1 power-failure diagnostics, Hue migration and
recovery follow-ups. Those are separate from this learning exercise. Leave the
owner's chosen October 22 solar total unchanged.

All plans belong in `plans/`; deployable configuration belongs under
`infrastructure/node-main`. Downloads, plaintext credentials and validation
output belong only in ignored repository `tmp/`; set `TMPDIR="$PWD/tmp"` for
tools that write temporary files. No plaintext secrets enter Git, chat, command
arguments, shell history, ConfigMaps or screenshots.

## Small lessons and manual execution steps

### 1. Draw the scrape path — no changes

- [ ] Owner explains the path in their own words.

Explain: HA exports a text endpoint; Prometheus periodically **pulls** it and
stores samples; Grafana queries Prometheus. Grafana does not scrape HA, and this
does not import HA's recorder history.

```text
HAOS VM 102                         K3s VM 103
192.168.30.166:8123                 192.168.30.103
/api/prometheus  <--- scrape ---   Prometheus <--- query --- Grafana
```

Ask: “If Grafana is unavailable, does Prometheus stop collecting?”
Success: owner can identify exporter, scraper, storage, dashboard and network
boundary. Explain `up=1` means scrape success, not proof that sensors are fresh.

### 2. Inspect the existing stack — read only

- [ ] Owner checks baseline and interprets one existing target.

Start cluster work with `kubectl get nodes -o wide`. Then teach the difference
between Kubernetes readiness, Argo sync status and Prometheus target health.
Use the commands in monitoring operations to inspect the `monitoring`
Application, Pods and `up` without retrieving Grafana credentials or opening a
port-forward. Existing jobs: prometheus, node-exporter, node-exporter-network,
kube-state-metrics and kubelet-resource.

Success: node Ready, monitoring Synced/Healthy, expected targets present/up.
Stop on an unhealthy baseline before changing monitoring.

### 3. Create a recovery point — one backup

- [ ] Fresh post-P1-migration backup completed and off-VM copy verified.

Teach native backup versus Proxmox VM backup and creation versus restore proof.
Owner creates a native backup containing settings/history and applicable
folders/apps, downloads into repository `tmp/`, protects the file and verifies
completion/readability with guidance. Owner retains the encryption key privately.
Record ID, version, time, file/checksum and covered data. Do not expose the key.

Success: a clearly identified post-migration backup and understood restore
procedure. An unreadable or incomplete copy blocks editing/installing/restarting.

### 4. Choose file access, then install only if chosen

- [ ] Owner understands the access and selects a supported method.
- [ ] Selected method works; no unintended exposed port.

Explain why the HA Prometheus integration requires YAML and why HA's YAML tools
page is not itself a configuration file editor. Review the official File editor
app as one option. Its manifest at preparation time (6.3.0) uses Supervisor
**ingress** and requests HA API plus read/write mounts for HA configuration,
app configurations, backup, media, share and SSL. It lists no direct host port.
Recheck its current manifest before installation; do not silently accept broader
permissions or choose an unrelated editor/SSH app.

Owner chooses the method and, if File editor, manually installs/starts it through
HA's Apps interface. Explain ingress versus opening an additional LAN port and
start-on-boot versus automatic software updates. Leave unattended updates off.

Success: owner can open the existing `configuration.yaml`. HA Core sees its
configuration at `/config`; an editor app may present the same directory under
a different mount name. Verify the existing file instead of guessing a path.
Do not edit `.storage`, add socket access or replace the existing configuration.

### 5. Read the prepared YAML before typing it

- [ ] Owner can explain each key and indentation level.

Open the prepared reference file and explain it in small pieces:

| Configuration | Meaning |
| --- | --- |
| `prometheus: !include prometheus.yaml` | Top-level entry in existing `configuration.yaml`; load the separate file's mapping |
| `namespace: homeassistant` | Prefix for metric names; unrelated to Kubernetes namespaces |
| `requires_auth: true` | Keep the endpoint authenticated |
| `filter:` / `include_entities:` | Export entity metrics for the explicitly selected entities |
| `sensor.p1_meter_power` | HA entity ID; different from the exported Prometheus metric name |
| kWh, MWh, W and kW | Different units; inspect actual metric names/units before writing queries |

The initial reference selects six P1/Envoy sensors and eight Hue light entities,
verified against the October 23 backup. Recheck entity IDs against current HA.
Explain group lights versus individual lights; do not sum both as consumption.
The integration also exports process/runtime and metadata metrics; an entity
filter is not a promise that every other metric is absent.

Ask: “Which file gets the `prometheus:` key, and why does the included file not
repeat it?” Success: owner can predict the YAML structure and selected coverage.

### 6. Make and check the small HA edit

- [ ] Owner adds the include and included file; configuration validation passes.

Owner adds the include to the **existing** configuration and creates the included
file from the reviewed reference. Review the difference before saving; preserve
other settings. Explain YAML mappings, lists and spaces with the actual lines.
Run HA's supported configuration check. Do not restart on validation failure.

Success: check passes and the owner can show the exact non-secret change.
Rollback before restart: remove only the new include/file, or restore the saved
configuration. Do not overwrite unrelated configuration or use a VM restore for
an ordinary syntax correction.

### 7. Restart HA once and smoke-test

- [ ] Owner restarts HA; P1/Envoy/Hue and Repairs/logs checked afterward.

Checkpoint **restart about to begin**, then owner restarts through HA's supported
UI. Wait for startup instead of repeatedly restarting. Check availability,
fresh P1/solar readings, preserved counters and Hue control/feedback.

Success: HA loads the integration, no new relevant errors, original functions
work. Record versions and result. If startup fails, use the chosen supported
file-access method to undo the small edit; use documented backup recovery only
if ordinary configuration rollback cannot recover it.

### 8. Decide credential transport before creating/sending a token

- [ ] Owner understands and chooses the transport arrangement.

Current HA URL is HTTP on the trusted LAN. Explain that authentication checks
who may access the endpoint while TLS protects a bearer token in transit.
Anyone possessing a token can use its account permissions. A dedicated account
helps lifecycle/revocation; it does not turn an HA token into a metrics-only token.

Choose a supported transport and trust arrangement together before sending a
long-lived token. If using HTTPS, verify hostname and certificate trust from
Prometheus; do not disable certificate verification. If deliberately retaining
trusted-LAN HTTP, record the owner's explicit decision and its scope. Do not
introduce public exposure, broad firewall rules or unauthenticated export.

Success: documented URL/scheme/trust decision, with no credential transmitted
during exploratory endpoint checks.

### 9. Create the dedicated credential manually

- [ ] Owner creates a dedicated non-admin account/token and stores it privately.

Explain user permissions, long-lived token lifecycle and revocation first.
Verify that the chosen account can access the endpoint without admin privileges;
do not promise a metrics-only scope or silently fall back to an admin token.
Owner completes account/credential UI steps themselves. Keep a recognizable
purpose/name. Save the token in the password manager and a protected ignored
local file for sealing, using a method that avoids terminal echo/history.

Success: credential available privately, no value copied into chat or Git.
Record account purpose and revocation procedure, not credentials.

### 10. Test the authenticated endpoint

- [ ] Endpoint rejects anonymous access and returns metrics with valid auth.

Teach the difference between connection failure, 401/403, missing endpoint and a
successful text exposition response. Owner tests anonymously and with the
credential through the chosen transport using a token file, never an argument.
Do not paste the response wholesale; inspect selected expected metric names,
labels, values and units. Validate access from the Prometheus network as well as
the laptop. Scope any temporary diagnostic workflow and close it afterward.

Success: auth enforced; the six sensors and intended lights are represented;
metric names/units/availability behavior are recorded for later queries.

### 11. Seal the credential for Git

- [ ] Owner verifies backed-up public certificate and creates strict SealedSecret.

Teach Kubernetes Secret encoding versus SealedSecret encryption. Follow the
existing Argo/sealing runbook: verify active certificate fingerprint against
backed-up inventory **before** sealing. Do not retrieve private keys merely to
create another credential. If backup coverage is unknown, resolve it using the
runbook before proceeding.

Proposed identity: namespace `monitoring`, Secret `home-assistant-metrics`, key
`token`. Seal with the exact verified certificate and strict scope. Owner runs
the steps, assistant reviews safe metadata. Use existing lifecycle confirmation
annotations and patterns; no plaintext Secret manifest enters Git.

Success: only ciphertext and safe metadata are committed; owner understands why
renaming namespace/Secret requires resealing. Record certificate fingerprint and
backup evidence, never the key or token.

### 12. Add and explain one Prometheus scrape job

- [ ] Owner prepares the job and read-only Secret mount in Git.

Use existing files under `infrastructure/node-main/ns-monitoring`:
`base/prometheus.yaml`, `base/prometheus-deployment.yaml` and
`base/kustomization.yaml`. Add the SealedSecret resource and a read-only token
file mount. Use `authorization.credentials_file`, not an inline credential.

Explain each field before editing: unique `job_name` (`home-assistant`),
`scrape_interval: 60s`, `metrics_path: /api/prometheus`, chosen `scheme`, target
address and TLS configuration if applicable. The HTTP target address would be
`192.168.30.166:8123`; choose the actual target after step 8. Explain the
`job`/`instance` labels and metrics labels supplied by HA.

Success: small reviewed diff, valid token path and no plaintext secrets. Keep
existing jobs, images, retention, PVCs and curated five-target dashboard intact.

### 13. Validate, commit and deploy through Argo

- [ ] Owner renders/checks configuration, reviews diff and pushes intended files.
- [ ] Argo reconciles; Prometheus ready; new target present/up.

Teach Kustomize resource inclusion, generated ConfigMap hashes, the resulting
Prometheus rollout, and Git/Argo ownership. Render and perform the appropriate
config/server dry-run checks before push. Avoid printing secret-bearing renders.
Do not manually apply an Argo-owned workload or blanket-stage the dirty workspace.

Prometheus uses Recreate with a PVC, so configuration rollout briefly pauses
scraping while preserving stored data. Record Git revision **before** deployment,
wait for Synced/Healthy and readiness, then inspect target/error and `up`.
Success: new target up=1, existing five targets still healthy, expected HA series
present. If target fails, diagnose its precise error before adding dashboards.
Rollback: owner reverts the scoped Git change and waits for Argo. Do not delete
PVCs or restart repeatedly. HA export can remain authenticated while rollback is
reviewed; revoke the credential if abandoning the connection.

### 14. Read one metric, then one time window

- [ ] Owner explains an instant value and its history in Grafana Explore.

Use the actual exported names/units observed in step 10. Start with current P1
power, then one cumulative energy counter. Explain why `rate` is appropriate for
some counters and not a power gauge, and distinguish kWh from MWh. Compare values
with HA and allow for scrape delay. No historical HA data is backfilled.

Success: owner writes/interprets one query from memory and distinguishes zero,
missing data and an old sample. Preserve entity identity when comparing series.

### 15. Build a small separate Grafana dashboard

- [ ] Owner creates and validates a basic HA dashboard; understands persistence.

Start with target health, P1 power, solar power and entity availability. Add
energy trends only after units/counter behavior are understood. Use a separate
HA dashboard; existing curated dashboard assumes exactly five infrastructure
jobs. Explain UI-only persistence versus Git provisioning; export/review a
Git-managed dashboard as a later small step following monitoring operations.

Success: panels agree with HA, meaningful units, missing data remains visible,
queries/dashboard definition recoverable through the chosen ownership model.

### 16. Observe, then design stale alerts as a separate lesson

- [ ] Observe overnight/day boundary and document limits.
- [ ] Decide whether to start the separate alerting lesson.

Installed Core source exports `entity_available` and `last_updated_time_seconds`
with the namespace prefix. Explain that last_updated describes state/attribute
updates, **not necessarily a successful device polling heartbeat**. Unchanged
solar at night and unchanged lights must not cause false stale alarms. Metrics
can disappear when unknown/unavailable; test absent-series behavior explicitly.

Before alerts: decide expected availability/freshness per device, missing-target
behavior, actual heartbeat evidence and an owner-chosen notification route.
Test firing and recovery using a scoped reversible method, without disrupting
production devices just to demonstrate failure. Alerts and delivery remain
unfinished until both paths work. VM/Proxmox host monitoring is separate scope.

## Session log and learning notes

### 2026-10-23 — preparation, not an executed lesson

- Owner wants to perform the steps manually to learn the configuration.
- Prior automatic installation request superseded; no installation authorized.
- Prepared YAML and inspected existing monitoring; no HA/Kubernetes monitoring
  change deployed. File-editing method, transport and credential decisions open.
- No learning assessment yet. Begin with step 1 and adapt pace from the owner's
  explanation; do not treat preparation as proof they understand these concepts.

Append a short record after each lesson: concept practiced, prediction/result,
what the owner explained independently, remaining confusion, exact checkpoint
and next action. Keep transient runtime evidence here and durable owner facts in
the appropriate `system.md`; do not duplicate manifests in system records.

## Primary references

Recheck version-dependent instructions when the lesson begins.

- [HA Prometheus integration](https://www.home-assistant.io/integrations/prometheus/)
- [Installed Core exporter source, 2026.10.2](https://github.com/home-assistant/core/blob/2026.10.2/homeassistant/components/prometheus/__init__.py)
- [HA splitting configuration](https://www.home-assistant.io/docs/configuration/splitting_configuration/)
- [HA backup and recovery](https://www.home-assistant.io/common-tasks/general/)
- [Official File editor app manifest](https://github.com/home-assistant/addons/blob/master/configurator/config.yaml)
- [HA token authentication](https://developers.home-assistant.io/docs/auth_api/#long-lived-access-token)
- [Prometheus scrape configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [Grafana Prometheus query editor](https://grafana.com/docs/grafana/latest/datasources/prometheus/query-editor/)
