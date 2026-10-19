# Monitoring lessons and operations

Implementation follows [the approved plan](../../../plans/monitoring-plan.md),
one lesson at a time. Begin with [Lesson 1: the collection loop](../../../lessons/0001-prometheus-collection-loop.html).
Owner checkpoints record readiness to continue; deployment alone does not establish learning.

## Ownership and updates

`cluster/` manually owns the monitoring namespace. The existing ApplicationSet
discovers this production overlay and Argo deploys its contents after push to
`main`. Do not manually apply monitoring workloads: self-healing would correct
drift. Review rendered output and the diff before pushing.

```sh
kubectl kustomize infrastructure/node-main/ns-monitoring/overlays/prod
kubectl apply --dry-run=server -k infrastructure/node-main/ns-monitoring/overlays/prod
kubectl diff -k infrastructure/node-main/ns-monitoring/overlays/prod
kubectl -n argocd get application monitoring
kubectl -n monitoring rollout status deployment/prometheus
kubectl -n monitoring port-forward svc/prometheus 9090:9090
```

Open `http://localhost:9090/targets`, `/metrics` and `/query`.
Query `up{job="prometheus"}` and `prometheus_build_info`. A successful scrape
does not establish application correctness. The Service is internal; there is
no Prometheus ingress. Stop port-forward with Ctrl-C when finished.

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
