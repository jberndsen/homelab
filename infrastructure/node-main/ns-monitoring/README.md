# Monitoring operations

Implementation follows [the approved plan](../../../plans/monitoring-plan.md),
one lesson at a time. Teaching takes place exclusively in conversation;
this file contains operations notes only.

Lessons 1–2 are deployed and verified. Resume at Lesson 2's owner checkpoint
using the plan's continuation instructions. Evidence and initial measurements
are recorded in the plan. Lessons 3–5 remain pending.

## Ownership and updates

`cluster/` manually owns the monitoring namespace. The existing ApplicationSet
discovers this production overlay and Argo deploys its contents after push to
`main`. Do not manually apply monitoring workloads: self-healing would correct
drift. Review rendered output and the diff before pushing.

After Argo takes ownership, `kubectl diff` can show removal of Argo-injected
tracking annotations because the local render lacks them. This read-only check
does not remove annotations. Inspect Argo's comparison for its normalized diff;
do not manually apply the local render to make that metadata difference vanish.

```sh
kubectl kustomize infrastructure/node-main/ns-monitoring/overlays/prod
kubectl apply --dry-run=server -k infrastructure/node-main/ns-monitoring/overlays/prod
kubectl diff -k infrastructure/node-main/ns-monitoring/overlays/prod
kubectl -n argocd get application monitoring
kubectl -n monitoring rollout status deployment/prometheus
kubectl -n monitoring rollout status daemonset/node-exporter
kubectl -n monitoring port-forward svc/prometheus 9090:9090
```

Open `http://localhost:9090/targets`, `/metrics` and `/query`.
Query `up{job="prometheus"}` and `prometheus_build_info`. A successful scrape
does not establish application correctness. The Service is internal; there is
no Prometheus ingress. Stop port-forward with Ctrl-C when finished.

Node Exporter is internal at `node-exporter.monitoring.svc.cluster.local:9100`.
Check `up{job="node-exporter"}`, `node_scrape_collector_success` and
`node_filesystem_device_error`; scrape success alone does not prove every
collector succeeded. The exporter observes the Ubuntu VM, not Proxmox.
The Service and explicit node label are single-node scoped: change to per-node
discovery before broadening the DaemonSet to multiple nodes.

Host procfs/root mounts require recursive read-only runtime support and do not
propagate new host mounts. Replace only the exporter Pod after host mount changes
and verify filesystem series. Keep recursive read-only access enabled. Host
network/PID namespaces and API credentials are not needed for the currently
enabled collectors. VM network collection is deferred to the dashboard lesson;
do not enable namespace-sensitive collectors without validating their host view.

ConfigMap generation retains content hashes, so config edits trigger a new Pod.
Recreate strategy prevents concurrent database writers. History lives on the
PVC, not in the ConfigMap or Grafana. Retention is an upper bound, not a
seven-day guarantee or disk quota; keep room for WAL, head and compaction.

PVC confirmation annotations protect against Argo prune/deletion only.
Direct deletion bypasses them; never delete a claim to test persistence or
roll back. Restore configuration through Git and wait for Argo. Do not blindly
downgrade a database image. Future stateful image updates require release review,
an application-native backup, one digest update, rollout and smoke test.

History is disposable after storage loss. Reinstall configuration from Git;
do not assume verified backup coverage for a new PVC. Persistence testing and
the broader recovery lesson are scheduled for Lesson 5.
