# Monitoring deployment and teaching plan

Status: Lessons 1–3 completed, deployed and verified. Owner passed the Lesson 3
checkpoint and requested Lesson 4. Lesson 4 is incomplete and stopped for the
evening at the owner's request. Do not implement Lesson 5 without readiness.

## Resume instructions

- Lesson 4 started after checkpoint commit `5535041`. Grafana PVC stage
  `624ddec`, sealed bootstrap credential `52b9a24`, workload/internal data source
  `d754aff`, automatic plugin install/update disablement `426aa18`, ingress
  `251024e`. Runtime login/data source/ingress are verified. Owner has logged
  in and run Explore `up` in Instant and Range modes, setting legend `{{job}}`.
  Screenshots show four instant series at 1 and overlapping range traces at 1.
  Resume by creating the first editable panel (Scrape health) in a new
  Monitoring — Learning dashboard; last instruction was to inspect Explore's
  Add dropdown, whose contents have not yet been confirmed. Do not repeat login
  or Explore setup unless needed. No dashboard created yet; do not
  auto-create the whole dashboard or begin Lesson 5. Checkpoint remains pending.
- Sealing coverage verified against mounted NAS inventory/recovery README:
  active certificate SHA-256
  `b77a5acd6d48fc4bf2d7620d1da9a443cd4c205789efbf0de1726c3e14e6b1a4`;
  encrypted backup checksum matches the verified 2026-10-08 record. Used that
  exact fetched public certificate; random password generated/sealed in memory.
  No plaintext credential saved or printed.
- Existing localhost:9090 port-forward responds Ready. Exporter network
  investigation confirms /host/proc/1/net/dev includes VM ens18, whereas
  /host/proc/net/dev shows Pod eth0. v1.12.1 netdev defaults to namespace-local
  netlink. Network collector is still disabled; settle/validate design before
  any VM traffic panel. Do not substitute container traffic.
- Read AGENTS.md, relevant infrastructure system records and this plan. Start
  cluster work with kubectl get nodes -o wide, then check Argo/Pod health.
- Lesson 3 is deployed: kube-state-metrics (`e5eb568`), node-scoped Prometheus
  access (`27863ce`) and authenticated kubelet scraping (`fbe421b`). Do not
  redeploy or repeat its checkpoint. Owner correctly identified kubelet metrics
  for growing memory usage after clarification, and independently explained
  Prometheus history versus recent kubectl top measurements. Lesson 3 checkpoint
  passed; owner subsequently requested Lesson 4 (in progress above).
- Owner passed object-state recall: desired=1/available=0 means the requested
  replica cannot become available; would inspect logs. Explained checking Pod
  status/events first, since scheduling/image-pull failures may have no logs.
- Keep teaching exclusively conversational and minimal: explain one small
  change before Git/Argo deployment; one Kubernetes resource per file.
  Owner requested clarification of metrics-server versus kube-state-metrics
  and HTTPS/authentication before continuing. Those explanations were given.
- All created files belong inside this repo. Use ignored tmp/ for artifacts,
  downloaded tools and TMPDIR. Earlier /private/tmp artifacts were not moved
  or deleted; no new files were written there after the owner's instruction.
- Direct kubelet /metrics/resource uses namespace CA verification, rotating
  Pod-bound token, single-node discovery and name-scoped RBAC. Do not replace
  this with insecure TLS or nodes/proxy. Only CPU and memory working set are
  retained, with namespace/Pod/container and node identity. All eight media
  containers verified. See operational README for queries and access details.
- Port-forward was re-established on localhost:9090 after rollout; check it on
  resume. CPU 1-minute rates returned all eight containers. A full 5-minute
  kubelet rate window has not yet been validated; prefer it after history builds.
- Lessons 1–2 checkpoints passed. Owner knows the PVC preserves samples,
  CPU counters need rates, and shared filesystem free space differs from PVC
  requested capacity. Grafana's display/query role has been recapped.
