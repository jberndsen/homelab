# Immich implementation handoff

Status: final, owner-approved sequential handoff; no implementation step has started.
Planning date: 2026-11-06. No workloads or NAS settings were changed during
planning. A subsequent owner request to execute the next step authorizes that
step within the agreed scope. Do not execute implementation during plan review.
This document is sufficient for an agent without the conversation.

## Clean-session execution contract

Owner prompt: **"Do the next step of the Immich plan until completion."**

1. Read this plan, [the progress ledger](immich-progress.md), AGENTS.md and the
   relevant system records. Inspect Git status/branch and recorded checkpoints.
   The ledger is the authoritative execution status; this plan defines scope
   and acceptance. Do not infer completion from files merely existing.
2. Select the lowest-numbered step not marked complete. If it is in progress or
   waiting, resume it; do not skip it. Verify preceding steps' recorded evidence
   and relevant live prerequisites without repeating completed work unnecessarily.
3. Execute **only that step**, including its checks, documentation and evidence.
   "Until completion" means completion of the selected step, not this whole
   plan. Do not begin the next step even if time/context remains or it looks easy.
4. When owner action, unavailable evidence or an external dependency prevents
   completion, record exactly what is needed, current live state and a safe
   resume action. Mark waiting or in progress, not complete; report and stop.
   Do not spend a session polling for days of history or a phone action. Never
   infer owner approval or successful phone/playback checks from silence.
5. Update the ledger with completed work, verification, remaining caveats,
   branch/commit checkpoint and any pending owner action. Keep concise durable
   evidence in plans/immich-step-N.md; bulky/raw/private output belongs in tmp/.
   A fresh agent must not need tmp/ files or this conversation to understand
   the checkpoint. Keep secrets and personal photo details out of evidence.
6. Commit task-related completed work in a coherent checkpoint following the
   repository's current branch/PR rules. Record branch and commit; preserve
   unrelated work. Push only to the intended branch, respecting the step's
   deployment boundary. Record the implementation commit in a subsequent ledger
   update rather than trying to embed a commit's own hash inside itself. If a
   checkpoint cannot be committed, document exact outstanding files and why.
7. End with: selected step and status, changes, verification, live impact,
   limitations/owner action, checkpoint, and next step's name. **Halt here.**
   The next invocation of the owner prompt is the instruction to advance after
   a completed step, or resume an incomplete one. No additional confirmation
   loop is required for routine work already authorized within a step.

Update documentation with each step; final reconciliation is an audit, not a
reason to leave the repository inconsistent between sessions. If a newly found
issue changes scope or makes the design infeasible, explain it and get an owner
decision rather than silently rewriting accepted policy. If a step outgrows a
session, checkpoint its remaining subtasks in the ledger and resume that same
step in the next session. Do not mark it complete merely to reach a clean stop.

## Confirmed scope

- Add Immich for iPhone photo backup as an additional copy. Retain existing
  originals during this initial deployment and validation. The long-term goal
  is for Immich to serve as the primary backup after sustained reliable use,
  avoiding an increased iCloud subscription. This session does not authorize
  deleting originals or reducing iCloud coverage. Before deleting existing
  copies to rely primarily on Immich, require an independent, versioned offsite
  backup of required assets and database backups, plus one successful recovery
  test demonstrating database/library recovery together. The owner explicitly
  accepted these later-transition prerequisites during critical review; both
  remain outside initial deployment scope.
- Import the entire existing library, including metadata, rather than only
  new photos. Establish which metadata and Apple Photos organization must
  survive, choose an import method accordingly, and test a representative
  sample before the full import. Do not assume mobile upload preserves every
  Apple-specific library feature.
- Metadata priorities: preserve capture dates and locations, plus other
  embedded metadata where feasible. Apple Photos albums, favorites, captions
  and other organization are secondary: preserve where trivially feasible,
  without making complex migration tooling a requirement.
- For edited photos, preserve the edited version the owner sees and wants to
  keep. Do not assume edits are embedded in the source original. Verify actual
  import/export behavior with edited samples, including dates and locations;
  clarify any tradeoff between edited renditions and source originals before
  bulk import. Live Photos should be included in representative validation.
- Critical-review decision: use standard Immich mobile backup as the default
  initial-import and ongoing-upload method. Check edited-photo behavior in the
  sample. If it does not preserve the intended edited appearance, explain the
  limitation and agree a simple supported alternative with the owner before
  full import; do not introduce custom migration tooling or silently claim
  edits are preserved. No extra original-plus-edited duplication is required
  unless the sample results lead to an explicit owner decision.
- Deletion must be independent: deleting a photo from the iPhone/iCloud must
  leave its uploaded Immich copy intact. Deletion in Immich requires a separate
  deliberate action. Verify this with disposable test assets, not real library
  photos, and document client settings/behavior that could affect this policy.
- The owner explicitly declined an isolated restore exercise for initial
  acceptance as too extensive. Do not make one a deployment-completion gate.
  Instead verify actual uploaded photo/video files directly on the NAS, sample
  original downloads and key metadata, and confirm database dumps are created
  at the intended location with retention configured. Document recovery steps
  but label recovery as untested; file visibility and dump creation do not prove
  restorability. Revisit recovery validation only for the later decision to
  reduce reliance on existing originals/iCloud, not during initial acceptance.
