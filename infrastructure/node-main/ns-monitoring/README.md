# Monitoring operations

Implementation follows [the approved plan](../../../plans/monitoring-plan.md),
one lesson at a time. Teaching takes place exclusively in conversation;
this file contains operations notes only.

Lessons 1–3 are deployed, verified and complete; the owner passed Lesson 3's
checkpoint. Lesson 4 is in progress. See the plan for validation evidence and
resume instructions. Lesson 5 remains pending. Lesson port-forwards were stopped
at evening cleanup; reconnect only when needed.

## Grafana (Lesson 4 in progress)

Open http://grafana.home.arpa on the trusted LAN. Login is required (username
admin). Copy the initial password privately in the owner's Mac terminal and
save it in the password manager; this pipeline prints no credential:

```sh
kubectl -n monitoring get secret grafana-admin -o jsonpath='{.data.admin-password}' | base64 -d | pbcopy
```

Do not paste it into chat. After a password change, use the password manager;
the Secret retains the original bootstrap value. For troubleshooting only:

```sh
kubectl -n monitoring port-forward svc/grafana 3000:3000
kubectl -n monitoring rollout status deployment/grafana
```

The localhost forward must be reconnected after Pod replacement. Ingress,
real session login, anonymous API denial, data-source health and the query
`up` through Grafana were verified. The owner also ran `up` in Explore using
Instant and Range modes with legend `{{job}}`. The owner has now saved the
editable Monitoring — Learning dashboard with Scrape health and demonstrated
VM CPU, memory and root disk panels. Manual dashboard practice is paused;
the owner wants the agent to create the final dashboard later. Lesson 4
remains incomplete; see the plan for remaining work and verification limits.

Grafana uses the protected 1 GiB local-path grafana-data PVC and Recreate
strategy. Its provisioned Prometheus data source (UID prometheus) connects
server-side to http://prometheus:9090, with a 30-second scrape interval setting.
The configuration hash triggers rollout after data-source configuration edits.
Editable dashboards live in Grafana's database; UI edits do not update Git.
Dashboard provisioning and Pod-replacement testing remain Lesson 5.

Strict-scope grafana-admin supplies the initial admin-user/admin-password.
Anonymous access and self-signup are disabled. The bootstrap password only
initializes a new database: later password changes are database state, and a
Secret edit alone does not reset an existing account. Never print credentials
in tool output. Keep the password in the owner's password manager.

Plugin preinstallation and automatic plugin updates are explicitly disabled.
The initial startup downloaded optional plugins into the data PVC before this
default was corrected; no plugin auto-update mechanism is left enabled.

Sealing-key coverage was checked against the verified NAS inventory before
sealing; see the plan and ns-argo/SETUP.md for the recovery workflow. No new
Grafana application-data backup coverage or persistence test is claimed.

## Read-only cluster inspections

Start with `kubectl get nodes -o wide`, Argo Applications and Pods across all
namespaces. Inspect events and `kubectl top nodes` / `kubectl top pods -A` for
current problems and recent consumption. Use Prometheus for stored history;
Grafana Explore uses the provisioned Prometheus source. No saved dashboard is
required. A query can also run without login credentials or a port-forward:

```sh
kubectl -n monitoring exec deployment/prometheus -- wget -qO- 'http://localhost:9090/api/v1/query?query=up'
```

For other queries, URL-encode PromQL in the `query` parameter, or use Grafana
Explore / a temporary localhost Prometheus forward and close it afterward.

| Question | PromQL |
| --- | --- |
| Scrape health now | `up` |
| Observed scrape failures in 15 minutes | `min_over_time(up[15m]) < 1` |
| VM CPU busy percent over 5 minutes | `100 * (1 - avg(rate(node_cpu_seconds_total{job="node-exporter",mode="idle"}[5m])))` |
| VM available RAM in GiB | `node_memory_MemAvailable_bytes{job="node-exporter"} / 1024^3` |
| VM root available disk in GiB | `node_filesystem_avail_bytes{job="node-exporter",mountpoint="/"} / 1024^3` |
| Exporter collector failures | `node_scrape_collector_success{job="node-exporter"} == 0` |

Object-state and media CPU/memory queries are in the sections below. Preserve
namespace/Pod/container identity when investigating an app. Compare with kubectl;
sample times/windows differ. Check current targets as well as `up`: a missing
series is not a healthy zero, and disappeared targets can leave stale history.
Node Exporter observes the Ubuntu VM, not Proxmox. VM network metrics cover
ens18; Proxmox/NAS/UPS exporters are not configured. Neither scrape success nor a Ready
Pod proves playback, search, downloads or backup/recovery work.

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
enabled collectors.

