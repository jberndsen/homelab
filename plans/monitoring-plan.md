# Monitoring deployment and teaching plan

Status: Lessons 1–3 complete; their checkpoints passed. Grafana is deployed and
verified, but Lesson 4's dashboard and checkpoint are incomplete. On 2026-10-21
the owner paused manual dashboard practice and asked to continue the plan,
with the agent to create the final dashboard later. Lesson 5 concepts have
been introduced conversationally; Grafana Pod-replacement persistence passed
on 2026-10-21. Owner explicitly skipped further Pod-replacement testing to
continue the lesson; do not repeat it. Owner requested moving the existing UI
dashboard to Git: four-panel export/provisioning deployed and verified at
`4bcf858`, preserving the separate editable learning copy.
Lesson 4 remains incomplete.

## Resume instructions

- Read AGENTS.md, relevant system records, this plan and
  [monitoring operations](../infrastructure/node-main/ns-monitoring/README.md).
  Start cluster work with `kubectl get nodes -o wide`, then check Argo/Pod health.
- Teach exclusively in conversation. Explain each small change before Git/Argo
  deployment; keep one Kubernetes resource per file. Do not create teaching files.
- Owner saved the editable **Monitoring — Learning** dashboard and its
  **Scrape health** panel. Screenshots confirm VM CPU busy (~2%), VM memory
  used (~23%) and VM root disk available (~62 GiB) panels. Memory and disk
  units/scales are verified; a subsequent browser baseline confirms CPU displays
  percent with a 0–100% scale. Media container CPU was proposed, not confirmed saved.
  Owner understands scrape health does not prove playback and that shorter
  rate windows expose brief spikes with less smoothing. Manual panel practice
  is paused. Owner has now requested Git provisioning of the existing dashboard;
  adding the remaining promised panels is still deferred.
- Conversational checks on 2026-10-21: owner correctly distinguished the
  displayed one-hour history from each point's five-minute rate window, and
  missing data from a measured zero. Initially proposed renaming the metric
  for bits/sec; after explanation, correctly converted 1,000 bytes/sec to
  8,000 bits/sec. Owner correctly identified that only the committed/provisioned
  dashboard can be recreated from Git after Grafana PVC loss. Owner also
  correctly explained that Prometheus supplies history, data persists on PVCs
  beyond Pod lifetime, and Pod readiness alone does not prove persistence.
  Do not claim the full Lesson 4 checkpoint from these answers.
- Persistence baselines captured read-only on 2026-10-21: ignored
  `tmp/monitoring-persistence-baseline.json` contains fixed-time raw historical
  samples, their fingerprint, and monitoring Pod/PVC identities. Grafana
  dashboard UID `adhnb5z` (Monitoring - Learning) loads after browser reload
  with four saved panels: Scrape health, VM CPU Busy, VM Memory Used, VM root
  disk available. Grafana replacement subsequently passed; no dashboard
  export/provisioning was performed in that baseline step. Owner subsequently skipped the Prometheus
  replacement exercise; leave it untested and do not request it again. Continue
  with resource/retention and recovery concepts, then requested moving the
  existing dashboard to Git. Preserve its four-panel scope and editable copy.
  The historical baseline is only useful within retained history.
- Preserve verified kubelet TLS, rotating Pod-bound credentials, single-node
  discovery and node-name-scoped RBAC. No insecure TLS or nodes/proxy access.
- VM network collection is deployed and verified (2026-10-21, `8190f98`).
  A netdev-only Node Exporter sidecar reads `/host/proc/1/net/dev` with netlink
  disabled and an explicit `ens18` filter. Other collectors retain `/host/proc`.
  Both endpoints remain internal; no host network/PID namespace was added.
  There are now five scrape jobs, including `node-exporter-network`.
- Lesson port-forwards are stopped. Re-establish localhost:9090 only when needed;
  Grafana is available through its ingress. Temporary artifacts/tools were
  removed from ignored root `tmp/`; set TMPDIR there for new temporary files.
- Sealing-key backup coverage was verified before sealing Grafana credentials.
  Recheck the active certificate against the verified NAS inventory before
  creating future credentials; use that exact covered certificate.

## Remaining work

- [x] Lesson 4: validate VM network collection.
- [ ] Lesson 4: finish all promised dashboard panels (agent creation deferred
  by owner), verify settings/filters/no-data behavior and pass checkpoint.