- Use separate accounts and photo libraries for each person. Validate with
  the owner's iPhone first, then support onboarding his wife after confirmation
  that the initial setup works. Include onboarding instructions for both stages.
- Use Immich's built-in email/password accounts with manual onboarding; no SSO
  or outgoing email setup. Keep credentials in the owner's password manager,
  outside Git and chat. Document account creation and supported administrative
  password recovery without relying on email delivery.
- Everything supported through code should be managed through code. Use
  Git-managed system configuration rather than UI configuration, including
  backups, storage template, timezone and enabled features. The owner accepts
  that IMMICH_CONFIG_FILE disables system-setting edits in the web UI. Use
  supported automation for account/storage-label configuration where practical,
  with private inputs and idempotent behavior; document unavoidable UI/iOS/NAS
  steps. Never commit plaintext credentials, tokens, account exports or photos.
- Assume automatic iPhone app updates. Server updates remain manually reviewed
  through Renovate, with release review, fresh native backup and smoke checks.
  Document the chosen release's mobile/server compatibility policy and how to
  catch up the server after an automatic mobile update; do not enable unattended
  upgrades. Current upgrading guidance says mobile typically supports the
  current and preceding major server version and recommends clients first.
  The FAQ's matching-major/minor troubleshooting advice is more restrictive;
  do not present exact minor matching as a universal compatibility requirement.
- Enable automatic mobile backup, including background backup. The owner
  accepts occasionally opening Immich at home to check upload progress and
  allow it to catch up. Document iOS photo/local-network permissions,
  Background App Refresh and OS scheduling limitations; do not promise
  immediate unattended uploads. Include foreground upload on power/home Wi-Fi
  for the initial full import where needed, with clear progress/error checks.
- Provide home-network access only; do not expose Immich publicly. Access
  through the existing VPN is welcome if it works without further changes.
  Verify this rather than assuming tunnel routing and DNS support it; VPN
  changes are outside the agreed scope.
- Use plain HTTP through the existing internal Traefik ingress, as explicitly
  chosen by the owner for home-network use. HTTPS/certificate setup is outside
  scope. Use the existing wildcard home.arpa DNS pattern; proposed application
  hostname is `immich.home.arpa`. Verify LAN resolution and iPhone connectivity.
- Critical-review decision: retain HTTP initially, and track trusted HTTPS
  across internal applications (including Immich) as separate repository TODO
  work. Design certificate trust, renewal and ingress changes consistently;
  do not introduce an Immich-only certificate system during this deployment.
- The purpose is an owner-controlled additional copy of original photos and
  videos, independently accessible if an iPhone is lost, rather than relying
  solely on Apple's cloud. Acceptance must verify retrieval of original files,
  not only viewing thumbnails. "Original" does not imply camera RAW format;
  preserve the source formats. Verify originals download and mobile import
  behavior during the sample-validation step.
- Use namespace `immich` and a dedicated Argo CD application.
- Use the official Helm chart as a reference for plain Kubernetes/Kustomize
  configuration; do not install with Helm.
- Integrate with the existing observability stack and Renovate workflow.
- Observability includes existing CPU, memory, readiness and restart views plus
  Immich-specific metrics where supported by the selected release, all through
  the existing Prometheus/Grafana stack. Do not add a separate monitoring
  system. Notification delivery remains out of scope, per the existing TODO.
  Verify scrape configuration, dashboard queries and actual series. Existing
  workload dashboards and kube-state-metrics collection focus on Deployments;
  account explicitly for any StatefulSets introduced by this deployment.
- Enable face recognition and smart search from the start, provided resource
  checks demonstrate sufficient headroom alongside existing workloads. Include
  initial machine-learning indexing in the capacity assessment; resolve any
  capacity shortfall with the owner before deployment.
- Resource priority was revised during critical review: media playback takes
  priority over Immich background processing when resources contend. Backups,
  indexing and face recognition can wait until overnight. Preserve usable
  Immich browsing/uploads where possible. This supersedes the earlier preference
  for no special tuning; derive requests, appropriate limits and any supported
  job controls from measured capacity and simultaneous-use validation. Explain
  resource behavior and tradeoffs to the owner during implementation.
- During critical review, the owner expanded scope to reasonable CPU/memory
  requests for both Immich and the existing media stack, with Grafana insights
  into resource allocation, consumption and headroom. This supersedes any
  interpretation that resource configuration is outside scope. Tune background
  processing when needed to meet the subsequently agreed media-first priority.
- Owner clarification: the objective is smooth simultaneous operation of media
  and Immich within known physical/VM capacity, avoiding harmful interference;
  the owner initially understood the resource discussion to include limits.
  No requests-only/no-CPU-limit policy has been accepted. Choose requests and
  limits against this objective, allowing Immich background work to slow before
  media playback. Metrics and requests alone cannot guarantee performance
  isolation. Do not equate Kubernetes PriorityClass with CPU scheduling priority
  or assume resource requests alone protect playback from storage/network load.