- Full Pod-replacement persistence validation remains Lesson 5. Node Exporter
  has recursive read-only proc/root mounts; replace its Pod after host mount
  changes. VM network collection remains a Lesson 4 design/verification task;
  do not present container network series as VM traffic.

## Implementation and teaching record

- Owner stopped Grafana configuration for the evening after successfully
  running Explore `up` in Instant and Range modes with legend `{{job}}`.
  Conversation covered series labels, unitless scrape success, time range and
  step/resolution, overlapping traces, and missing data versus failed scrapes.
  This records the exercise, not a passed Lesson 4 checkpoint. First saved
  dashboard panel remains pending. Owner asked for remaining original scope
  beyond Grafana: VM network collection, Lesson 5 persistence/resource/retention
  checks and recovery operations; optional device phases and notifications
  remain separate. No further deployment or Lesson 5 work performed at wrap-up.

- Lesson 4 initial stages: node Ready, all three Argo Applications initially
  Synced/Healthy, all workload Pods Ready. Clean starting tree at `5535041`.
  PVC render/server dry-run passed; live diff adds only PVC plus existing Argo
  tracking/defaulted metadata differences. Both Standards/Spec reviews passed.
  After Git/Argo deployment claim was Pending with WaitForFirstConsumer as
  expected; Grafana scheduling subsequently bound it at 1 GiB.
- Strict-scope grafana-admin SealedSecret validated with controller; deployed
  condition Synced=True and generated Opaque Secret verified without values.
  Reviewed official Grafana 13.2.2 release changelog including security fixes,
  resolved multi-platform image digest from Docker Hub. Workload/data-source
  render, server dry-run, live diff and both review axes passed. No existing
  workload template, kubelet TLS/RBAC, image or PVC changed. Runtime Grafana
  validation subsequently passed as recorded below; no checkpoint claimed.
- Grafana startup revealed default optional-plugin downloads and attempted
  bundled-plugin updates against read-only image storage. Corrected via Git/Argo
  `426aa18` using tagged defaults.ini settings: preinstall_disabled=true and
  preinstall_auto_update=false. Corrected rollout has no background installer
  activity or update errors. Initial optional plugins remain on the data PVC;
  automatic installation/updates are disabled. No stateful image change.
- Lesson 4 runtime validation around 22:13–22:16 CEST: all three Applications
  Synced/Healthy at ingress stage `251024e`; all four monitoring Pods Ready,
  zero restarts. Grafana PVC Bound, 1 GiB. grafana.home.arpa resolves to
  192.168.30.103; Traefik redirects anonymous browser to /login, anonymous
  /api/user returns 401. Real session login, database health, provisioned
  Prometheus health and query `up` all pass through ingress and localhost;
  four scrape targets return 1. Credentials used only in process memory.
  All eight media Deployments remain 1/1 available; no media changes.
- Both review axes passed for Grafana workload/credential/ingress stages.
  Server dry-runs and reviewed live diffs passed; expected diff exit 1 for
  additions/Argo metadata. Port-forwards: existing localhost:9090 verified;
  localhost:3000 reconnected after the configuration rollout. Owner login and
  dashboard interaction are next; network collector remains disabled pending
  a validated collection design. Lesson 4 checkpoint and all Lesson 5 work
  remain pending. Preserve narrow kubelet RBAC and verified TLS.

- Lesson 3 checkpoint passed: owner correctly explained that Prometheus stores
  historical time series while kubectl top only supplies recent measurements.
  Initially associated kube-state-metrics with memory usage; clarified object
  state versus consumption. On retry, owner correctly chose kubelet metrics
  for investigating growing Radarr memory usage. Earlier desired/available
  replica recall also passed. Lesson 4 later started as recorded above;
  Lesson 5 remains unstarted.

- Lesson 3 consumption stage deployed through Git/Argo `fbe421b`. Matching
  promtool syntax check, Kustomize render, server dry-run, live diff and separate
  Standards/Spec reviews passed; no findings. Full promtool check in the running
  Pod also passed with real mounted credential/CA paths. No image or PVC change.
