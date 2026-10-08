# Argo CD operations

Run commands from the repository root using the laptop's configured `kubectl`.
Read [node-main](../system.md) and [network](../../network/system.md) first.
The complete staged rollout and recovery requirements are in
[the agreed plan](../../../argocd-plan.md).

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
resources only when synced. Both generated Applications have automatic sync,
pruning and self-healing disabled during adoption. Removing a discovered folder
preserves deployed resources: the set uses `preserveResourcesOnDeletion: true`
and generated Applications have no Argo deletion finalizer. The Sealed Secrets
CRD has `Prune=confirm,Delete=confirm`; deleting it would delete its custom
resources. No `CreateNamespace=true` is used: declare future app namespaces in
`cluster/namespace/`, register them in its Kustomization, and manually apply
`cluster/` before their first sync.

After reviewing, committing and pushing these files:

```sh
kubectl apply --dry-run=server -f infrastructure/node-main/ns-argo/application-set.yaml
kubectl apply -f infrastructure/node-main/ns-argo/application-set.yaml
kubectl -n argocd get applications,applicationsets
kubectl -n argocd get applications -o custom-columns='NAME:.metadata.name,NAMESPACE:.spec.destination.namespace,AUTO:.spec.syncPolicy.automated.enabled,FINALIZERS:.metadata.finalizers'
```

Expect exactly `media` → `media` and `sealed-secrets` → `kube-system`, both
with `AUTO=false` and no finalizers. Media's render error is expected until
checkpoint 3 replaces its ignored Secret files. **Do not sync media yet.**

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

**Verified complete on 2026-10-08; paused before checkpoint 4.** Both Applications
are Synced/Healthy. Automatic sync, pruning and self-healing remain disabled.
The owner confirmed Homepage's four widgets and Jellyfin playback of existing
NAS media work. No application restart was needed.

Argo now owns the existing media resources through manual sync. Ownership adds
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
Both policies should show false/false/false, with no finalizers or operation.
The render contains ciphertext only. The private baseline is
`plans/runtime/argocd-baseline-2026-10-08.json`; it is comparison evidence, not a
backup. Compare identities and bindings against it before making storage changes.

Smoke checks passed: seven HTTP web endpoints, Transmission rejects anonymous
RPC and accepts original credentials for read-only `session-get`, and Homepage
reaches Jellyfin/Sonarr/Radarr/Prowlarr APIs through service DNS using existing
keys. Ten selected application settings files and both ConfigMaps were unchanged.
Verify widgets and play existing NAS media in Jellyfin after future changes.
Do not test by triggering downloads, imports, deletions or library cleanup.

## Recovery and everyday operations

### Fresh cluster

Follow [setup.MD](../../../setup.MD) and the recorded network/node prerequisites.
Manually apply `cluster/` and bootstrap Argo, then apply the ApplicationSet with
automation disabled. Restore the verified sealing keys from the NAS recovery
README before syncing the controller; if it already started, restart it after
importing keys so it reloads them. Do not delete existing keys to resolve conflicts.
Restore application data before starting workloads. Sync the controller and wait
for readiness, then sync the two media SealedSecrets and verify decryption.
Review storage and manually sync the media groups. Enable automation only after
all recovery and smoke checks and explicit continuation to checkpoint 4.

Git restores configuration; Proxmox backups restore application data. NAS media,
Git and sealing keys have separate recovery needs. The encrypted key backup is
at `smb://192.168.1.32/backups/NUC/kubernetes`, mounted on the laptop at
`/Volumes/backups/NUC/kubernetes`. Its README contains exact encryption,
verification, restore and cleanup commands. The passphrase is in the owner's
password manager under `Homelab Sealed Secrets key backup`.

### Guarded Proxmox backup and restore

A normal full-VM restore starts Argo again, potentially against newer Git.
Disconnecting Git or the NIC does not reliably pause saved operations or
in-cluster reconciliation. A backup without the following preparation does not
guarantee a pause before reconciliation.

Before taking a guarded recovery backup:

1. Set `automated.enabled`, `prune` and `selfHeal` to false in the ApplicationSet
   file and manually reapply it. Verify both generated policies using the command
   above. Changing a generated Application directly will be overwritten.
2. Wait for existing operations to finish, or terminate them in the UI. Check
   `.operation`, `.status.operationState`, all pending prune/delete lists and
   resource deletion timestamps. Proceed only when no deletion is pending.
3. Record the two controller replica counts privately, then stop the controllers:

   ```sh
   kubectl -n argocd get deployment argocd-applicationset-controller -o jsonpath='{.spec.replicas}{"\n"}'
   kubectl -n argocd get statefulset argocd-application-controller -o jsonpath='{.spec.replicas}{"\n"}'
   kubectl -n argocd scale deployment/argocd-applicationset-controller --replicas=0
   kubectl -n argocd scale statefulset/argocd-application-controller --replicas=0
   kubectl -n argocd get pods
   ```

   Wait until both controllers' Pods have disappeared. Take and identify a
   successful VM 103 backup with PVC disk coverage before restoring their
   recorded replica counts. This uses the existing backup facility.

On restoring that guarded backup, verify both controllers are still stopped
before reapplying installation manifests. Review storage identities/bindings,
Git revision, saved operations and pending deletion. Restore missing sealing
keys. Start the ApplicationSet controller with the template's automation still
disabled, verify generated policies, then start the application-controller using
recorded replica counts. Review diffs and smoke tests before any manual sync or
future re-enabling of automation.

### Normal changes and intentional removal

Argo polls Git; use the UI Refresh button to request comparison sooner. While
paused at checkpoint 3, pushes update desired configuration but deployment still
requires manual Sync. Review the complete diff and pending resource list first.
Routine changes use direct pushes; future Renovate changes use reviewed PRs.
Stateful image updates still require release review, an application-native
backup, one pinned digest update, rollout verification and the app's smoke test.

For a new app, add `ns-<name>/overlays/prod`, declare its namespace under
`cluster/namespace/` and register it in that Kustomization. Manually apply
`cluster/` before its first sync. Seal credentials using a backed-up certificate,
protect PVCs and review/push. Directory discovery creates its Application.
Bootstrap namespaces and the NAS PV remain outside Argo ownership.

Pause/resume policies through the manually applied ApplicationSet file. A Git
commit alone does not apply that bootstrap file. Verify every generated policy
and terminate active operations when pausing. Disabling automation stops new
automatic syncs; it does not block manual sync or ApplicationSet creation/deletion.
Resolve Git differences before resuming. Checkpoint 4 has not been authorized here.

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