- Cover application backups and recovery, correct setup and operations guides,
  and accurate system records in the implementation and acceptance criteria.
- Produce a complete plan here for an agent with no conversation context.
- Implementation completion means a working deployment and a verified
  representative sample, with a full-import checklist handed to the owner.
  The owner completes the full-library import and confirms it is satisfactory
  before onboarding his wife. Do not claim the full library is backed up merely
  because the deployment/sample checks pass.
- Offsite NAS protection through the owner's requested UniFi service and
  Backblaze is separate work tracked in TODO item 11. Its availability and
  capabilities have not been verified in this session.
- Do not add NAS snapshots in this deployment. The owner chose to defer them
  because of additional storage retention concerns. Revisit snapshot/retention
  policy with the separate offsite-backup work; retain the agreed 14 daily
  database dumps. Do not silently enable snapshots as a backup prerequisite.

## Storage decisions and prerequisites

- Enable Immich's storage template before bulk import, with readable per-user
  storage labels and capture-date layout `person/year/month/filename.ext`
  (for example `jeroen/2026/11/IMG_1234.HEIC`). Preserve actual source extensions
  and formats, including videos; do not convert everything to HEIC. Confirm
  template syntax and user-label behavior against the chosen release. Let
  Immich manage collision handling and file moves; document that direct NAS
  browsing/copying is allowed but manual moves/renames break database paths.
  Validate capture-date/timezone handling with representative samples.
- The owner's iPhone had Optimize iPhone Storage enabled. During planning,
  the owner chose to switch to Download and Keep Originals to retain local
  full-resolution copies. Download completion and available device capacity
  are unverified; changing the setting alone is not evidence that the entire
  library is present locally. Verify before choosing the bulk import path.
- Owner-reported planning estimates: his iPhone uses 165 GB of 256 GB
  (approximately 91 GB free), and his iCloud Photos library occupies 57.5 GB.
  Local originals appear feasible with this headroom; verify completed download
  and actual free space before import. These are planning estimates, not durable
  capacity facts or measurements of his wife's library.
- The owner estimates approximately 200 GB of combined iCloud usage, mostly
  photos. Treat this as a sizing estimate, not a measured photo-library size.
- Use the new empty Shared Drive named `photos` on the UNAS Pro for Immich
  photo storage. NFS is not configured yet. This is a separate export from the
  existing media share; do not infer its path or permissions from that share.
- Include owner-guided UNAS NFS configuration, confirmed export path and
  permissions, client access restrictions, and read/write/remount tests from
  the actual Kubernetes workload context. Use the documented Shared Drive NFS
  configuration and verify the installed UniFi version's behavior.
- Use Collaborative Mode (All Squash), preserving UniFi/SMB browsing. Restrict
  NFS access to the K3s VM where supported; verify the actual configured rule.
  The owner confirms existing NFS shares follow this same convention. Test
  create/read/write/rename behavior under the application's effective identity;
  do not switch to Isolated Mode to bypass a permissions problem.
- Storage must grow easily. The owner is willing to buy additional NAS drives
  when physical capacity requires it. Document NAS capacity/quota checks and
  growth procedures, distinguishing actual storage limits from a static NFS
  PV/PVC's declared capacity. Account for generated assets and future growth.
- Keep live PostgreSQL data on local persistent storage in the K3s VM; keep
  photo/video storage on the NAS. Copy application-native database backups to
  the NAS. Do not place live PostgreSQL data on NFS. Use Immich's built-in daily
  database dumps, retaining 14 copies, written directly to NAS-backed storage
  rather than accumulating on the VM disk. Take fresh application-native
  backups before upgrades. Confirm the exact location, schedule/timezone and
  database/file consistency procedure before execution. Measure live database
  and dump sizes after import and include disk headroom checks.
  Recovery must restore the database together with its associated photo files;
  neither nightly VM backups nor database dumps alone establish full coverage.
- Deploy one PostgreSQL instance using ordinary Kubernetes manifests; do not
  introduce a database operator. Select a pinned image with the extensions
  required by the chosen Immich release, cross-checking its supported Compose
  deployment and official chart guidance. Manage through Git/Argo and document
  database-specific upgrade and recovery procedures.

## Repository integration and architecture

Read AGENTS.md, infrastructure/**/system.md, setup.MD, ns-argo/SETUP.md and
ns-monitoring/README.md before implementation. Paths below are relative to
infrastructure/node-main unless otherwise stated.