- Validation around 21:54–21:56 CEST: all three Argo Applications Synced/Healthy;
  monitoring at `fbe421b`, all 14 desired resources Synced, none pending prune.
  Prometheus rolled out Ready, zero restarts; all four targets up with empty
  errors. Exactly one kubelet target discovered at
  https://192.168.30.103:10250/metrics/resource. All eight real media containers
  have both required series and match current kubectl Pod/container identities.
  Exactly two container metric families retained (50 series across 25 cluster
  containers at inspection), no empty identity or POD infrastructure series.
  Full namespace/Pod/container identity and node label verified.
- Memory MiB snapshot agrees with kubectl top within expected sampling changes:
  Radarr 179.33 vs 179; Seerr 335.36 vs 335; Transmission 6.09 vs 6.
  Jellyfin changed from 687 to about 700 between observations. One-minute CPU
  rates returned all eight containers; Radarr about 0.00124 cores (1.24m).
  These are snapshots, not steady-state/peak values. Full-window 5-minute CPU
  validation remains for later accumulated history. No media restart/scale.
- Conversational teaching covered existing kubelet versus new exporter,
  certificate verification/authentication/authorization, scoped discovery,
  rate in cores (1 core = 1000m), memory working set bytes/MiB and cache caveat,
  differing sampling windows versus kubectl top, and stored Prometheus history.
  Object-state availability is distinct from consumption and media-flow health.
  Lesson 3 checkpoint subsequently passed as recorded above; stop here before
  Grafana or Lessons 4–5.

- Lesson 3 kubelet access stage: rechecked node/Argo/Pod health and serving
  certificate. IP SAN 192.168.30.103, issuer k3s-server-ca, expiry 2027-10-15;
  verification against namespace CA passed, anonymous cadvisor request 401.
  Prepared dedicated Prometheus ServiceAccount and node-name-scoped list/watch
  nodes plus get nodes/metrics permissions. No nodes/proxy or write permissions.
  Server dry-run and live diff passed; no workload-template change at this stage.
  Deployed access stage in `27863ce`; Argo Synced/Healthy. Short-lived token
  held only in memory: selector-bounded node list 200, unbounded list 403,
  API node-proxy request 403, direct cadvisor/resource metrics 200 with verified
  TLS, direct /pods 403. Credential not saved. Both endpoints contain the media
  CPU/memory series. Chose /metrics/resource for its stable names and simpler
  namespace/Pod/container labels. Prepared one new job with bounded node
  discovery, CA/token paths, node label and filtering to two real-container
  measurements. Prometheus Pod opts into its rotating ServiceAccount credential.
  Local matching promtool syntax validation, render and server dry-run passed;
  live diff changes only Pod identity/config plus generated ConfigMap (and
  expected tracking/defaulted metadata). Runtime full promtool check follows
  where mounted credential/CA paths exist.

- Lesson 3 object-state stage deployed via Git/Argo commit `e5eb568`.
  Selected kube-state-metrics v2.20.0 after reviewing its official release and
  tagged CLI/deployment documentation; Kubernetes client-go v0.36.3 matches the
  running 1.36 API family. Resolved official registry image index SHA-256.
  Five separate resource files, six selected collectors and matching list/watch
  RBAC; no Secrets permission, annotation/label allowlists or extra telemetry
  scrape. Matching promtool, render, server dry-run, pre-push live diff and
  separate Standards/Spec reviews passed. No stateful image update.
- Runtime validation around 21:42–21:47 CEST: Argo revision `e5eb568`
  Synced/Healthy; all 11 desired resources Synced, none pending prune.
  Both changed Deployments rolled out; all three monitoring Pods Ready with
  zero restarts. All three scrape targets up with empty scrape errors.
  Exporter logs confirm exactly the six intended stores and empty label and
  annotation allowlists. Its telemetry reports successful watches for all six
  resource types; no failed watch series. Secret listing and Pod creation denied
  for its ServiceAccount. 41 comparisons matched kubectl: desired/available
  replicas for all eight media Deployments, readiness/restarts for all eight
  containers and Bound phase for all nine media PVCs. Jellyfin and Seerr each
  retain their pre-existing single restart; other media containers report zero.
  All media Deployments remain 1/1 available. No media restarts/scales performed.
  Brief CPU/memory snapshot: exporter 1m/12Mi, Node Exporter 1m/7Mi,
  Prometheus 2m/38Mi; not steady-state or peak measurements.