- [x] Full 5-minute CPU rates return all eight media containers (final inspection).
- [x] Lesson 5: controlled Grafana Pod replacement preserves the saved learning
  dashboard; both PVC identities/volumes remain unchanged.
- [x] Lesson 5: export/provision the existing four-panel dashboard from Git
  with stable UID `monitoring-overview`, preserving editable `adhnb5z`.
- Prometheus controlled Pod replacement: skipped by owner on 2026-10-21;
  not tested and no longer required for this guided session.
- [ ] Lesson 5: provision remaining promised panels and verify applicable
  namespace/workload filters; finish resource/retention checks and recovery
  operations. Preserve all claims; no media disruption or whole-stack outage test.
- Optional after Lesson 5: Proxmox, NAS and UPS metrics; notifications are separate.

## Completed work and validation

Detailed session history is retained in Git. Key milestones:

2026-10-21 dashboard Git provisioning (`4bcf858`): exported the current learning
dashboard through Grafana's Classic JSON export. Verified its local reference
matches the UI export after canonicalization. The committed snapshot changes
only identity/management metadata: stable UID `monitoring-overview`, title
Monitoring - Overview, null local ID, description, version and editable=false.
All four panel queries, units and layout are preserved. Generated ConfigMaps
mount the provider and JSON read-only; hashes trigger a Grafana rollout.
Standards and Spec reviews passed without findings. Render, server dry-run,
reviewed live diff and all four real-data query checks passed. Argo deployed
the commit Synced/Healthy; Grafana is Ready with zero restarts, and its overview
shows all four panels with real data in folder Homelab. The original dashboard
`adhnb5z` reloads separately with all four panels and its Edit action. Startup
logs show dashboard provisioning completed without error-level entries.
Both claims remain
Bound and all eight media Deployments remain available. Remaining promised
network/media panels and namespace/workload filters are not part of this export.
No storage-loss recovery or new backup coverage is claimed.

2026-10-21 resource/retention lesson: active Prometheus status flags confirm
`storage.tsdb.retention.time=1w` and `storage.tsdb.retention.size=7GiB`.
Recent kubectl top snapshot totals approximately 13m CPU and 289 MiB memory
across monitoring containers, below the 2 GiB planning target in this snapshot.
Data-directory usage is about 31.4 MiB for Prometheus and 52.5 MiB for Grafana;
the VM filesystem mounted at /prometheus has about 61.7 GiB available.
The latter is shared filesystem capacity, not a 10 GiB PVC quota. These are
recent observations, not peak-load, growth-trend or seven-day retention tests.
No further Pod replacement was performed after the owner skipped that exercise.

2026-10-21 Grafana persistence exercise: deleted only Grafana Pod
`grafana-7498f5d556-bjj7b` after owner readiness. Its controller created
`grafana-7498f5d556-tcxch`, Ready with zero restarts. The Pod UID changed;
both monitoring PVC UIDs and volume bindings remained identical to baseline.
After browser reload, dashboard `adhnb5z` retained all four saved panels and
displayed real data. Other monitoring Pod identities were unchanged. All 60
fixed-time historical CPU samples matched the baseline fingerprint exactly;
all five scrape jobs reported up=1. Argo's three Applications were Synced/Healthy,
all eight media Deployments were available, and no monitoring Warning events
were present. Detailed local evidence is in ignored
`tmp/grafana-persistence-result.json`. This verifies Grafana Pod-replacement
persistence only; Prometheus replacement, storage-loss recovery, backup
coverage and full dashboard completion are not claimed.

2026-10-21 network follow-up: rendering, server dry-run, live diff and running
promtool validation passed. Standards and Spec reviews found no blocking issues.
Argo reached `8190f98` Synced/Healthy; exporter Pod is 2/2 Ready and Prometheus
is Ready, both with zero restarts. All five targets report up=1 with empty
lastError. Deployed network collector success=1; only ens18 network series are
exported. Receive counter 36,167,071,274 bytes exactly matched the surrounding
host reads; transmit 34,123,010,087 bytes lay between host reads 34,123,005,844
and 34,123,010,365 bytes. This validates the VM view rather than Pod eth0.
One-minute receive/transmit rates returned real series. The network sidecar
used about 1m CPU/3 MiB memory in the post-rollout snapshot; all monitoring
containers totaled about 315 MiB. No monitoring warning events were present,
all three Argo Applications were Synced/Healthy and all eight media Deployments
remained available. These are recent snapshots, not peak-load measurements.
This configuration-driven Prometheus replacement is not the full Lesson 5
persistence exercise; Grafana persistence and saved dashboards still need testing.