| Area | Implementation |
| --- | --- |
| Namespace and NAS PV | Add immich Namespace under cluster/namespace and a dedicated photos NFS PV under cluster/nfs-storage; include both in their kustomizations. These remain manually managed bootstrap resources. Use Retain for the NAS PV. |
| Argo application | Add ns-immich/base and ns-immich/overlays/prod. Existing ns-argo/application-set.yaml discovers this directory and creates Application immich targeting namespace immich. Do not add a competing hand-written Application or Helm release. |
| Components | One replica each of server, machine learning, Valkey and PostgreSQL; ordinary Deployments fit the existing monitoring model. Use Recreate for single-writer local state and avoid concurrent database writers. No database operator or HA work. |
| Application data | Dedicated NAS PVC explicitly bound to photos PV, mounted at the chosen release's data directory. Keep originals, generated assets and built-in dumps there. The person/year/month layout is inside Immich's managed library subtree, not necessarily at the export root. |
| Local data | PostgreSQL local-path PVC. A persistent local model cache avoids repeated model downloads; cache is regenerable, unlike the database. Check actual disk use including images, cache and database/WAL. Follow selected upstream Valkey persistence defaults and document job-queue loss/retry behavior after restart. |
| Secrets | Use existing Sealed Secrets workflow for database credentials. Verify sealing-key backup coverage before sealing. Private bootstrap inputs stay out of logs/chat/Git. Application users live in the database, not manifest plaintext. |
| Application configuration | Generated ConfigMap with minimal supported overrides and a content hash to roll Pods. Mount IMMICH_CONFIG_FILE; use simple supported resource/concurrency controls where evidence requires them to protect playback. Avoid custom schedulers. Use Europe/Amsterdam for intended local schedule and template date handling, verifying midnight/date-offset samples. |
| Ingress | Traefik HTTP host immich.home.arpa, ClusterIP services internally. Only server HTTP gets ingress; database, Valkey, ML and metrics remain internal. Validate WebSockets and a large video upload; do not copy nginx-only annotations from the chart. |
| Monitoring | Add direct static Prometheus scrapes for Immich's actual internal endpoints and Git-provisioned application panels. Do not install Prometheus Operator or ServiceMonitor CRDs just because the chart uses them. |

Use the chart's server/ML split, service ports, probes and data/config wiring
as references, and release-specific Compose for the compatible database image,
extensions and initialization. Translate health checks into Kubernetes probes;
Compose depends_on is not a readiness mechanism. Include PostgreSQL shared
memory requirements and correct storage ownership in the translation.

The chart inspected during planning used server 2283 with /api/server/ping,
ML 3003 with /ping, and server metrics 8081/8082 with
IMMICH_TELEMETRY_INCLUDE=all. Revalidate these against a fixed release/chart
revision before writing manifests. Record that provenance in the application
README. Do not install from mutable main/release/latest tags or copy today's
main Compose as if it were a released compatibility contract. Pin all images
by version and digest; keep server and ML on the same selected release.

## Sequential implementation steps

Execute one numbered step per owner invocation, then stop. The cross-cutting
sections below supply requirements for these steps, not additional permission
to continue into a later step. Each step is intended to fit one focused agent
session; unforeseen issues may require resuming the same step.

### Step 1 — Inspect prerequisites and establish the resource/release baseline

**Changes allowed:** planning evidence and durable fact corrections only.
No cluster, NAS or workload configuration changes.

- Start with `kubectl get nodes -o wide`. Inspect Argo state, Pods, PVCs,
  StorageClasses, events, effective resource settings and `kubectl top`.
- Use the monitoring runbook's credential-free Prometheus queries for retained
  CPU/memory/disk history. Inventory media containers and other node demand;
  distinguish typical/peak measurements, missing activity and data gaps. Check
  VM CPU instruction support and available storage. Record what playback modes
  are observable; ask the owner only for usage facts that cannot be inspected.
- Select a supported stable Immich release, compatible database/extensions,
  Valkey and fixed chart reference. Review release-specific Compose, probes,
  mounts, memory/shared-memory needs and configuration. Resolve image digests;
  record exact provenance and revalidation needs if later sessions are delayed.
- Produce an initial capacity allocation proposal with request/limit rationale,
  OS/system allowance, media priority and estimates for Immich. Do not invent
  observed Immich consumption. No GPU passthrough or VM resizing in this step.
- Identify exact metrics available for requests/limits/allocatable capacity and
  any narrowly scoped collection additions needed for useful contention views.

**Complete when:** evidence, selected references, initial sizing rationale and
feasibility conclusion are recorded in plans/immich-step-1.md. If capacity or
missing owner facts prevent a defensible design, record the blocker and stop;
do not silently promise isolation or proceed with oversized requests.

### Step 2 — Add resource-allocation and headroom visibility

**Changes allowed:** relevant monitoring configuration, dashboards and docs.
Do not change media resources or deploy Immich yet.

- Implement the resource views specified below using actual exposed series.
  Show absent requests/limits explicitly for existing media workloads.
- Where necessary, add narrowly scoped metrics for diagnosing configured CPU
  limits/contention; validate permissions and cardinality. Do not add a new
  monitoring system or notifications. Explain any limitations left in the views.
- Render/validate Prometheus and dashboard configuration, review diff, deploy
  through the existing Argo workflow and verify actual Grafana queries and
  original scrape targets. Preserve existing monitoring history/configuration.
- Update monitoring operations and query guidance.

**Complete when:** dashboards show real media usage, configured resources,
node allocation and actual headroom with truthful missing-data behavior; checks
and evidence are recorded. No need to wait for new multi-day history if existing
history suffices. Record gaps instead of claiming unobserved peaks.

### Step 3 — Apply and validate media resource settings

**Changes allowed:** media CPU/memory requests/appropriate limits, supporting
resource documentation and necessary corrections to step 2 views.

