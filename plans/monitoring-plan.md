# Monitoring — completed

Status: **complete**, closed 2026-10-21. The agreed VM/Kubernetes monitoring
scope is deployed and verified. No lesson or implementation step remains active.
Detailed deployment history and the original teaching plan remain in Git.
Use [monitoring operations](../infrastructure/node-main/ns-monitoring/README.md)
for maintenance, queries, dashboard edits and recovery.

## Delivered and verified

- Prometheus collects self-metrics, Ubuntu VM metrics, ens18 network traffic,
  Kubernetes object state and container CPU/memory through five scrape jobs.
- Grafana requires login and has a provisioned Prometheus data source. The
  Git-managed **Monitoring - Overview** has 19 metric panels in System, Cluster
  and Workloads sections; the editable learning dashboard remains separate.
- Dashboard queries, units, single/multiple namespace and Deployment selections,
  and missing-data behavior were checked against real series and in Grafana.
  Missing samples remain distinct from measured zeroes.
- Grafana Pod replacement preserved the saved learning dashboard and its PVC.
  Dashboard provisioning landed in `4bcf858`; the complete layered dashboard
  deployed in `d8fbdcd`. Both changes passed Standards and Spec reviews.
- At final deployment verification, Argo Applications were Synced/Healthy,
  all five targets were Up, both monitoring claims were unchanged and Bound,
  and all eight media Deployments remained available.
- Resource, storage, scrape-error and retention checks completed. No OOM state
  or monitoring Warning events were observed; no CPU throttling counters
  increased during the 158-second sample. Lifetime throttling was nonzero for
  some containers. The final 15-minute scrape window was clean; the preceding
  hour contained one Node Exporter failure around its rollout.
- Effective retention was one week / 7 GiB. The low-usage snapshot was about
  16m CPU and 258 MiB monitoring RAM, with 32.2 MiB Prometheus data and 52.5 MiB
  Grafana data. These are historical observations, not capacity guarantees.
- Recovery, configuration rollback and future stateful-update procedures are
  documented. The owner-confirmed backup policy is in the
  [node record](../infrastructure/node-main/system.md#recovery-sources).

## Accepted limits

- The owner explicitly skipped further Pod-replacement exercises. Prometheus
  replacement was not tested; do not resume that exercise automatically.
- Manual dashboard practice was superseded by the requested implementation
  pass. The full Lesson 4 teaching checkpoint was not separately assessed.
- Full seven-day observed retention/growth, peak load, VM backup restoration
  and application consistency after restore have not been tested. These remain
  validation limits, not unfinished steps in this deployment.
- Infrastructure health does not prove media playback, search or downloads.

Optional Proxmox/NAS/UPS metrics, notification delivery and recovery validation
are separate future topics in [Next steps](TODO.md). Notification design must
account for the monitoring VM being unable to report its own host's outage
without an external observer.