| Stage | Commits | Evidence |
| --- | --- | --- |
| Prometheus | `884e512` and filename follow-up | Internal Service, 10 GiB local-path PVC, seven-day/7-GiB retention, self-scrape verified |
| VM metrics | `055c25d`, `2da684f` | CPU/memory/filesystem agree with VM; recursive read-only host mounts verified; network deferred |
| Kubernetes object state | `e5eb568` | Six collectors and matching read-only RBAC; media replicas, readiness/restarts and PVC phases match kubectl |
| Kubelet access and consumption | `27863ce`, `fbe421b` | Direct verified HTTPS /metrics/resource; all eight media containers; only CPU and memory working-set families retained |
| Lesson 3 checkpoint | `5535041` | Owner distinguishes object state, consumption and stored history; Lessons 1–3 passed |
| Grafana storage/credentials | `624ddec`, `52b9a24` | Protected 1 GiB PVC; strict-scope SealedSecret Synced=True; backup coverage checked first |
| Grafana workload/data source | `d754aff`, `426aa18` | Pinned reviewed 13.2.2 image; server-side internal Prometheus source; plugin installation/updates explicitly disabled |
| Grafana ingress | `251024e` | DNS resolves to VM; login required; real session, database/data-source health and four up=1 results verified through Traefik |

- Manifest rendering, server dry-runs, reviewed live diffs and Standards/Spec
  reviews passed for deployed stages. Runtime promtool config validation passes.
- Grafana's first startup downloaded optional plugins and attempted bundled-plugin
  updates; `426aa18` disabled preinstallation and automatic updates. Existing
  optional plugins remain on its PVC; corrected startup has no installer errors.
- Sealing certificate SHA-256:
  `b77a5acd6d48fc4bf2d7620d1da9a443cd4c205789efbf0de1726c3e14e6b1a4`.
  It matches the verified 2026-10-08 NAS inventory; encrypted backup checksum
  matches its recovery README. Password generation/sealing used process memory,
  with no plaintext saved or printed. No new key backup was necessary.
- Final cleanup check on 2026-10-19: node Ready, all three Argo Applications
  Synced/Healthy, all four monitoring Pods Ready with zero restarts, both claims
  Bound, all eight media Deployments available. Monitoring render/server dry-run
  and running promtool check passed. No workload changes were made during cleanup.
- Subsequent final inspection: all namespace controllers/Pods healthy (completed
  bootstrap Jobs excluded from readiness), all 11 PVCs/PVs Bound, no warning
  events or observed scrape failures in the last 15 minutes. Existing restarts
  are historical; running Pods' last terminations are from the October 15 boot.
  API readyz passes; all SealedSecrets Synced=True; Argo has no conditions or
  pending prune. VM 5-minute CPU busy about 1.9%, available RAM 12.1 GiB, root
  disk available 61.7 GiB, load5 0.09. Monitoring uses about 236 MiB in a recent
  kubectl top snapshot. No exporter collector/device errors, TSDB compaction
  failures or WAL corruptions. These are current observations, not peak-load,
  seven-day retention or backup tests. Agent inspection recipes are linked
  from AGENTS.md; no permanent forward or workload change is required.
  All checked app ingress pages respond HTTP 200; Transmission returns the
  expected unauthenticated 401. Prometheus, Grafana and kube-state-metrics
  report no error-level log entries in the last 15 minutes. These HTTP checks
  do not exercise media playback or authenticated Transmission RPC.
- Persistence through controlled Pod replacement, seven-day observed retention,
  Grafana application-data backup coverage and media playback were not tested
  in this lesson. No Lesson 4 checkpoint or whole-stack completion is claimed.

## Confirmed decisions

- Goal: learn Prometheus and Grafana through a guided, step-by-step deployment
  that explains concepts as each component is introduced.
- Initial scope: K3s VM and Kubernetes/media workload health, including resource
  usage, container restarts, and expected versus available Deployment replicas.
- Use explicit Kubernetes manifests with Kustomize and the existing Argo workflow.
- Use namespace `monitoring`. During implementation, add its manifest to
  `cluster/namespace/monitoring-namespace.yaml` and register it in
  `cluster/namespace/kustomization.yaml`, consistent with existing namespaces.
- Workload configuration belongs in `infrastructure/node-main/ns-monitoring/base`
  and `infrastructure/node-main/ns-monitoring/overlays/prod`.