- Recheck step 1 sizing against current state and step 2 dashboards. Choose
  per-container values based on evidence, preserving playback priority and
  capacity reserved for the proposed Immich deployment and system services.
- Render/dry-run/diff media changes and deploy in controlled workload groups
  through Git/Argo. Do not restart every media application in one uncontrolled
  batch. Avoid interrupting active playback; if activity cannot be determined,
  request a suitable owner window rather than assuming inactivity.
- Verify scheduling, readiness, restarts/OOMs and each changed application's
  basic function. Validate representative playback with the owner where needed.
  Do not change images, data paths or unrelated application settings.
- Observe configured resources in Grafana, explain results and document values'
  rationale. Roll back bad resource settings through Git, never by deleting data.

**Complete when:** media has documented, validated resource settings and working
playback, with room for the planned Immich allocation. Owner-dependent playback
checks still pending mean this step is waiting, not silently complete.

### Step 4 — Prepare and validate the photos NFS storage

**Changes allowed:** photos Shared Drive NFS configuration, manual namespace/PV
bootstrap manifests, temporary scoped tests and storage/setup records.

- With the owner, inspect UniFi and confirm export path/address, quota/capacity,
  NFS version, Collaborative Mode and supported client restrictions. Configure
  only photos; leave the existing media export intact. No snapshots or offsite
  work. Record verified settings in the appropriate setup guide/system record.
- Add immich namespace and a dedicated Retain photos PV under cluster/, render,
  review and manually apply bootstrap resources per existing ownership rules.
- Use a clearly temporary test claim/Pod against the dedicated export, with the
  intended application identity, to test create/read/write/rename/delete and
  replacement/remount. Verify direct UniFi/SMB browsing. Use disposable paths
  only; document cleanup and final PV binding state. Do not leave a Released PV
  with a stale test claimRef that will prevent the later production claim from
  binding. No real user library exists at this stage; never generalize test
  cleanup to existing claims or production data.
- Do not publish an ns-immich/overlays/prod directory on main yet.

**Complete when:** export permissions and access work from Kubernetes and the
owner can inspect test files through NAS tools; test artifacts are cleaned up,
bootstrap resources are ready for production binding and precise evidence is
recorded. Storage readiness is not a backup/restore test.

### Step 5 — Prepare the complete Immich deployment without releasing it

**Changes allowed:** application manifests, tests/docs, encrypted credentials,
Renovate grouping and consistent policy changes on a dedicated preparation
branch. No Immich deployment and no production metrics scrape activation yet.

- Create ns-immich/base and overlays/prod on a named preparation branch. Record
  it in the ledger. Never merge/push that overlay to main in this step: the
  existing ApplicationSet would discover and deploy it immediately. Push the
  preparation branch for durability; a draft PR may be created if appropriate.
- Implement the architecture, resource allocation, data/config wiring, probes,
  secrets and private HTTP ingress. Confirm sealing-key backup coverage before
  generating ciphertext. Configuration must render without plaintext secrets.
- Prepare Immich-specific telemetry scrapes/panels and expected-target updates
  on this same branch for release with the application. Keep metrics internal.
- Implement narrowly scoped server/ML Renovate grouping and reconcile every
  affected one-image policy, including AGENTS.md and operations docs. Validate
  config/extraction; PostgreSQL/Valkey remain independent and automerge stays off.
- Prepare fresh-install, native-backup, recovery and mobile onboarding docs;
  complete as much of the update runbook as can be verified statically.
- Render all affected overlays, validate config/provenance/digests and run
  server dry-runs where prerequisites exist. Verify storage binding and PVC
  protection annotations. Record a deployment-ready diff and exact branch tip.

**Complete when:** a coherent, reviewed and statically validated branch is ready
for step 6, the live system has no new Immich workloads, and the ledger records
its branch/commit and any runtime-only validation still required. Do not make
an unfinished application reachable just to claim a deployment check passed.

### Step 6 — Release Immich and verify infrastructure operation

**Changes allowed:** release the prepared branch, correct deployment problems,
activate prepared metrics and complete infrastructure-level documentation.

- Verify the recorded branch, clean checkpoint, current main and prerequisites.
  Resolve/revalidate conflicts without discarding other work. Recheck selected
  release/mobile compatibility if time has passed. Verify no unexpected storage
  mapping changes. Confirm namespace/PV and sealing-key readiness.
- Review the concrete deploying diff, follow current branch/PR rules and release
  to main. This step is authorized to deploy via Argo; do not manually apply
  Argo-owned workloads. Ensure immediate first-admin bootstrap is possible with
  private owner inputs; do not leave an unclaimed installation exposed between
  sessions. Configure the owner's storage label before any sample upload.
- Wait for Application immich Synced/Healthy, PVC binding and component
  readiness. Verify exact images, file-managed settings, internal service
  connectivity, HTTP ingress and original/new metrics targets. Inspect migration,
  permissions and OOM/restart issues. Update runbook commands against reality.
- Keep full-library backup disabled. This step establishes infrastructure, not
  successful phone backup or completed photo import.