- Conversation distinguished metrics-server from kube-state-metrics and
  Prometheus history; explained identity/RBAC/binding, desired versus available,
  labels and cardinality, readiness/restarts and missing versus zero samples.
  Owner subsequently passed the object-state recall check; kubelet stage and
  final checkpoint status are recorded above. No whole-stack completion claimed.
- Owner requires all created files to stay inside this repository. Added rule
  to AGENTS.md and ignored root tmp/; use it for artifacts and TMPDIR henceforth.
  Earlier session artifacts under /private/tmp were not moved or deleted.

- Lesson 2: deployed through Git/Argo in commits `055c25d` and `2da684f`.
  Reviewed official Node Exporter v1.12.1 release notes (perf/edac fixes) and
  tagged collector documentation/source, and resolved the multi-platform digest
  directly from Quay's registry. Added a node-bounded DaemonSet and internal
  Service plus one explicit 30-second scrape job. No stateful image update.
  Prometheus retained its image and PVC; config hash triggered its rollout.
- Validation around 21:22–21:24 CEST: matching v3.15.0 promtool config check,
  Kustomize render, server dry-run and pre-push Kubernetes diff passed. Review
  found ordinary read-only root mounts could expose writable submounts; fixed
  with required recursive read-only mounts before push. Standards re-review
  and Spec review have no remaining findings. Argo revision `2da684f` reports
  Synced/Healthy with all six desired objects Synced and no pending prune.
  Both monitoring Pods Ready with zero restarts; runtime mount status confirms
  recursiveReadOnly Enabled on both host mounts. Both targets up with empty
  scrape errors; all five exporter collectors report success and all three
  real filesystem metrics have device_error=0. Raw metric types confirm CPU
  counter and memory/filesystem gauges.
- VM capacity verified: eight idle CPU series match eight vCPUs; MemTotal
  16,761,421,824 bytes matches Kubernetes capacity 16,368,576 KiB; root ext4
  capacity 103,474,552,832 bytes exactly matches the independent kubelet summary.
  Snapshot: available memory 12.39 GiB, memory unavailable fraction 20.64%,
  root available 63.64 GiB versus free 68.57 GiB; 1-minute non-idle CPU 1.83%.
  Initial CPU/working-set sample: exporter 1m/6Mi, Prometheus 3m/33Mi; these are
  brief snapshots, not steady-state or peak measurements. All eight media
  Deployments stayed 1/1 available; all three Argo Applications Synced/Healthy.
- Lesson 2 conversational teaching covered DaemonSets, internal Service scraping,
  read-only host views, Grafana recap, gauges/counters, rate, label filtering,
  per-node CPU averaging, memory availability, root filesystem units and
  free versus available versus requested PVC capacity. The owner initially
  associated rate with averaging and free space with the PV; clarified rate
  as per-second counter increase, avg as cross-CPU aggregation, and filesystem
  measurements as distinct from both PVC and PV capacity. The owner then
  correctly answered both follow-up checks: shared disk availability decreases
  while the PVC request stays unchanged, and 15 idle seconds in 30 seconds
  means 50% busy. Lesson 2 checkpoint passed; wait for readiness before Lesson 3.
  Lesson 3 subsequently began as recorded above. No lesson files were created.
- Lesson 1: manifests committed and pushed; bootstrap namespace
  created through the manually managed `cluster/` workflow. Monitoring workloads
  were deployed exclusively by the existing ApplicationSet/Argo workflow.