- Start with dashboards only; notification delivery is a separate future improvement.
- Retain seven days of Prometheus history on a persistent local-path volume,
  with a size retention limit chosen after checking available disk space.
  History should survive Pod replacement but is disposable after storage loss;
  there is no requirement to restore historical samples. Do not use NFS for TSDB.
- Expose Grafana at `http://grafana.home.arpa` through existing Traefik ingress,
  with login enabled and the admin credential managed through Sealed Secrets.
  HTTPS is a possible later improvement.
- Keep Prometheus internal (ClusterIP); use `kubectl port-forward` for lessons
  and troubleshooting. Grafana queries it through its internal Service.
- Introduce these sources one at a time: Prometheus self-metrics, Node Exporter
  for the K3s VM, kube-state-metrics for Kubernetes object state, and the kubelet's
  existing metrics for container CPU and memory. Explain host mounts, discovery,
  authentication and least-privilege RBAC as they are introduced.
- Build small dashboards manually with the owner, then export JSON to Git and
  provision the curated dashboards and Prometheus data source. Grafana gets a
  small persistent local-path volume so experimental dashboards survive Pod
  replacement. Explain that UI edits do not automatically update Git and that
  provisioning can overwrite edits; use a separate editable learning dashboard.
- Teach the simplest collection loop first: Prometheus scrapes itself, the owner
  inspects a metric and queries it, then adds coverage and Grafana progressively.

## Independently startable follow-up phases

Each phase depends on the working initial metrics stack, not on another follow-up.
Establish device capabilities, credentials, network access, desired metrics and
exporter choice before implementing each one; do not infer missing host facts.

- Proxmox monitoring: physical host and VM metrics.
- NAS monitoring: capacity, health and supported performance metrics.
- UPS monitoring: power state, charge and runtime through an owner-confirmed interface.

## Future improvement: notifications

Design alert rules and notification delivery in a separate phase. Decide routing,
credentials, thresholds, pending durations and how to test delivery. Explicitly
address that monitoring running inside the K3s VM cannot deliver notifications
when that VM or its underlying host is down without an external observer.
No Alertmanager or notification integration is required for the initial deployment.

## Agreed budget and completed capacity check

The owner approved a 10 GiB Prometheus local-path PVC, approximately 7 GiB
size retention, seven-day time retention, a 1 GiB Grafana local-path PVC,
30-second scrapes, and a target of less than 2 GiB total monitoring memory
under normal operation. The size and time policies apply together: the earlier
one wins, so seven days is a maximum rather than a guaranteed minimum.
The size retention policy covers retained blocks, not a hard cap on total
database usage; allow room for WAL, head data and compaction.

Read-only capacity preflight completed on 2026-10-19. The 8-vCPU/15.61-GiB
VM had approximately 11.64 GiB available memory and 64.04 GiB available disk;
all pressure conditions were False. The agreed budget fit that idle snapshot.
Local-path uses `/var/lib/rancher/k3s/storage`, WaitForFirstConsumer and Delete
reclaim policy, without expansion. Requested capacity is not a filesystem quota.
These observations are historical, not peak measurements or current guarantees.

## Instructions for the implementation and teaching agent

Use the `implement` and `teach` skills as appropriate to the guided session.
Read their instructions at session start. Work one lesson at a time: explain
the concept, show the small manifest/configuration change, deploy through the
existing Git/Argo workflow, inspect evidence with the owner, and pause for
questions and readiness before the next lesson. Do not deploy the whole stack
in one action. This pedagogical pacing is requested by the owner.

Teach exclusively through the conversation. Do not create lesson files,
teaching stylesheets, reference materials or teaching workspace scaffolding.
Keep implementation progress in this plan and operational documentation under
`ns-monitoring`; these are not substitutes for conversational teaching.

Use ordinary manifests and Kustomize; no Helm, Prometheus Operator, monitoring
CRDs, remote-write backend, logs or tracing are needed for this scope.
Choose supported releases compatible with the running K3s version at execution
time, review their documentation/releases, and pin exact image digests.
Do not copy version-sensitive retention syntax from this plan without checking
the chosen Prometheus release; flags and configuration syntax can change.

### Lesson 1: namespace, GitOps and the first collection loop

- Explain namespace scope and why collecting across namespaces still needs RBAC.
- Add `cluster/namespace/monitoring-namespace.yaml` and register it in that
  directory's kustomization. Apply using the manually managed cluster workflow
  before allowing Argo to create monitoring workloads. Do not move namespace
  ownership into the monitoring Application or add automatic namespace creation.