**Complete when:** running deployment, claimed admin account, effective settings,
metrics and infrastructure checks are verified and documented. Record remaining
sample-validation work explicitly. If bootstrap requires owner presence, pause
before exposing an unclaimed installation, then resume this same step.

### Step 7 — Validate phone samples, native backups and playback coexistence

**Changes allowed:** owner-assisted account/mobile setup, disposable samples,
necessary resource/concurrency tuning, documented corrections and smoke tests.

- Run the representative sample checks below: original files, edits, dates,
  locations, video/Live Photo, storage layout and independent deletion. Resolve
  unexpected edited-file behavior with the owner before allowing full import.
- Verify direct NAS file access and persistence through scoped application Pod
  replacement/remount without deleting claims or disrupting the NAS itself.
- Trigger/verify the native database dump, NAS location, schedule and retention.
  Prove creation, not restorability; no isolated restore exercise.
- Test representative media playback while uploading/indexing samples. Verify
  usable applications, resource behavior and meaningful dashboard evidence.
  Apply simple supported adjustments only if needed. No custom schedulers or
  automatic throttling controllers. If capacity is insufficient, obtain an
  owner decision rather than claiming success from low-load testing.
- Check existing VPN access only if readily testable without changes; otherwise
  record that optional access as unverified. Do not block home-only acceptance.

**Complete when:** mandatory sample, dump and simultaneous-playback checks pass
(or an explicit owner decision revises a failing requirement), with evidence
and finalized operational commands recorded. Wait for required owner phone
checks rather than labelling them complete. Full library import is still not
part of this agent step.

### Step 8 — Audit the repository and deliver the owner handoff

**Changes allowed:** final documentation/policy reconciliation and bounded fixes
needed to satisfy already agreed acceptance. No new features or scope expansion.

- Audit all step evidence against this plan. Verify expected deployed state,
  Argo health and final render/config checks; do not repeat disruptive tests
  without a reason. Resolve any incomplete mandatory acceptance from prior steps.
- Finalize README.md, setup.MD, Argo/monitoring/Immich guides, system records and
  TODO status. Ensure separate offsite/HTTPS work remains tracked, and no old
  resource policy or one-image instruction contradicts the accepted exceptions.
- Complete and validate plans/immich-update.md against the installed setup,
  including exact backup/smoke links and fixed upstream reference versions.
  Mark that update runbook ready without executing an actual upgrade.
- Verify Renovate extraction/grouping evidence. If waiting on a hosted scan
  needed to prove functionality, record waiting and resume; do not claim a
  successful scan from configuration alone.
- Give the owner full-import, progress/error checks and later spouse-onboarding
  instructions. Record post-full-import resource review as owner-triggered
  follow-up; do not wait indefinitely for the entire library in this step.

**Complete when:** all required implementation/sample evidence and docs are
consistent, the update runbook is ready, and the ledger marks all eight steps
complete. Report implementation complete, full import owner-led/pending,
recovery untested, and offsite/HTTPS/snapshots outside this completed scope.
Stop. A future "next step" prompt must report no implementation steps remain,
not start full import, onboard the wife or begin another TODO automatically.

## Monitoring and dependency-update details

### Resource requests and capacity visibility — expanded scope

The media workload manifests inspected during planning have no container
CPU/memory requests or limits (PVC storage requests are unrelated). Inspect
live admission defaults and effective resources rather than assuming manifests
alone describe runtime settings. Inventory every media/Immich container and
base CPU/memory requests on retained usage history, representative activity,
startup behavior and upstream guidance. Do not distribute resources equally
or calculate requests from idle-only snapshots. Record rationale and observation
windows in the implementation evidence; distinguish estimates for new Immich
components from measured media behavior. Recheck after sample indexing and
after owner-run full import. Explain findings along the way.

Extend Git-managed Grafana views for both namespaces to show CPU cores and
memory working set against requests, configured limits where present, observed
peaks, restarts/OOM evidence and node pressure. Show per-workload values and
aggregate requested versus node-allocatable resources across all namespaces,
alongside real VM available memory/CPU utilization. Leave allowance for host
services and transient rollout/startup demand. Requests are scheduler accounting,
not consumption caps; exceeding a request alone must not be labelled failure.
Missing requests/limits/metrics must be explicit, not divided by zero or
displayed as healthy headroom. Unallocated requests and actual available memory
are different measurements and need separate labels.

Existing kube-state-metrics Pod/Node collection may supply request/limit and
allocatable series; verify actual availability and pinned-version semantics.
The current kubelet /metrics/resource scrape retains only CPU usage and memory
working set. It cannot prove CPU throttling is absent. If diagnosing throttling
or contention needs additional metrics, design a narrowly scoped collection
change and verify its permissions/cardinality rather than implying the current
dashboard establishes that every application has enough resources.

Apply media resource changes in controlled rollout steps with application smoke
checks; changing Pod resources can restart stateful services. Include media
render/diff checks, Argo reconciliation and scheduling verification in acceptance,
and ensure requests fit effective allocatable capacity before rollout. Do not
alter existing monitoring/cluster resource policies incidentally. Update the
monitoring guide and application operations documentation for the new views and
the method used to choose/revisit requests.

