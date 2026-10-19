# Monitoring deployment and teaching plan

Status: implementation started on 2026-10-19; Lesson 1 in progress.
Continue one lesson at a time, pausing at each owner checkpoint.

## Implementation and teaching record

- Lesson 1: manifests and teaching materials prepared; bootstrap namespace
  created through the manually managed `cluster/` workflow. Monitoring workloads
  are to be deployed exclusively by the existing ApplicationSet/Argo workflow.
- Selected Prometheus v3.15.0 after reviewing the official release notes.
  Resolved the pinned multi-platform image digest from the official registry.
  Downloaded matching release tools and verified the archive against its
  published SHA-256 checksum. `promtool check config`, Kustomize rendering,
  server dry-run and live diff passed; the diff adds only the four intended
  monitoring objects. Runtime validation is pending deployment.
- Owner asked whether Argo should deploy everything and undo manual changes.
  Explained the existing boundary: bootstrap resources are manually managed;
  Argo manages discovered workload overlays and self-heals their drift.
- [Lesson 1](../lessons/0001-prometheus-collection-loop.html) and
  [operations notes](../infrastructure/node-main/ns-monitoring/README.md) explain
  the collection loop and provide the first query exercise.
- Owner understanding/readiness checkpoint remains pending. Lessons 2–5 have
  not started; no whole-stack completion or persistence validation is claimed.

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
  measured resource usage in this plan or operations/lesson notes. Add concise
  operations/lesson notes under `ns-monitoring` and link them from existing docs.

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