- Create `ns-monitoring/base` and `ns-monitoring/overlays/prod`. The existing
  ApplicationSet discovers this overlay and maps it to namespace `monitoring`;
  no new ApplicationSet or manual duplicate Application is required.
- Deploy one Prometheus replica with configuration, internal Service and 10 GiB
  PVC. Use a single-writer rollout strategy suitable for local RWO storage,
  readiness/startup checks, resource requests and reasonable limits. Choose
  per-component settings within the agreed overall planning target and measure
  them; under 2 GiB is a target, not a guarantee or a namespace hard limit.
- Protect both new PVCs with the existing Argo
  `Prune=confirm,Delete=confirm` pattern; retain existing reclaim policies.
- Start with one explicit self-scrape job. Use a rollout on configuration changes
  so new ConfigMap content is applied predictably without adding a reload sidecar.
- Explain pull collection, target, scrape, timestamp, time series and labels.
- Validate configuration with the chosen release's `promtool`, render Kustomize,
  inspect the Argo diff, then verify Synced/Healthy and Prometheus readiness.
- Port-forward to Prometheus. Inspect `/metrics`, the target page and `up`;
  explain that target health means scrape success, not application correctness.

Checkpoint: owner can explain where samples are stored and why Grafana is not
needed to collect or query them.

### Lesson 2: VM metrics and first useful PromQL

- Add Node Exporter as a DaemonSet, exposing an internal metrics endpoint and
  correctly reading the K3s VM's host filesystem/process information. Explain
  each required read-only host mount and namespace option; avoid gratuitous
  privilege, host writes or runtime socket access. It observes the Ubuntu VM,
  not the underlying Proxmox host.
- Add an explicit scrape job using stable Service discovery appropriate to
  this single-node setup. Do not enable annotation-based scraping of arbitrary
  workloads by default.
- Teach gauges, counters, `rate`, label filtering and aggregation. Inspect actual
  series before building queries; demonstrate CPU utilization, available versus
  total memory and real filesystem free space, excluding pseudo filesystems.
- Confirm the exporter reports the VM's capacity, rather than container-only
  resources. Verify the new target is up and queries return plausible values.

Checkpoint: owner can explain why a CPU counter needs a rate and why filesystem
free space is different from a PVC's requested capacity.

### Lesson 3: Kubernetes state and resource consumption

- Add kube-state-metrics for nodes, Pods, Deployments and relevant storage
  objects; restrict enabled collectors and list/watch permissions accordingly.
  Do not grant access to Secrets for this scope or export arbitrary annotations
  and labels. Explain ServiceAccounts, Roles/ClusterRoles and bindings.
- Add authenticated kubelet/container metrics scraping. Inspect the actual K3s
  endpoints and serving certificates; use verified TLS and scoped credentials.
  Prefer direct authenticated node metrics access with narrow RBAC when
  supported. Do not silently fall back to insecure TLS or broad `nodes/proxy`
  permissions; API proxy access can confer capabilities beyond metrics reads.
  If direct access is blocked, explain the concrete limitation before changing
  the access design.
- Use bounded node discovery and relabeling that preserves namespace, Pod and
  container identity. Verify exact metric names and endpoint availability with
  the running Kubernetes release. No separate cAdvisor Pod is required.
- Teach desired versus observed state, restart counters, CPU cores, memory
  working set, and how infrastructure health differs from a working media flow.
- Verify media Pod restart/readiness metrics and Deployment replicas against
  `kubectl`; verify container CPU and memory series exist for real media
  containers and filter infrastructure/empty container series correctly.
- Explain missing series, zero values and cardinality. Validate one stage before
  adding the next, without deliberately restarting or scaling media workloads.

Checkpoint: owner can choose between object-state and consumption metrics for
a question, and explain why metrics-server/`kubectl top` is not stored history.

### Lesson 4: Grafana and a dashboard built together

- Deploy one Grafana replica with a 1 GiB PVC, probes, resource settings,
  internal Service and Traefik ingress for `grafana.home.arpa`.
- Create the admin credential through the existing strict-scope Sealed Secrets
  workflow; check existing key-backup coverage. Never put plaintext in Git or
  command output. Disable anonymous access and self-signup. Explain bootstrap
  credentials versus later password changes stored in Grafana's database.
- Provision Prometheus as a data source using its internal Service address.
  Grafana queries Prometheus server-side, not through the owner's browser.