Validate representative media playback (including transcoding if normally used)
alongside Immich upload/indexing. Observe CPU, memory, relevant disk/network
load and application responsiveness; adjust to protect playback without
starving essential Immich services or causing memory-limit restart loops.
Account for all media workloads: giving every download/scanning job precedence
is not equivalent to protecting movie playback. Use supported Immich scheduling
or concurrency controls if needed; do not promise arbitrary overnight scheduling
without verifying the chosen release supports it. Validate settings against real
behavior before implementing a restrictive schedule. The owner is equally
comfortable with prompt or overnight photo uploads and prioritizes simplicity.
There is no requirement for an overnight-only upload window. Prefer normal
automatic uploads and straightforward supported resource or concurrency controls
as necessary to protect playback. Do not build custom upload schedulers or
dynamic throttling infrastructure. Overnight processing is an acceptable option,
not a promised capability or mandatory schedule.

### Application metrics and dependency updates

Existing container/resource and object-state collection is cluster-wide, so
new Deployments should appear without namespace-specific collector changes.
Verify this in the overview's namespace/workload selectors. If implementation
uses StatefulSets instead, extend collector RBAC/resources and owner-aware
dashboard queries deliberately; do not leave the database invisible.

Add Immich metrics to ns-monitoring/base/prometheus.yaml using internal service
DNS. Inspect the selected release's telemetry and series before choosing panels
(API performance and available processing metrics). Include scrape health and
document what the metrics cannot prove, especially upload completeness and
backup freshness. Update the overview's fixed expected-target panel so newly
configured jobs can be distinguished as healthy, down or missing. Keep current
panels working; do not enable tracing infrastructure or add speculative metrics.
Explain local VM disk and NAS quota/capacity checks separately: current VM
exporters do not establish remaining photos-share quota or NAS health.

Root renovate.json already matches infrastructure/node-main YAML and disables
automerge. Validate actual extraction for every new image, including database
tags containing PostgreSQL/extension versions. Add narrowly scoped extraction
or versioning rules only where necessary; a PostgreSQL major-version upgrade
is a migration, not an ordinary image replacement. Verify Dependency Dashboard
or equivalent extraction evidence and document any pending hosted scan.

The owner approved a narrowly scoped exception to the repository's
one-digest-at-a-time policy: update immich-server and immich-machine-learning
together as one manually reviewed release, with both exact digests pinned,
a fresh application-native backup, rollout checks and smoke tests. Keep their
release versions aligned. PostgreSQL and Valkey updates remain separate.
Make this explicit in a Renovate package rule matching only those two image
repositories, with a descriptive group name and rule description; verify that
it overrides the existing ungrouped-image rule. Keep automerge disabled.
Reconcile AGENTS.md, ns-argo/SETUP.md and any other blanket one-image update
instructions so they state or link this exception; document it in the Immich
README and near image declarations where useful. Do not silently broaden the
exception to other applications. These policy/configuration edits belong to
implementation, before the first Immich dependency update. An automatically
updated iPhone is not authorization for unattended server changes. Never
blindly downgrade against a migrated database.

## Acceptance and owner checklists

- Argo immich is Synced/Healthy; Pods and PVCs are healthy; existing applications
  and monitoring still work. Sample files survive a planned application Pod
  replacement and NAS remount without deleting claims or disturbing the NAS.
- iPhone and browser reach http://immich.home.arpa on home Wi-Fi/LAN. Existing
  VPN access is checked opportunistically with no VPN changes; record failure
  as a limitation rather than expanding scope.
- Test a representative HEIC/JPEG, video, Live Photo, edited photo and assets
  with known dates/locations. Compare appearance, resolution, metadata and
  downloaded files against the intended source. Confirm physical NAS files
  and person/year/month layout using UniFi/SMB; filesystem modification time
  is not a substitute for photo capture metadata. Check collision handling.
- Test phone-only deletion using a disposable asset and prove the Immich copy
  remains. Avoid bulk deletion, storage-freeing tools or changes to real assets.
- If the mobile app uploads an unedited original where the owner wants the
  edited rendition, establish a supported export/import path with the owner
  before bulk import. Keep dates/locations; avoid silent recompression or loss
  of originals. Do not claim all Apple metadata is preserved without evidence.
- Verify face/search processing on suitable sample assets and observe usage
  with the selected resource/concurrency settings. Explain what the owner is
  seeing and verify playback remains responsive.
- Trigger a database dump and verify successful completion and a nonempty
  backup file on the NAS. Verify daily configuration and retention. Do not
  claim a full 14-day history or scheduled success before it has occurred.
  No isolated restore exercise is required or authorized by this plan.
- Owner full-import checklist: confirm local originals/download state and
  sufficient phone space, select the entire library including videos, enable
  background refresh and Wi-Fi backup, keep the app open/on power as needed,
  check remaining/error counts, sample older and newer files/metadata, and
  verify that new photos subsequently upload. Counts can differ for Live
  Photos/deduplication; explain differences instead of declaring success from
  one aggregate count. Keep existing originals/iCloud throughout this stage.
