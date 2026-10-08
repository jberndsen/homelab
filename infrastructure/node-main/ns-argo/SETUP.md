# Argo CD operations

Run commands from the repository root using the laptop's configured `kubectl`;
start cluster work with `kubectl get nodes -o wide`. Read
[node-main](../system.md) and [network](../../network/system.md) first.
Choose a [recovery path](#choose-the-recovery-path) before rebuilding or restoring.

## Ownership and reconciliation

`cluster/` manually owns namespaces and the NAS PV. `ns-argo/` manually owns
Argo CD and its separately applied ApplicationSet. Argo never manages itself.
The pinned standard non-HA install retains upstream NetworkPolicies; upstream
workloads have no resource requests/limits, so check usage after bootstrap.

ApplicationSet `node-main` discovers `ns-*/overlays/prod` on public GitHub
`main` over anonymous HTTPS. It strips `ns-` from folder names for Application
names and destination namespaces, explicitly maps `sealed-secrets` to
`kube-system`, and excludes `ns-argo`. Go templates fail on missing keys.
An ApplicationSet creates Applications; each Application renders and syncs
its Git resources. Separate Applications do not impose startup order.

Automatic sync, pruning and self-healing are enabled. `allowEmpty: false`
blocks an entirely empty desired application, but not partial resource removal.
`preserveResourcesOnDeletion: true` and absent Argo deletion finalizers preserve
resources when a discovered Application disappears. All media PVCs, both
SealedSecrets and the Sealed Secrets CRD require `Prune=confirm,Delete=confirm`.
See [Intentional removal](#intentional-removal) before deleting resources.

## Manual bootstrap

Use this procedure for a fresh cluster. For a restored VM, follow its recovery
path first: applying installation manifests can start stopped controllers.
The Kustomization installs Argo only; apply a disabled ApplicationSet separately
as described under [Fresh cluster](#fresh-cluster).

1. Verify the node and review the cluster diff. Exit code 1 from `kubectl diff`
   means differences were found; an error has exit code greater than 1.

   ```sh
   kubectl get nodes -o wide
   kubectl kustomize infrastructure/node-main/cluster
   kubectl diff -k infrastructure/node-main/cluster
   kubectl apply -k infrastructure/node-main/cluster
   ```

   `cluster/` owns namespaces and the NAS PV manually. On an existing cluster,
   stop if the diff unexpectedly changes storage. A Namespace groups resources;
   declaring it here keeps its lifecycle outside Argo ownership.

2. Render and validate the pinned install, then apply it:

   ```sh
   kubectl kustomize infrastructure/node-main/ns-argo
   kubectl apply --server-side --dry-run=server -k infrastructure/node-main/ns-argo
   kubectl apply --server-side -k infrastructure/node-main/ns-argo
   ```

   Kustomize assembles upstream resources with our exact ConfigMap patch and
   Ingress. Server-side apply avoids storing the large CRD schemas in a
   last-applied annotation. Do not add force/replace options to resolve conflicts.
   Argo never manages this installation; changing these files requires a manual
   reapply. The ApplicationSet will be applied separately after its CRD exists.

3. Wait for the new API types and workloads:

   ```sh
   kubectl wait --for=condition=Established --timeout=60s \
     crd/applications.argoproj.io crd/applicationsets.argoproj.io crd/appprojects.argoproj.io
   kubectl -n argocd wait --for=condition=Available deployment --all --timeout=300s
   kubectl -n argocd rollout status statefulset/argocd-application-controller --timeout=300s
   kubectl -n argocd get pods,services,ingress
   kubectl top nodes
   kubectl -n argocd top pods
   ```

   A CRD registers a new Kubernetes resource type. A controller watches resources
   and repeatedly works toward their desired state. Healthy Pods show the
   controllers are running; an Application's eventual Synced/Healthy status will
   tell us whether its Git resources match the cluster and are working.

4. Open **http://argocd.home.arpa** on the trusted LAN. Traefik routes this host
   to `argocd-server` port 80; `server.insecure: "true"` disables Argo's own TLS.
   HTTP exposes credentials/session traffic to the network; restrict this to
   the trusted LAN as agreed. DNS should resolve to `192.168.30.103`.

   Retrieve the initial password in your own terminal (do not share its output):

   ```sh
   kubectl -n argocd get secret argocd-initial-admin-secret \
     -o jsonpath='{.data.password}' | base64 --decode; echo
   ```

   Log in as `admin`, open **User Info → Update Password**, and save the
   replacement in your password manager. Log out and verify login with the new
   password. Then remove the initial-password Secret:

   ```sh
   kubectl -n argocd delete secret argocd-initial-admin-secret
   ```

   The actual admin password hash lives in `argocd-secret`; deleting the initial
   Secret after changing the password removes the bootstrap copy. Never commit
   either Secret or put a password into a command argument/history.

   If you install the matching CLI, use an interactive password prompt:

   ```sh
   argocd login argocd.home.arpa:80 --plaintext --grpc-web --username admin
   ```

   `--insecure` alone still uses TLS. See the upstream
   [getting-started guide](https://argo-cd.readthedocs.io/en/stable/getting_started/)
   and [Ingress configuration](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/).

## Sealed Secrets

A SealedSecret holds encrypted configuration; its controller creates the Secret
applications read. Strict scope binds ciphertext to the exact name and namespace.
Do not rename it to move credentials. Media uses `homepage-widgets` and
`transmission-rpc`; their plaintext values and sealing private keys stay out of
Git, logs, annotations and ConfigMaps.

During bootstrap/recovery, restore verified keys before starting the controller.
In the Argo UI, review and sync only `sealed-secrets`, with prune/force/replace
off. If keys are imported after startup, restart the controller to reload them.
Wait for readiness before syncing media SealedSecrets:

```sh
kubectl wait --for=condition=Established --timeout=60s crd/sealedsecrets.bitnami.com
kubectl -n kube-system rollout status deployment/sealed-secrets-controller --timeout=300s
kubectl -n argocd get application sealed-secrets
```

The upstream default renews sealing keys every 30 days; retain old keys for
existing ciphertext.

Install laptop tools with `brew install kubeseal age`. Key recovery uses
**smb://192.168.1.32/backups/NUC/kubernetes**, mounted at
**/Volumes/backups/NUC/kubernetes**, with exact backup/restore commands in the
adjacent NAS **README.md**. The owner keeps its age passphrase in their password
manager under **Homelab Sealed Secrets key backup**, outside Git and chat.
Only encrypted `.yaml.age` backups, public certificates, fingerprint inventories
and recovery documentation belong there.
Keep private keys in protected local temporary files and clean them after use.

Before sealing, fetch the active certificate and compare its SHA-256 fingerprint
with the verified backup inventory. If missing, export and back up **all** key
Secrets using label `sealedsecrets.bitnami.com/sealed-secrets-key`, including
older keys. Encrypt locally with `age -p`, copy only ciphertext to NAS, decrypt
the NAS copy locally with `age -d`, and verify offline recovery of a harmless
test using `kubeseal --recovery-unseal --recovery-private-key`. Keep the previous
backup until the replacement is verified. Seal using that exact backed-up
public certificate with `kubeseal --cert`; this avoids renewal between checking
coverage and sealing. Adding a credential alone does not require another key
backup. No production key replacement is needed to test recovery.

When adopting an existing Secret, preserve its name, namespace, keys, values,
type and UID. Add `sealedsecrets.bitnami.com/managed=true` to the live Secret
before syncing its SealedSecret, and remove any
`kubectl.kubernetes.io/last-applied-configuration` annotation containing plaintext.
Construct minimal sealing inputs without runtime metadata or owner references;
verify successful decryption and unchanged credentials without displaying values.
Never delete an existing Secret to resolve an ownership conflict.

For existing workload adoption, first confirm a fresh successful VM backup with
PVC disk coverage and privately capture resource identities/specifications and
storage bindings. Pause automation, review the complete diff, and selectively
sync Secrets, storage metadata and workload groups with prune/force/replace off.
Compare storage and application settings after each group; stop on unexpected
differences. Preserve existing images, selectors, mounts and credentials.

## Recovery and everyday operations

### Choose the recovery path

| What was lost | Recovery source and check |
| --- | --- |
| Kubernetes configuration | This Git repository and its history; review the chosen commit before applying it. Keep a separate repository copy for loss of GitHub access. |
| Application databases, settings and local PVC data | Proxmox VM 103 backup with all application-data disks included; Git does not contain this data. |
| NAS media | Separate NAS-media backup; a VM backup covers neither the external share nor its files. Confirm the backup source before claiming full disaster recovery. |
| Sealing private keys | Verified encrypted NAS key backup plus the passphrase in the password manager; retain old keys for old ciphertext. |

A PVC (PersistentVolumeClaim) is an application's request for storage. A PV
(PersistentVolume) is the storage assigned to it. Their binding links the claim
to its data. The eight local PVs use reclaim policy `Delete`, which allows disk
cleanup after claim deletion; the shared NAS PV uses `Retain`.

Use the NAS key-backup directory and password-manager entry in
[Sealed Secrets](#sealed-secrets). Keep an independent copy for NAS loss.

Before recovery, verify backup availability, freshness, application-disk coverage
and key coverage. Confirm independent copies for loss of the host or NAS, and
validate the intended restore method before relying on it.

### Fresh cluster

1. Follow [setup.MD](../../../setup.MD) and the recorded network/node
   prerequisites. Verify NAS access and the chosen data backups. Run
   `kubectl get nodes -o wide`, manually apply `cluster/`, and bootstrap Argo
   using [Manual bootstrap](#manual-bootstrap). **Do not apply the tracked
   ApplicationSet yet: it enables automation.**
2. Create and apply this disabled copy. The assertions stop if the tracked
   template changes; Git is unchanged:

   ```sh
   python3 - <<'PY_DISABLED'
   from pathlib import Path
   text = Path('infrastructure/node-main/ns-argo/application-set.yaml').read_text()
   for field in ('enabled', 'prune', 'selfHeal'):
       old = f'          {field}: true'
       assert text.count(old) == 1, f'Review changed template before recovery: {field}'
       text = text.replace(old, f'          {field}: false')
   Path('/private/tmp/homelab-applicationset-disabled.yaml').write_text(text)
   PY_DISABLED
   kubectl apply --dry-run=server -f /private/tmp/homelab-applicationset-disabled.yaml
   kubectl apply -f /private/tmp/homelab-applicationset-disabled.yaml
   kubectl -n argocd get applications -o custom-columns='NAME:.metadata.name,AUTO:.spec.syncPolicy.automated.enabled,PRUNE:.spec.syncPolicy.automated.prune,SELFHEAL:.spec.syncPolicy.automated.selfHeal,FINALIZERS:.metadata.finalizers,OPERATION:.operation'
   ```

   Wait for exactly `media` and `sealed-secrets`, with false/false/false and no
   finalizers or operation. Do not continue if any generated policy is enabled.
3. Restore verified sealing keys using the NAS README before syncing the
   controller. If it already started, restart it after importing keys so it
   reloads them. Never delete existing keys to resolve conflicts. Manually sync
   only `sealed-secrets`, with prune/force/replace off; wait for its CRD and
   controller using [Sealed Secrets](#sealed-secrets).
4. Restore application data before starting media workloads. Data recovery uses
   Proxmox VM backups. For a fresh cluster, establish and validate a data
   extraction/restore procedure with the owner. Confirm restored directories,
   ownership and PV/PVC mapping before binding claims. Do not silently provision empty volumes
   over missing data. A new cluster has new resource UIDs (unique object IDs);
   confirm its intended storage mapping rather than demanding old UIDs.
5. In the Argo UI, sync only the two media SealedSecrets first; verify their
   `Synced=True` conditions and generated Secret names/keys without displaying
   values. Review the full storage diff, then manually sync storage metadata,
   Homepage, Transmission, the four Arr apps, Seerr and Jellyfin in order.
   Check availability and storage after each group; keep prune/force/replace off.
   Stop on unexpected storage differences. Never delete/recreate storage.
6. Run the [read-only checks](#recovery-checks). Enable automation only after
   keys, application data, storage and smoke checks pass and the owner approves
   resuming. Use [Pausing and resuming](#pausing-and-resuming);
   verify inherited policies and all deletion protections again. Remove the
   temporary disabled copy when finished.

### Ordinary VM restore

An ordinary full-VM restore may resume reconciliation immediately, including
saved operations, possibly against newer Git. It brings back application data,
Kubernetes state and any keys present at backup time. It does not restore NAS
files or keys created after that backup. Compare the chosen Git revision and
key inventory, then check storage, credentials and apps. Do not re-bootstrap
Argo or apply the automated template blindly over restored state.

Disconnecting Git or the NIC does not reliably pause saved operations or
in-cluster reconciliation. If a pause before reconciliation is required, choose
a backup prepared by the guarded procedure below. An unprepared backup does
not provide that guarantee; plan recovery access before starting it.

### Guarded Proxmox backup and restore

These are recovery instructions, **not a validation exercise on a healthy
cluster**. They prepare a recovery point where Argo's two reconciliation
controllers are stopped. Other workloads, including Sealed Secrets, may still
start on VM restore. This procedure does not make application databases
consistent by itself; retain the VM backup's application-data requirements.

Before taking a guarded recovery backup:

1. Follow [Pausing and resuming](#pausing-and-resuming): set the ApplicationSet
   template's `enabled`, `prune` and `selfHeal` to false, apply it manually,
   and verify both Applications inherited false/false/false. Keep
   `allowEmpty: false`, resource preservation and all confirmation annotations.
2. Finish or terminate active syncs in the Argo UI. Check the operation fields
   below, each app's complete diff/resource list for pending prune, and deletion
   timestamps. Completed `Succeeded`/`Failed` phases are not running operations.
   Require no `.operation`, no `Running`/`Terminating` phase, no pending prune or
   deletion, and no deletion approval. Terminating a sync does not undo completed
   changes or cancel Kubernetes deletion. Resolve these before continuing.
3. Record controller replica counts privately, then stop them:

   ```sh
   kubectl -n argocd get deployment argocd-applicationset-controller -o jsonpath='{.spec.replicas}{"\n"}'
   kubectl -n argocd get statefulset argocd-application-controller -o jsonpath='{.spec.replicas}{"\n"}'
   kubectl -n argocd scale deployment/argocd-applicationset-controller --replicas=0
   kubectl -n argocd scale statefulset/argocd-application-controller --replicas=0
   kubectl -n argocd get pods -l app.kubernetes.io/name=argocd-applicationset-controller
   kubectl -n argocd get pods -l app.kubernetes.io/name=argocd-application-controller
   ```

   Both selections must be empty, including terminating Pods; verify desired
   replica counts are zero. Recheck saved operations/deletions after shutdown.
   Take and identify a successful VM 103 backup with PVC disk coverage **while
   both controllers are stopped**. Record archive, time, disk coverage and Git
   revision privately. If the backup fails, it is not a guarded recovery point.
4. To resume the source VM after the backup, use the same restart order below.
   Leave automation disabled until the review is complete.

On restoring that guarded backup:

1. Start cluster checks with `kubectl get nodes -o wide`. Verify both controllers
   still have zero replicas and no Pods **before** reapplying installation
   manifests, which could start them. Review stored Application policies,
   operations/deletions, chosen Git revision and storage identities/bindings.
   For a restored VM, compare against its recorded inventory. Stop on unexpected
   storage differences; restore missing sealing keys using the NAS README.
2. Keep the live ApplicationSet template disabled. If necessary, apply the
   disabled copy from [Fresh cluster](#fresh-cluster) while controllers remain
   stopped. Do not apply the tracked automated template.
3. Check the discovered folder list before starting the ApplicationSet
   controller: it can create/delete Applications even with automatic sync
   disabled. Restore its recorded count first. For the recorded one-replica
   installation the commands are:

   ```sh
   kubectl -n argocd scale deployment/argocd-applicationset-controller --replicas=1
   kubectl -n argocd rollout status deployment/argocd-applicationset-controller --timeout=300s
   kubectl -n argocd get applications -o custom-columns='NAME:.metadata.name,AUTO:.spec.syncPolicy.automated.enabled,PRUNE:.spec.syncPolicy.automated.prune,SELFHEAL:.spec.syncPolicy.automated.selfHeal,FINALIZERS:.metadata.finalizers,OPERATION:.operation'
   ```

   Require false/false/false, no deletion finalizers, operations or deletion
   timestamps; no operation phase may be `Running` or `Terminating`.
4. Only then restore the application-controller's recorded count (one here):

   ```sh
   kubectl -n argocd scale statefulset/argocd-application-controller --replicas=1
   kubectl -n argocd rollout status statefulset/argocd-application-controller --timeout=300s
   ```

   With automation still disabled, review complete diffs and pending resources
   before any manual sync. Run recovery/smoke checks, then resume through the
   reviewed ApplicationSet only when approved. Verify all inherited safeguards.

### Recovery checks

These commands read status without restoring data or changing controllers:

```sh
kubectl -n argocd get applications -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status,AUTO:.spec.syncPolicy.automated.enabled,PRUNE:.spec.syncPolicy.automated.prune,SELFHEAL:.spec.syncPolicy.automated.selfHeal,FINALIZERS:.metadata.finalizers,OPERATION:.operation,PHASE:.status.operationState.phase,DELETING:.metadata.deletionTimestamp,APPROVAL:.metadata.annotations.argocd\.argoproj\.io/deletion-approved'
kubectl -n media get deployments,pods,pvc
kubectl -n media get pvc -o custom-columns='NAME:.metadata.name,UID:.metadata.uid,VOLUME:.spec.volumeName,DELETING:.metadata.deletionTimestamp,PROTECTION:.metadata.annotations.argocd\.argoproj\.io/sync-options'
kubectl get pv -o custom-columns='NAME:.metadata.name,UID:.metadata.uid,CLAIM:.spec.claimRef.name,CLAIM_UID:.spec.claimRef.uid,RECLAIM:.spec.persistentVolumeReclaimPolicy,DELETING:.metadata.deletionTimestamp'
kubectl -n media get sealedsecrets -o custom-columns='NAME:.metadata.name,SYNCED:.status.conditions[0].status,DELETING:.metadata.deletionTimestamp,PROTECTION:.metadata.annotations.argocd\.argoproj\.io/sync-options'
kubectl get crd sealedsecrets.bitnami.com -o jsonpath='{.metadata.annotations.argocd\.argoproj\.io/sync-options}{"\n"}'
curl --noproxy '*' --fail http://homepage.home.arpa/api/healthcheck
```

Before enabling automation, require two Synced/Healthy Applications, eight
available Deployments, Ready Pods, nine Bound PVCs with verified data/bindings,
two successfully decrypted SealedSecrets and confirmation options on all PVCs,
both SealedSecrets and the CRD. No operation, deletion approval or deletion
should be pending. In each Argo app's resource list, inspect **all** pending
prunes, not just PVCs; also inspect deletion timestamps for every managed
resource. Keep namespaces and the NAS PV outside Argo ownership.

Open the seven web apps (Homepage, Jellyfin, Sonarr, Radarr, Prowlarr, Bazarr,
Seerr); expect working pages. Verify Transmission rejects anonymous RPC and
accepts existing credentials for read-only `session-get`. Check Homepage's four
widgets and play existing NAS media in Jellyfin. Do not use downloads, imports,
deletions or library cleanup as tests. The manifests under `tests/` include a
NAS **write** test; do not apply them as a read-only recovery check.

### Normal changes

Argo polls Git; use the UI Refresh button to request comparison sooner. Automatic
sync is enabled: reviewed changes under discovered app paths deploy after
push, manual drift is corrected, and ordinary Git removals are pruned. Review
the complete change before pushing; while automation is paused, review the
complete Argo diff and pending resource list before any manual Sync or resume.
Routine changes use direct pushes; future Renovate changes use reviewed PRs.
Stateful image updates still require release review, an application-native
backup, one pinned digest update, rollout verification and the app's smoke test.
Complete release review and backup before pushing or merging a deploying change.

For a new app, add `ns-<name>/overlays/prod`, declare its namespace under
`cluster/namespace/` and register it in that Kustomization. Manually apply
`cluster/` before its first sync. Seal credentials using a backed-up certificate,
protect PVCs with `Prune=confirm,Delete=confirm` and review/push. Protect any
Argo-managed Namespace the same way; prefer manually owned bootstrap namespaces.
Check its destination mapping, mounts and storage before pushing: new apps
inherit the current enabled policy and may deploy immediately. For staged
onboarding, pause the set first, verify every generated policy is disabled,
then manually sync/review the app before resuming. Directory discovery creates
its Application.
Bootstrap namespaces and the NAS PV remain outside Argo ownership.

### Pausing and resuming

Change `enabled`, `prune` and `selfHeal` together in the manually applied
ApplicationSet template: false/false/false pauses; true/true/true resumes.
Review/commit/push a lasting policy change, then diff, validate and manually
apply that reviewed file:

```sh
kubectl diff -f infrastructure/node-main/ns-argo/application-set.yaml
kubectl apply --dry-run=server -f infrastructure/node-main/ns-argo/application-set.yaml
kubectl apply -f infrastructure/node-main/ns-argo/application-set.yaml
kubectl -n argocd get applications -o custom-columns='NAME:.metadata.name,AUTO:.spec.syncPolicy.automated.enabled,PRUNE:.spec.syncPolicy.automated.prune,SELFHEAL:.spec.syncPolicy.automated.selfHeal,EMPTY:.spec.syncPolicy.automated.allowEmpty,FINALIZERS:.metadata.finalizers'
kubectl -n argocd get applicationset node-main -o jsonpath='{.spec.syncPolicy.preserveResourcesOnDeletion}{"\n"}'
```

For a temporary pause, use the disabled copy under [Fresh cluster](#fresh-cluster),
record that live bootstrap
state differs from Git, and keep the source template unchanged. A Git commit
alone does not apply this bootstrap file. Editing a generated Application's
policy is overwritten by the ApplicationSet controller.

Verify every generated policy after applying. Pausing stops new automatic
syncs; finish or terminate active operations in the UI and verify the operation
and deletion checks above. It still permits manual sync and ApplicationSet
creation/deletion. Before resuming, resolve Git differences, review complete
diffs/pending lists, verify keys/data/storage/smoke checks and preserve
`allowEmpty: false`, preservation=true, absent deletion finalizers and all
confirmation annotations. A stopped work session does not pause Argo.

### Intentional removal

For removal of one service inside media, remove its workload resources first and
retain its PVCs. For a whole discovered Application, save its inventory, verify
it has neither Argo deletion-finalizer variant, then remove the discovered folder.
Wait for its Application to disappear; resources are preserved by the set's
policy. Explicitly remove only the intended workload resources afterward.
Preserve PVCs/namespaces by default. Delete selected PVCs only after the owner
explicitly chooses data erasure. Local PV reclaim policy Delete permits storage
cleanup; NAS PV Retain does not erase NAS files. Never use cascading Application
deletion, delete the shared namespace or run blanket `kubectl delete -k`.
Folder preservation does not disable pruning while the Application still exists.

Argo confirmation annotations do not protect against direct Kubernetes deletion.
Approval is Application-wide: inspect the whole pending list and never commit
`argocd.argoproj.io/deletion-approved`. Deleting a managed SealedSecret deletes
its generated Secret; deleting the CRD removes all SealedSecrets.

For someone else's setup, fork/change the repository URL, hosts, NAS/storage and
node assumptions. Generate their own sealing keys and ciphertext for their own
credentials. This repository's ciphertext cannot initialize another cluster's
credentials. Start with empty data or independently restored data belonging to
that installation.

## Troubleshooting

```sh
kubectl -n argocd get pods
kubectl -n argocd get events --sort-by=.lastTimestamp
kubectl -n argocd logs deployment/argocd-server --tail=50
kubectl -n argocd get configmap argocd-cmd-params-cm -o jsonpath='{.data.server\.insecure}'
curl --noproxy '*' -I http://argocd.home.arpa
```

An API timeout can be laptop VPN routing: verify LAN/VPN connectivity before
changing cluster configuration. For image pulls or Git access from the VM,
inspect UniFi Security logs for blocked registry, GitHub and API traffic.
If changing the HTTP ConfigMap after initial install, reapply then restart only
`deployment/argocd-server` and wait for its rollout.

Sources: [automatic sync](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/),
[Application deletion](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Application-Deletion/),
[confirmation options](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/),
and [Sealed Secrets key recovery](https://github.com/bitnami/sealed-secrets/blob/v0.40.0/README.md#how-can-i-do-a-backup-of-my-sealedsecrets).