- Selected Prometheus v3.15.0 after reviewing the official release notes.
  Resolved the pinned multi-platform image digest from the official registry.
  Downloaded matching release tools and verified the archive against its
  published SHA-256 checksum. `promtool check config`, Kustomize rendering,
  server dry-run and live diff passed; the diff adds only the four intended
  monitoring objects.
- Runtime validation around 20:55 CEST: Argo automatically deployed revision
  `884e512`; all four resources report Synced and Application health is Healthy.
  Inspected Argo's resource comparison: no OutOfSync resources or pending prune.
  Automatic sync began on discovery; this Argo comparison was after sync, while
  the pre-push diff was the Kubernetes server diff. The post-sync Kubernetes
  diff showed only Argo-injected tracking annotations and a dry-run generation
  change; no workload configuration difference. Rollout completed with one
  Ready Pod and zero restarts. Readiness and the Targets page returned HTTP 200;
  target API reports exactly one healthy self-target, no scrape error and a
  30-second interval. Raw `prometheus_build_info` confirms v3.15.0, and
  `up{job="prometheus"}` returns 1. Startup logs confirm 1-week/7-GiB retention.
  Initial `kubectl top pods` snapshot was 22m CPU and 34 MiB memory, less than
  a minute after creation; it is not a steady-state or peak measurement.
  Seven-day retention and Pod-replacement persistence have not been empirically
  tested. All eight media Deployments remained 1/1 available; existing Argo
  Applications remained Synced/Healthy.
- Review: no Lesson 1 manifest defects found. Corrected the existing recovery
  procedure to include the third Application and disposable metrics history.
- Owner asked whether Argo should deploy everything and undo manual changes.
  Explained the existing boundary: bootstrap resources are manually managed;
  Argo manages discovered workload overlays and self-heals their drift.
- Owner requested conversation-only teaching. Removed the generated lesson,
  shared lesson stylesheet and teaching workspace scaffolding. Keep
  [operations notes](../infrastructure/node-main/ns-monitoring/README.md)
  for running the stack; do not create lesson or teaching reference files.
- Owner correctly answered the storage checkpoint with "the PVC" and received
  explanations of GitOps ownership and ConfigMap generation. The owner then
  requested a filename consistency fix, which was deployed and verified, and
  requested this continuation record. At that checkpoint Lessons 2–5 had not started; no
  whole-stack completion or full persistence validation is claimed.

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

Read-only preflight completed on 2026-10-19 around 19:07–19:09 CEST:

| Check | Observed result |
| --- | --- |
| Node | `vm-ubuntu-k3s`, Ready, K3s `v1.36.2+k3s1` |
| Capacity | 8 vCPU; 16,368,576 KiB memory (15.61 GiB) |
| Actual usage (`kubectl top nodes`) | 85m CPU (1%); 4,130 MiB memory (25%) |
| Available memory (kubelet summary) | 12,494,651,392 bytes (11.64 GiB) |
| Node filesystem capacity | 103,474,552,832 bytes (96.37 GiB) |
| Node filesystem available | 68,757,733,376 bytes (64.04 GiB) |
| Existing local PVC filesystem | Same capacity and approximately same available space as node filesystem |
| Local-path base directory | `/var/lib/rancher/k3s/storage` |
| StorageClass | `local-path`, `WaitForFirstConsumer`, reclaim `Delete`, expansion disabled |
| Pressure | MemoryPressure, DiskPressure and PIDPressure all False |
| Scheduled resource requests | 200m CPU and 140 MiB RAM across the existing cluster |

Outcome: the agreed budget fits current observed capacity. An additional
2 GiB working set would bring memory use to roughly 38%; writing the full
11 GiB planned data budget would leave roughly 53 GiB available on this
filesystem before other growth. These are estimates from an idle snapshot,
not peak-load measurements or reservations. Existing workloads mostly have
no requests/limits, so scheduler reservations alone do not establish headroom.
Local-path requested capacity is not enforced as an individual filesystem quota.
No test Pods, PVCs, workloads or cluster configuration were created or changed.
The preflight is completed here, not a future implementation phase.

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