- Verify ingress routing, required login, data-source connectivity and a query.
- Teach Explore, instant/range queries, time ranges, resolution, units and legends.
- Build a small editable learning dashboard together before importing any large
  community dashboard. Include VM CPU, memory, disk free and network activity;
  media container CPU/memory; restarts; readiness and desired/available replicas.
  Include a simple scrape-health view so missing data is visible.
- Use namespace/workload filters and sensible units. Treat no data explicitly;
  do not turn absent samples into a misleading healthy zero.

Checkpoint: owner can take one query from Prometheus to Explore to a panel and
explain the displayed units and time range.

### Lesson 5: reproducibility, persistence and operating the stack

- Export curated dashboard JSON with stable UIDs, commit it and provision via
  ConfigMaps/files. Keep a separate editable learning copy. Teach the cycle:
  edit the copy, export, review, commit, sync. Data source and curated dashboard
  configuration are reproducible from Git; experiments remain in Grafana's DB.
- Verify dashboard queries survive provisioning and namespace filters work.
- Replace only the new monitoring Pods in a controlled sequence, preserving
  PVCs. Demonstrate historical samples and saved learning dashboards surviving
  Pod recreation. Never delete claims to test persistence.
- Measure actual monitoring memory, CPU and disk usage; inspect for OOM kills,
  throttling and scrape failures. Do not claim seven-day retention is empirically
  validated immediately; verify configured limits now and growth over time.
- Explain TSDB head/WAL/blocks, retention and local-path's shared disk/quota
  limitations. Describe PVC prune safeguards and direct deletion bypassing them.
- Document recovery: reinstall manifests and restore credentials/data source/
  curated dashboards from existing sources; historical metrics may be lost.
  Grafana UI-only changes require a database backup to recover after storage loss.
  Do not claim new PVC backup coverage without verifying it. Future stateful
  upgrades still require the repository's application-native backup procedure
  even though long-term metrics history is disposable.
- Rollback configuration through Git and wait for Argo. Do not blindly downgrade
  an image against an incompatible database schema or delete PVCs to roll back.
- Update `system.md` only if execution establishes or changes durable facts
  that cannot be discovered from code or normal read-only inspection, such as
  owner-confirmed recovery arrangements outside the cluster. No system-record
  change is required merely because monitoring was deployed. Keep versions,
  storage, access and RBAC configuration in manifests; record validation and
  measured resource usage in this plan or operations notes. Add concise
  operations notes under `ns-monitoring` and link them from existing docs.

Completion: Argo is Synced/Healthy; all intended scrape targets are up; Grafana
login and data source work; each promised dashboard panel has real data;
new Pod replacement preserves state; no monitoring secrets are plaintext in Git;
existing media workloads remain available. No whole-stack outage test is needed.

## Follow-up phase acceptance criteria

For each optional Proxmox, NAS and UPS phase, first read its relevant system
record and ask the owner for unknown interface, host and credential facts.
Choose a maintained exporter and least-privilege access; store only encrypted
credentials in Git. Deploy its Kubernetes configuration beneath node-main,
or explicitly document owner-approved installation on an external device.
Teach one raw metric, one query and one useful panel; verify observed data
against the device's own interface. Record confirmed facts in the appropriate
system record. Each phase can be undertaken independently after Lesson 5.

Notifications remain a separate improvement independent of those device phases.
Application-specific metrics and synthetic media checks can also be considered
later; collecting resource metrics does not prove playback/search/download health.

## Primary references

- [Prometheus getting started](https://prometheus.io/docs/prometheus/latest/getting_started/)
- [Prometheus storage and retention](https://prometheus.io/docs/prometheus/latest/storage/)
- [Node Exporter guide](https://prometheus.io/docs/guides/node-exporter/)
- [kube-state-metrics and compatibility](https://github.com/kubernetes/kube-state-metrics)
- [Kubernetes node metrics](https://kubernetes.io/docs/reference/instrumentation/node-metrics/)
- [Grafana provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)

Check version-sensitive details against the selected releases during implementation.

## Repository constraints

Read the current `system.md` records before implementation and begin cluster work
with `kubectl get nodes -o wide`. Preserve unrelated working-tree changes.
Keep credentials outside Git; follow existing Sealed Secrets practices.
Pin reviewed container digests. Future stateful image updates require release
review, application-native backup, one digest update, rollout and smoke test.
Do not add unattended updates or runtime socket access.
Update system records only for confirmed changes to facts within their scope,
as defined in `AGENTS.md`; avoid duplicating code or discoverable runtime state.
