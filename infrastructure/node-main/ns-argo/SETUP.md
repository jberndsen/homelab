# Argo CD operations

Run commands from the repository root using the laptop's configured `kubectl`.
Read [node-main](../system.md) and [network](../../network/system.md) first.
The complete staged rollout and recovery requirements are in
[the agreed plan](../../../argocd-plan.md). Execution 6 finishes this guide;
automatic reconciliation remains enabled on the current cluster. The checkpoint
sections record earlier stages; use the recovery paths below for a rebuild.

## Checkpoint 1: manual bootstrap

**Verified complete on 2026-10-08.** Argo v3.5.4 is running with all seven Pods
Ready and its three CRDs Established. HTTP UI, health and version checks passed.
The owner completed admin password rotation and replacement-password login;
the initial-password Secret is absent. Eight media Deployments remain available
and nine PVCs Bound, with Deployment specs/images and PVC/PV identities,
bindings and PV specs matching the private baseline. At checkpoint 1, no
Applications or ApplicationSets existed. Checkpoint 2's resulting state is below.

The `ns-argo/kustomization.yaml` below installs Argo CD only. The ApplicationSet
is applied separately in checkpoint 2. Media remains manually managed until
the deliberate adoption in checkpoint 3.

The pinned standard non-HA installation is **v3.5.4**, reviewed on 2026-10-08.
[Its release](https://github.com/argoproj/argo-cd/releases/tag/v3.5.4) fixes
critical vulnerabilities affecting the originally planned v3.5.3. The 3.5 line
is [tested with Kubernetes 1.36](https://argo-cd.readthedocs.io/en/stable/operator-manual/tested-kubernetes-versions/).
Upstream NetworkPolicies are retained. Upstream workloads have no resource
requests/limits; check actual usage after bootstrap.

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

## Checkpoint 2: controller and independent key recovery

**Verified complete on 2026-10-08. The following describes checkpoint 2;
checkpoint 3 has since resolved the media render error and adopted media.**
The owner confirmed the passphrase is saved in their password manager as
`Homelab Sealed Secrets key backup`. `sealed-secrets` is Synced/Healthy, its
sync Succeeded, CRD Established and controller Ready. Exactly `media` and
`sealed-secrets` exist, with automatic sync disabled and no deletion finalizers. Media has no sync
history; its ignored-Secret render error is expected until checkpoint 3.
All eight media Deployments and nine PVCs remain healthy, with Deployment/PVC
specs and UIDs and PV specs/UIDs/bindings matching the private pre-Argo baseline.

The actual NAS backup `sealed-secrets-keys-2026-10-08T184312Z.yaml.age` was
decrypted and matched the full key export. Controller validation and offline
recovery of a harmless strict-scope test passed. The active certificate and
every current key are covered; SHA-256 certificate fingerprint:
`b77a5acd6d48fc4bf2d7620d1da9a443cd4c205789efbf0de1726c3e14e6b1a4`.
No test objects were applied or production keys replaced. Protected temporary
plaintext files were removed. The adjacent NAS `README.md` contains the dated
inventory and exact backup, verification, restoration and cleanup commands.

Sealed Secrets is pinned to **v0.40.0**, reviewed on 2026-10-08 with its
[release notes](https://github.com/bitnami/sealed-secrets/releases/tag/v0.40.0)
and published advisories. This version includes fixes for
[the decryption oracle](https://github.com/bitnami/sealed-secrets/security/advisories/GHSA-qj4p-m373-p2wg)
and [scope widening](https://github.com/bitnami/sealed-secrets/security/advisories/GHSA-465p-v42x-3fmj).
The laptop uses matching `kubeseal v0.40.0` and `age v1.3.2`.

The manually applied `application-set.yaml` discovers
`infrastructure/node-main/ns-*/overlays/prod`. For example, segment 2 of
`infrastructure/node-main/ns-media/overlays/prod` is `ns-media`; trimming
`ns-` produces the Application name and namespace `media`. Sealed Secrets is
explicitly mapped to `kube-system`. Go templates fail on missing keys; Argo's
own installation is excluded and remains manually managed.

An ApplicationSet creates Applications; an Application renders Git and applies
resources only when synced. A deletion finalizer is a marker that makes
Kubernetes wait for a controller to clean up resources before deleting an object. Both generated Applications have automatic sync,
pruning and self-healing disabled during adoption. Removing a discovered folder
preserves deployed resources: the set uses `preserveResourcesOnDeletion: true`
and generated Applications have no Argo deletion finalizer. The Sealed Secrets
CRD has `Prune=confirm,Delete=confirm`; deleting it would delete its custom
resources. No `CreateNamespace=true` is used: declare future app namespaces in
`cluster/namespace/`, register them in its Kustomization, and manually apply
`cluster/` before their first sync.

These were the checkpoint-2 manual bootstrap commands. The current tracked
template enables automation: on a fresh/recovered cluster, first create the
disabled copy under [Fresh cluster](#fresh-cluster), then use that copy here.

```sh
kubectl apply --dry-run=server -f /private/tmp/homelab-applicationset-disabled.yaml
kubectl apply -f /private/tmp/homelab-applicationset-disabled.yaml
kubectl -n argocd get applications,applicationsets
kubectl -n argocd get applications -o custom-columns='NAME:.metadata.name,NAMESPACE:.spec.destination.namespace,AUTO:.spec.syncPolicy.automated.enabled,FINALIZERS:.metadata.finalizers'
```

Expect exactly `media` → `media` and `sealed-secrets` → `kube-system`, both
with `AUTO=false` and no finalizers. The old media render error was resolved at
checkpoint 3; a current checkout renders without ignored files. On recovery,
**do not sync media yet**: first restore keys, application data and storage.

In the Argo UI, open **sealed-secrets → Sync**, leave prune and force/replace
off, review the resource list, and sync only this Application. The equivalent
Kubernetes operation below requires the reviewed commit to be on GitHub; first
verify the Application has no active operation. `apply` selects ordinary apply,
and is not the destructive `replace` strategy.

```sh
git rev-parse HEAD
kubectl -n argocd get application sealed-secrets -o jsonpath='{.operation}{"\n"}'
# Replace REVIEWED_COMMIT with that pushed commit SHA.
kubectl -n argocd patch application sealed-secrets --type=merge -p \
  '{"operation":{"initiatedBy":{"username":"checkpoint-2"},"sync":{"revision":"REVIEWED_COMMIT","prune":false,"syncStrategy":{"apply":{}}}}}'
kubectl wait --for=condition=Established --timeout=60s crd/sealedsecrets.bitnami.com
kubectl -n kube-system rollout status deployment/sealed-secrets-controller --timeout=300s
kubectl -n argocd get application sealed-secrets
```

Separate Applications do not guarantee startup order. Wait for the controller
before creating any SealedSecrets. The upstream default renews sealing keys
every 30 days; old keys remain necessary for old ciphertext.

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

## Checkpoint 3: existing media adopted

**Verified complete on 2026-10-08.** Both Applications were Synced/Healthy at
the checkpoint-3 pause, with automatic sync, pruning and self-healing disabled.
Checkpoint 4 has since enabled them; its current policy and checks are below.
The owner confirmed Homepage's four widgets and Jellyfin playback of existing
NAS media work. No application restart was needed.

Argo adopted the existing media resources through manual sync. Ownership adds
tracking metadata; it does not move data. The original PVC/PV identities,
bindings, workload specifications, image digests, settings and mounts are
unchanged. Secrets `homepage-widgets` and `transmission-rpc` have the same names,
values, types and UIDs, now managed by their strict-scope SealedSecrets.

Adoption used commit `450da82` and these gates:

1. Compare live resources with the private pre-Argo baseline. Use the owner's
   same-day VM 103 backup/PVC disk-coverage confirmation. Recheck freshness after
   substantial time or storage changes; NAS media is external to that backup.
2. Check the actual NAS ciphertext checksum and public key inventory against
   every current key and the active certificate. Seal with that exact checked
   certificate. Construct minimal Secret inputs in memory, without runtime
   metadata, tracking or historical last-applied annotations.
3. Replace the two ignored plaintext references with tracked encrypted manifests.
   Protect every PVC with `Prune=confirm,Delete=confirm`. Verify CRD protection;
   protect the SealedSecrets too, since their deletion deletes generated Secrets.
4. Render from a clean committed checkout; review Kubernetes and Argo diffs.
   Only ciphertext additions, protection and tracking metadata were expected.
   Stop on unexpected storage differences. Never delete/recreate storage or
   use force/replace to resolve them.
5. Annotate the existing Secrets `sealedsecrets.bitnami.com/managed=true` and
   remove their historical last-applied annotations before syncing SealedSecrets.
   Sync only the two SealedSecrets and verify controller ownership, unchanged
   values/type/UID and successful decryption without printing credential data.
6. Selectively sync storage metadata, Homepage, Transmission, the four Arr apps,
   Seerr, then Jellyfin. Review the diff before every group and leave prune,
   force and replace off. Compare baseline identities/specs after every group.

A SealedSecret is encrypted configuration; the controller creates or adopts the
ordinary Secret that applications read. Strict scope binds encryption to the
exact name and namespace. Do not rename one to move credentials. The old ignored
Secret files are no longer needed to render from Git; local reference files must
remain untracked. Application credentials and sealing private keys never belong
in tool output, Git, annotations or ConfigMaps.

Useful checks from the repository root:

```sh
kubectl get nodes -o wide
kubectl kustomize infrastructure/node-main/ns-media/overlays/prod > /private/tmp/media-rendered.yaml
kubectl -n argocd get applications
kubectl -n media get deployments,pods,pvc
kubectl -n media get sealedsecrets
kubectl -n media get pvc -o custom-columns='NAME:.metadata.name,UID:.metadata.uid,VOLUME:.spec.volumeName,PROTECTION:.metadata.annotations.argocd\.argoproj\.io/sync-options'
kubectl get crd sealedsecrets.bitnami.com -o jsonpath='{.metadata.annotations.argocd\.argoproj\.io/sync-options}{"\n"}'
kubectl -n argocd get applications -o custom-columns='NAME:.metadata.name,AUTO:.spec.syncPolicy.automated.enabled,PRUNE:.spec.syncPolicy.automated.prune,SELFHEAL:.spec.syncPolicy.automated.selfHeal,FINALIZERS:.metadata.finalizers,OPERATION:.operation'
curl --noproxy '*' --fail http://homepage.home.arpa/api/healthcheck
```

Expect two Synced/Healthy Applications, eight available Deployments and original
Ready Pods, nine Bound/protected PVCs and two successfully synced SealedSecrets.
At checkpoint 3, both policies showed false/false/false. After checkpoint 4,
expect true/true/true, with no finalizers or operation.
The render contains ciphertext only. The private baseline is
`plans/runtime/argocd-baseline-2026-10-08.json`; it is comparison evidence, not a
backup. Compare identities and bindings against it before making storage changes.

Smoke checks passed: seven HTTP web endpoints, Transmission rejects anonymous
RPC and accepts original credentials for read-only `session-get`, and Homepage
reaches Jellyfin/Sonarr/Radarr/Prowlarr APIs through service DNS using existing
keys. Ten selected application settings files and both ConfigMaps were unchanged.
Verify widgets and play existing NAS media in Jellyfin after future changes.
Do not test by triggering downloads, imports, deletions or library cleanup.

## Checkpoint 4: automatic reconciliation enabled

**Verified complete on 2026-10-08.** Automatic sync
deploys reviewed Git changes; self-healing corrects manual changes in the
cluster; pruning removes ordinary resources removed from Git. Both Applications
inherit these settings from the manually applied ApplicationSet, at policy
commit `529fc09`:

```yaml
automated:
  enabled: true
  prune: true
  selfHeal: true
  allowEmpty: false
```

Before enabling/resuming, review the complete Argo diff and pending resource
list. Require both Applications Healthy, no operations or deletions pending,
and storage identities/specs/bindings matching the private baseline. Stop on
unexpected storage changes. Review/commit/push the template, then apply it
manually because Argo does not manage its own bootstrap files:

```sh
kubectl diff -f infrastructure/node-main/ns-argo/application-set.yaml
kubectl apply --dry-run=server -f infrastructure/node-main/ns-argo/application-set.yaml
kubectl apply -f infrastructure/node-main/ns-argo/application-set.yaml
kubectl -n argocd get applications -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status,AUTO:.spec.syncPolicy.automated.enabled,PRUNE:.spec.syncPolicy.automated.prune,SELFHEAL:.spec.syncPolicy.automated.selfHeal,EMPTY:.spec.syncPolicy.automated.allowEmpty,FINALIZERS:.metadata.finalizers,OPERATION:.operation'
kubectl -n argocd get applicationset node-main -o jsonpath='{.spec.syncPolicy.preserveResourcesOnDeletion}{"\n"}'
kubectl -n media get pvc -o custom-columns='NAME:.metadata.name,PHASE:.status.phase,PROTECTION:.metadata.annotations.argocd\.argoproj\.io/sync-options'
```

Expect exactly two Synced/Healthy Applications, true/true/true/false policies,
no finalizers or operation, preservation=true, and nine Bound PVCs with
`Prune=confirm,Delete=confirm`. Both SealedSecrets and the controller CRD retain
the same confirmation options. Neither Argo deletion-finalizer variant is
present. `allowEmpty: false` blocks an entirely empty desired application; it
does not protect against partial removal. Confirmation annotations still apply
when pruning is enabled. Never add force/replace or a deletion approval to Git.

The unused `argocd-reconciliation-check` ConfigMap held only `message: from-git`:

| Behavior | Verified evidence (UTC, 2026-10-08) |
| --- | --- |
| Automatic Git deployment | Addition `e51d741`, automatic sync Succeeded at 19:35:29 |
| Self-healing | Live `manual-test` reverted to `from-git` at 19:38:02, same ConfigMap UID |
| Pruning | Removal `c58fa7f`, automatic sync pruned exactly that ConfigMap at 19:39:20 |

Comparison refreshes were used, without requesting any manual Sync. No storage
or credentials were test targets. The test manifest and object are gone; media
manifests are identical to their checkpoint-3 state. Storage/workload specs,
UIDs and bindings, credentials, existing ConfigMaps and ten settings files
remain unchanged. The same eight media Pods are Ready with zero restarts.
Seven web endpoints, Transmission authenticated read-only RPC and Homepage's
four backend APIs passed again. The owner's widget and playback confirmation
was at checkpoint 3; checkpoint 4 added the read-only checks above.

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

The key backup is at **smb://192.168.1.32/backups/NUC/kubernetes**, mounted on the
laptop at **/Volumes/backups/NUC/kubernetes**. Its **README.md** has exact export,
`age -p` encryption, `age -d` decryption, restore and temporary-file cleanup
commands. The passphrase is in the password-manager entry **Homelab Sealed
Secrets key backup**. Loss of the NAS can also lose this key backup; confirm
an independent copy before relying on it for that failure.

Evidence reviewed on 2026-10-08: independent key recovery passed at checkpoint
2; this audit rechecked the actual encrypted file's checksum, all live key
certificates and active-certificate coverage without decrypting or restoring.
The owner confirmed a successful VM backup with PVC disk coverage. Their
Proxmox screenshot lists `vzdump-qemu-103-2026_10_08-20_53_04.vma.zst` on storage
`local`, with displayed date `2026-10-08 20:53:04` (display timezone not verified).
It does not establish a restore test, off-host copy or guarded controller state.
NAS-media backup/recovery evidence and an independent copy of the key backup
have not been supplied. Recheck backup freshness before future recovery work.

### Fresh cluster

1. Follow [setup.MD](../../../setup.MD) and the recorded network/node
   prerequisites. Verify NAS access and the chosen data backups. Run
   `kubectl get nodes -o wide`, manually apply `cluster/`, and bootstrap Argo
   using checkpoint 1 above. **Do not apply the tracked ApplicationSet yet:
   it enables automation.** Separate Applications do not enforce startup order.
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
   controller using checkpoint 2's readiness commands.
4. Restore application data before starting media workloads. A full VM restore
   is the recorded recovery source; no file-level fresh-cluster data restore has
   been tested here. Confirm restored directories, ownership and PV/PVC mapping
   with the owner before binding claims. Do not silently provision empty volumes
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
   resuming. Use checkpoint 4's reviewed tracked template and manual apply;
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
   timestamps. Historical `Succeeded`/`Failed` phases are not running operations.
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

### Normal changes and intentional removal

Argo polls Git; use the UI Refresh button to request comparison sooner. Automatic
sync is now enabled: reviewed changes under discovered app paths deploy after
push, manual drift is corrected, and ordinary Git removals are pruned. Review
the complete change before pushing; while automation is paused, review the
complete Argo diff and pending resource list before any manual Sync or resume.
Routine changes use direct pushes; future Renovate changes use reviewed PRs.
Stateful image updates still require release review, an application-native
backup, one pinned digest update, rollout verification and the app's smoke test.

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
Review/commit/push a lasting policy change, then run checkpoint 4's diff,
dry-run and manual apply commands with that reviewed file. For a temporary
pause, use the disabled copy under Fresh cluster, record that live bootstrap
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

## Troubleshooting this stage

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

## Documentation audit — Execution 6

Reviewed the recovery paths, everyday changes, pauses, onboarding and removal
against the plan and live safeguards. Local links, shell/Python syntax, clean
tracked-checkout renders and the disabled-copy transformation were checked.
Mutation/restore commands were reviewed and checked with safe dry-runs where
appropriate; no restore, key replacement, controller pause or demonstration
was performed. See [the progress record](../../../argocd-plan.md#implementation-progress--2026-10-08)
for completed checks and limits.

Sources for the operating rules: [automatic sync](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/),
[Application deletion](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Application-Deletion/),
[confirmation options](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/),
and [Sealed Secrets key recovery](https://github.com/bitnami/sealed-secrets/blob/v0.40.0/README.md#how-can-i-do-a-backup-of-my-sealedsecrets).