VM network counters come from a separate netdev-only Node Exporter process in
the same Pod, exposed internally on Service port 9101 and scraped by job
`node-exporter-network`. It uses `--path.procfs=/host/proc/1` with netlink
disabled, so it reads the VM PID 1 network view rather than the Pod's eth0.
Only ens18 is included, avoiding double-counting virtual bridges/interfaces.
Other collectors must retain `/host/proc`; PID 1's directory is only suitable
for this dedicated netdev process. The configured procfs backend is documented
in the [pinned collector source](https://github.com/prometheus/node_exporter/blob/v1.12.1/collector/netdev_linux.go).
Both byte counters were compared with surrounding raw host counter reads.

```promql
rate(node_network_receive_bytes_total{job="node-exporter-network",device="ens18"}[5m])
rate(node_network_transmit_bytes_total{job="node-exporter-network",device="ens18"}[5m])
```

These rates are bytes per second at the VM interface, not per-application traffic
or physical Proxmox link throughput. Monitor both `up` and
`node_scrape_collector_success{job="node-exporter-network",collector="netdev"}`;
also require the expected ens18 series to exist. A renamed/missing interface
can produce no network series despite a successful collection. Revalidate the
VM view before changing interface selection or enabling other network collectors.

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

## Kubernetes object state

kube-state-metrics exposes only nodes, Pods, Deployments, PVs, PVCs and
StorageClasses. Its dedicated ServiceAccount has matching list/watch access;
no Secret access or arbitrary Kubernetes label/annotation export is configured.
Prometheus scrapes the internal Service on port 8080 every 30 seconds.
The exporter has no persistent data; Prometheus stores scraped history.

Useful queries:

```promql
kube_deployment_spec_replicas{namespace="media"}
kube_deployment_status_replicas_available{namespace="media"}
kube_pod_container_status_ready{namespace="media"}
kube_pod_container_status_restarts_total{namespace="media"}
```

Compare with `kubectl -n media get deployments,pods`. Restart values belong to
current Pod/container identities; Pod replacement changes that identity. Missing
series must not be interpreted as zero. These metrics describe Kubernetes state,
not media playback or search success. CPU/memory consumption comes from the kubelet job below;
metrics-server remains the source for current `kubectl top` measurements.

For collector troubleshooting, temporarily forward Deployment port 8081 and
inspect `/metrics` for `kube_state_metrics_watch_total` success/error results.
This port is used for readiness but is not an additional persistent scrape job.
Keep saved output and tools in repository-root `tmp/`; set TMPDIR there for
commands that create temporary files. Never commit temporary artifacts.

## Container CPU and memory

Prometheus discovers only node `vm-ubuntu-k3s`, using a matching metadata.name
field selector required by its name-scoped list/watch RBAC. Node discovery
supplies the kubelet IP/port. Its `kubelet-resource` job reads /metrics/resource
directly over HTTPS, verifying the serving certificate with the mounted cluster
CA. Kubernetes mounts/rotates its Pod-bound ServiceAccount token. No manually
maintained credential Secret, insecure TLS or nodes/proxy permission is used.
The only kubelet permission is get nodes/metrics for that node. Adding a node
requires deliberate RBAC/discovery changes and certificate verification.

Only container_cpu_usage_seconds_total and container_memory_working_set_bytes
are retained from this job, excluding empty namespace/Pod/container identities
and container=POD. Namespace, Pod and container names are preserved; the node
label comes from discovery. Cardinality grows with container count and Pod churn.
The broader cAdvisor endpoint was inspected but is not scraped.

```promql
rate(container_cpu_usage_seconds_total{job="kubelet-resource",namespace="media",container!="",container!="POD"}[5m])
container_memory_working_set_bytes{job="kubelet-resource",namespace="media",container!="",container!="POD"} / 1024 / 1024
```

CPU units are cores, not percent of the entire VM: 0.1 cores = 100 millicores.
Use a 1-minute window initially when fewer than five minutes of samples exist;
rate needs multiple samples and a zero differs from a missing series. Memory is
MiB of working set, including some cache, not application heap alone. Compare
plausibility with kubectl top, allowing different sample times/windows.
Prometheus supplies stored history; metrics-server supplies recent measurements.

Check up{job="kubelet-resource"} and /targets; discovery failure can leave no
target at all. Inspect Prometheus logs for discovery/RBAC errors. Certificate
errors require checking the actual endpoint, CA, SANs and validity; never solve
them by disabling verification. If the trust CA changes, replace the Prometheus
Pod after confirming its trust bundle and serving certificate. Keep credentials
out of output/files. Verify config inside the running Pod:

```sh
kubectl -n monitoring exec deployment/prometheus -- /bin/promtool check config /etc/prometheus/prometheus.yaml
```

This does not establish application playback/search health. Grafana is deployed;
its learning dashboard is unfinished. Notification delivery is a separate future
phase, and controlled persistence testing remains Lesson 5.
