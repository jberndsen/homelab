# Goal

Maintain the K3s media stack on `node-main`. Deployable Kubernetes
configuration belongs under `infrastructure/node-main`.

Consult the `infrastructure/**/system.md` records before making changes. They
hold durable facts that agents cannot discover from repository code or normal
read-only inspection. Discover facts from those sources first; when necessary
information remains unknown, do not guess: ask the owner and record the confirmed
fact in the appropriate system record. Do not duplicate manifest configuration,
discoverable runtime state, or transient capacity snapshots in system records.

Use the locally installed `kubectl` and its preconfigured kubeconfig for
cluster work. Start with `kubectl get nodes -o wide`.

For cluster inspections, use `kubectl` for current state/events and `kubectl top`
for recent usage. Prometheus now provides historical VM, Kubernetes object-state
and container CPU/memory metrics; Grafana provides Explore access. Consult
[monitoring operations](infrastructure/node-main/ns-monitoring/README.md#read-only-cluster-inspections)
for queries without retrieving application credentials or leaving a port-forward
running. VM network metrics cover `ens18`; Proxmox/NAS/UPS coverage is not yet available.

For stateful image updates, review the release, take an application-native
backup, update one pinned digest, wait for rollout, and run the application's
smoke test. Do not introduce unattended updates or runtime socket access.

# Repository

The public repository is `https://github.com/jberndsen/homelab`, with default
branch `main`. Local `origin` uses `git@github.com:jberndsen/homelab.git`.
The owner authorizes pushing task-related changes to this repository. Keep
plaintext credentials, private keys, and local reference files out of commits.
Update the relevant system records whenever facts within that scope change.

# Plans

Always keep implementation plans, investigation plans, and handoff plans in the
repository-root `plans/` directory. Deployable configuration still belongs under
`infrastructure/node-main`.

# Temporary files

Never create or write files outside this homelab repository. Put temporary
files, downloaded tools, validation output and local reference files in the
repository-root `tmp/` directory, which must remain gitignored. Configure tools
that create temporary files to use this directory (for example, set `TMPDIR`
to its absolute path). Keep temporary files out of commits.