- Spouse onboarding follows owner confirmation: create a separate account and
  readable storage label, verify account isolation and sample uploads, then
  repeat the same full-import checklist. Do not enable partner sharing by
  default or solicit her credentials in chat.

## Documentation, recovery and completion

Maintain [the dedicated update instructions](immich-update.md) so the owner can
ask a clean agent session to execute an update. The owner explicitly requested
this during critical review. Initial implementation must complete its actual
paths, tested backup/smoke procedures and fixed upstream references, then mark
it ready. Link it from the application README. Every release review must compare
relevant release notes and Compose/chart changes, not only image references.

Create ns-immich/README.md with fresh installation, NAS prerequisites, ownership,
account/bootstrap steps, mobile setup, file layout, checks, monitoring, native
backup, compatible upgrades and recovery instructions. Document the difference
between code-managed settings and database-held account/library state.

Document recovery paths for local database loss, NAS file loss and whole-VM
restore. Use compatible versions and matching paths/files; coordinate Argo
automation before restoring. Reference the existing ordinary/guarded VM restore
procedures instead of inventing a conflicting one. Include recreatable caches,
required database dumps, originals and any non-regenerable metadata. A dump
on photos does not protect against NAS loss; no independent recovery or
application-consistent restore has been demonstrated. No restore test is part
of initial completion. Files must not be rearranged manually underneath Immich.

Update README.md, setup.MD, ns-argo/SETUP.md, ns-monitoring/README.md and
plans/TODO.md so application lists, bootstrap storage, recovery coverage and
links remain consistent. Preserve unrelated history with accurate status.
Move durable owner decisions into an appropriate Immich system.md during
implementation, linking node/network records; do not duplicate manifests,
runtime values or temporary capacity estimates there. Update shared system
records only for facts within their scope. Update this plan's execution status
and verification evidence as work progresses. Check all touched links, render
commands and setup steps before declaring completion.

The final implementation report must distinguish verified deployment/sample
behavior, owner-run full import still pending, recovery untested, and offsite
backup/snapshots deferred. Do not mark the long-term iCloud objective complete.

## Reference sources

- https://docs.immich.app/install/docker-compose/
- https://docs.immich.app/install/kubernetes/
- https://github.com/immich-app/immich-charts
- https://github.com/immich-app/immich-charts/tree/main/charts/immich/templates
- https://docs.immich.app/install/requirements/
- https://docs.immich.app/install/config-file/
- https://docs.immich.app/administration/storage-template/
- https://docs.immich.app/administration/backup-and-restore/
- https://docs.immich.app/features/mobile-backup/
- https://docs.immich.app/features/monitoring/
- https://docs.immich.app/FAQ/
- https://help.ui.com/hc/en-us/articles/26277250895895-Accessing-Your-UniFi-Drive-from-Linux-Desktop-Using-NFS

## Facts to verify during execution, not assumptions

No live cluster or NAS inspection was performed during this planning session.
The implementing agent must discover current headroom, installed versions,
export path/permissions/quota, supported config/API behavior, compatible image
digests, mobile edited-file behavior, completed originals download and VPN
reachability. These are concrete preflight/sample checks, not permission to
guess or silently change the agreed scope. Ask the owner only for facts that
cannot be inspected or decisions needed if a requirement proves infeasible.

## Critical review — accepted decisions

The following review does not override the confirmed choices above.

- Accepted review decision: group release-coupled Immich server/ML images into
  one reviewed release update. Make the exception explicit in configuration and
  documentation, reconciling AGENTS.md, Renovate and operational update guidance.
  Keep PostgreSQL/Valkey separate and automatic merging disabled.
- Accepted review decision: before reducing existing phone/iCloud copies to
  rely primarily on Immich, require independent versioned offsite backup and
  one successful recovery test. This does not expand initial deployment scope.
- Accepted review decision: retain HTTP for initial deployment and add separate
  trusted-HTTPS work across internal applications, including Immich. Home-only
  access does not provide application transport encryption.
- Accepted expanded scope: reasonable resource requests for Immich and media,
  plus appropriate limits and Grafana views for usage, allocation and headroom.
  Media playback takes priority; Immich background work may slow or wait until
  overnight. This supersedes the earlier no-special-tuning preference.
  Current requirements list 6 GB minimum/8 GB recommended RAM for
  the application; the shared VM has 16 GB allocated. No live-headroom conclusion
  has been established. Choose actual settings from evidence and simultaneous
  playback/import validation, not equal per-application allocations.
- Accepted review decision: standard mobile backup is the default. Verify edited
  samples and discuss any limitation before full import; no custom migration
  tooling or automatic requirement to store duplicate original/edited versions.
- Accepted review decision: keep plain manifests and require release-note plus
  relevant Compose/chart comparisons during every update. Provide dedicated
  clean-agent instructions in plans/immich-update.md, completed and validated
  against the initial installed setup before being marked ready.

Review sources:
- https://docs.immich.app/install/upgrading/
- https://docs.immich.app/install/requirements/
- https://docs.immich.app/administration/backup-and-restore/
- https://support.apple.com/en-gb/108782
- https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Security_Cheat_Sheet.html
