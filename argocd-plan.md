# Argo CD plan — TODO 1–3

Implement in small steps, explaining each concept before its commands. Pause at
the four checkpoints below for the owner to inspect and continue. This document
is a plan, not authorization to start deployment during the planning session.
This file captures all agreed design decisions. The implementation/teaching agent
needs this plan and the existing repository; no prior conversation, sample PDF,
or screenshots are required. Read the repository records and verify live facts
as instructed below.

## Agreed outcome

- Public source: `https://github.com/jberndsen/homelab.git`, branch `main`.
  Routine changes use direct pushes; future Renovate changes use reviewed PRs.
  Argo reads anonymously over HTTPS and polls Git; no webhook is needed.
- Manually bootstrap a pinned, standard non-HA Argo CD installation through
  Kustomize under `infrastructure/node-main/ns-argo`. Argo never manages itself.
- Declare namespace `argocd` alongside `media` in `cluster/namespace/`.
  Continue manually applying `cluster/`, including the existing NAS PV.
- Expose `http://argocd.home.arpa` through the existing Traefik Ingress pattern,
  with `server.insecure: "true"`. Use built-in `admin`; change its initial
  password and keep the replacement in the password manager. HTTP leaves admin
  credentials/session traffic unencrypted: restrict access to the trusted LAN.
- One Git directory ApplicationSet discovers
  `infrastructure/node-main/ns-*/overlays/prod`. Generate `media` and
  `sealed-secrets` Applications. Strip `ns-` from the folder name for app and
  destination namespace; explicitly map `sealed-secrets` to `kube-system`.
  Keep `ns-argo` outside the discovery layout and explicitly exclude it.
- Begin with manual sync. After adoption, enable automatic sync, pruning, and
  self-healing. Git changes then deploy; manual cluster drift is corrected.
- Set ApplicationSet `spec.syncPolicy.preserveResourcesOnDeletion: true` and
  omit cascading-deletion finalizers from generated Applications. Removing a
  discovered folder removes its Application but leaves deployed resources.
- Require `Prune=confirm,Delete=confirm` on managed PVCs and any managed Namespace
  manifests. Bootstrap namespaces remain outside Argo ownership. Keep local PV
  reclaim policy `Delete`; keep the NAS PV on `Retain`.
  These gates only govern Argo deletion; direct Kubernetes/namespace deletion
  bypasses them. Approval is Application-wide: review the whole pending list
  and never commit `argocd.argoproj.io/deletion-approved`.

## Boundaries and current facts

On 2026-10-06, GitHub `main` matched local commit `6a0a2f3`; the cluster node
`vm-ubuntu-k3s` was Ready at `192.168.30.103`, running K3s `v1.36.2+k3s1`.
All eight media Deployments were available; all nine PVCs were Bound. Eight
local-path PVs use `Delete`; the shared `media-nfs-pv` uses `Retain`. The VM has
8 vCPUs and about 16 GiB RAM; observed use was 2.6 GiB with no node pressure.
Traefik 3.7.4 already publishes `192.168.30.103` in media Ingress status.

Brief application restarts are acceptable. Preserve PVC/PV identities, bindings,
application settings, credentials, NAS paths and existing image digests. Never
delete/recreate storage or use force/replace sync to resolve adoption differences.
Stop on an unexpected storage diff. These controls reduce risk; they do not
guarantee zero data loss.

Before adoption, require a fresh successful Proxmox backup of VM 103 and owner
confirmation that its included disks contain all application PVC data. Full data
recovery comes from Proxmox backups. NAS media is external to that VM backup.
Creating data-backup systems, Renovate, observability, TLS/SSO, and other TODOs
are outside scope. Retain the existing image-update rule: release review,
application-native backup, one pinned digest change, rollout and smoke test;
complete review/backup before pushing or merging a deploying change.

## Intended files

All paths below are relative to `infrastructure/node-main/` unless stated otherwise.

| Path | Purpose |
| --- | --- |
| `cluster/namespace/argocd-namespace.yaml` | Argo namespace; register in existing Kustomization |
| `ns-argo/kustomization.yaml` | Pinned upstream install, HTTP patch and Ingress |
| `ns-argo/argocd-cmd-params-cm.yaml` | Explicit server HTTP setting |
| `ns-argo/ingress.yaml` | `argocd.home.arpa` → Argo server HTTP service |
| `ns-argo/application-set.yaml` | Applied separately after CRDs are established |
| `ns-argo/SETUP.md` | Bootstrap, recovery and everyday operations |
| `ns-sealed-secrets/base/kustomization.yaml` | Pinned upstream controller manifest |
| `ns-sealed-secrets/overlays/prod/kustomization.yaml` | Discoverable production entry point, namespace `kube-system` |
| Existing media bases/overlay | SealedSecret manifests and PVC protection annotations |
| Root `README.md`, existing `setup.MD`, relevant `system.md` | Entry point, recovery links and confirmed facts |
| NAS `backups/NUC/kubernetes/README.md` | Key-backup encryption and restoration instructions |

Prefer these few YAML files and plain commands over a new framework. Use Argo's
`default` project for this single-owner cluster. Pin reviewed
upstream release versions instead of `stable`/`latest`. Research on 2026-10-06
identified Argo CD `v3.5.3` (the 3.5 line is tested with Kubernetes 1.36) and
Sealed Secrets plus `kubeseal` `v0.40.0`; recheck release notes and advisories
at implementation. Standard non-HA fits one node; upstream HA needs three.
Use anonymous HTTPS for the public repository, platform-appropriate client
downloads, exact ConfigMap patch targets and the HTTP settings specified above.

## Execution

### 1. Establish the baseline and repository readiness

1. Read `AGENTS.md` and all `infrastructure/**/system.md`; start cluster work with
   `kubectl get nodes -o wide`. Check Git status and preserve unrelated changes.
2. Verify remote `main`, audit tracked files/history for actual credentials
   without printing values, and keep temporary reference files and private exports out
   of commits. Respect documented fake credentials in node-secondary.
   If real historical credentials are found, stop and arrange rotation with
   the owner; do not silently rewrite public history.
3. Record a private baseline of Deployment images, PVC/PV UIDs and bindings,
   volume paths, and application availability. Confirm the Proxmox backup gate
   before adopting existing resources. Do not export Secret values into logs.
4. Note the clean-clone blockers: Homepage references ignored
   `homepage-secret.yaml`; Transmission references ignored `00-rpc-secret.yaml`.
   Their live Secrets are `homepage-widgets` and `transmission-rpc` in `media`.

### 2. Bootstrap Argo — checkpoint 1

1. Add the namespace manifest; render and diff `cluster/` before applying it.
2. Build and inspect the pinned Argo Kustomization. Apply with
   `kubectl apply --server-side -k infrastructure/node-main/ns-argo`.
   Set Kustomize `namespace: argocd` and preserve upstream NetworkPolicies.
   Keep the ApplicationSet out of this Kustomization to avoid the initial CRD race.
3. Wait for Argo CRDs to be Established and all its controllers/services to be
   ready. Verify HTTP Ingress, login, password change and initial-password Secret
   removal. Route Ingress to `argocd-server` port 80 with class `traefik`.
   If using the CLI, use `argocd login argocd.home.arpa:80 --plaintext --grpc-web`
   and an interactive password prompt (`--insecure` alone does not disable TLS).
   Check CPU/RAM and pod readiness after installation; upstream bundles do not
   specify resource requests/limits. Do not put passwords into history or Git.
4. Write bootstrap commands and explain Namespace, Kustomize, CRD and controller
   in `SETUP.md`. Update system records. **Pause: owner can use the Argo UI.**

### 3. Install Sealed Secrets and secure recovery — checkpoint 2

1. Add its pinned Kustomize base/overlay. Add the ApplicationSet using Go
   templates with `missingkey=error`: name expression
   `{{ index .path.segments 2 | trimPrefix "ns-" }}`; source is `{{ .path.path }}`.
   Explicitly map the controller destination to `kube-system`. Create the
   ApplicationSet/Applications in `argocd`, targeting
   `https://kubernetes.default.svc`. Omit `CreateNamespace=true`; document
   creating future app namespaces under `cluster/` before their first sync.
2. Set generated Applications' `automated.enabled: false` during bootstrap and
   adoption. Commit/push reviewed files, then manually apply the ApplicationSet
   after CRDs exist. Verify exactly `media` and `sealed-secrets` are generated.
   Media may show a render error until its ignored Secret references are replaced;
   do not sync it yet. Manually sync only Sealed Secrets; wait for its CRD and
   controller readiness. Separate Applications do not guarantee startup order.
3. Install compatible `kubeseal` and `age` on the laptop. Export all controller
   key Secrets selected by `sealedsecrets.bitnami.com/sealed-secrets-key` from
   `kube-system`, and encrypt with `age` passphrase mode. Use protected local
   temporary files, check each command succeeds, and never put plaintext on NAS
   or in Git. Store a dated `.yaml.age` backup at
   `smb://192.168.1.32/backups/NUC/kubernetes`, mounted on the laptop at
   `/Volumes/backups/NUC/kubernetes`. Owner stores the passphrase in the password
   manager; retain the previous backup until the replacement is verified.
4. Create the adjacent NAS `README.md` with exact export, `age -p` encryption,
   `age -d` decryption, restore and temporary-file cleanup commands. Record backup
   date and key certificate fingerprints, never private values. Keep default
   30-day key renewal: before sealing, compare the active certificate against
   backup coverage and refresh the full key backup if a new key is present.
   Seal using that exact backed-up public certificate (`kubeseal --cert`) to
   avoid renewal between the check and sealing. Adding a secret alone does not
   require another key backup.
5. Seal a harmless test value. Decrypt the NAS backup locally and verify offline
   `kubeseal --recovery-unseal` recovers that value; do not replace production
   keys to test restoration. Remove temporary plaintext. Explain that older
   keys remain needed for older sealed manifests. **Pause: controller and
   independent encrypted key backup verified.**

### 4. Adopt media without replacing data — checkpoint 3

1. Seal the two existing live Secrets with strict name/namespace scope, preserving
   their names, keys and values. Construct minimal inputs; remove
   `kubectl.kubernetes.io/last-applied-configuration` (can contain plaintext),
   tracking annotations, ownerReferences, UIDs/resourceVersions and managedFields.
   Annotate each existing Secret `sealedsecrets.bitnami.com/managed: "true"`
   before applying its SealedSecret; verify adoption in place. Never delete it
   to resolve an ownership conflict. Check ciphertext files and template metadata
   for plaintext before committing; deleting a managed SealedSecret also deletes
   its generated Secret.
2. Replace ignored Secret references with tracked SealedSecret files. Add
   `Prune=confirm,Delete=confirm` to every media PVC. Protect the Sealed Secrets
   CRD likewise, since deleting it would remove its custom resources.
3. From a clean checkout containing no ignored files, verify
   `kubectl kustomize infrastructure/node-main/ns-media/overlays/prod` succeeds.
   Push reviewed changes; inspect Argo's diff against the live cluster.
4. With automatic sync and pruning off, selectively sync SealedSecrets first;
   verify successful decryption and matching values without displaying them.
   Adopt storage metadata and then one workload group at a time, starting with
   Homepage. Review diffs before each sync; keep images, selectors and mounts.
5. Verify all eight Deployments, nine PVCs, original PVC/PV UIDs and bindings,
   existing settings, Transmission authentication, Homepage widgets, service
   connectivity and Jellyfin playback of existing NAS media. Avoid download,
   import, delete or library-cleanup jobs as tests. Resolve unexpected drift
   before enabling automation. Update records. **Pause: media is adopted and
   fully working, with unchanged storage identities.**

### 5. Enable reconciliation — checkpoint 4

1. Set the ApplicationSet template to `automated.enabled: true`, `prune: true`,
   `selfHeal: true`, retaining `allowEmpty: false` and resource confirmations.
   Commit/push and manually reapply the ApplicationSet: Argo does not manage
   its bootstrap files. Verify generated Applications inherit the policy, all
   nine live PVCs have both confirmation options, and Applications lack both
   Argo resource-deletion finalizer variants. `allowEmpty: false` does not protect
   against partial removal of resources.
2. Demonstrate Git sync, self-healing and ordinary pruning using an otherwise
   unused disposable ConfigMap: add via Git, modify a declared value live, watch
   it revert, then remove via Git. Never use storage or real credentials for tests.
3. Verify both Applications are Synced/Healthy and media smoke checks still pass.
   Update records. **Pause: owner has seen all three reconciliation behaviors.**

### 6. Finish the operating and recovery documentation

Keep instructions short, executable from the repo root, and explain expected
results. `ns-argo/SETUP.md` must cover:

- **Clean cluster:** prerequisites from system records → manual `cluster/` →
  manual Argo bootstrap → ApplicationSet applied with automatic sync disabled →
  restore sealing keys before starting Sealed Secrets (or restart its controller
  after importing keys) → sync controller → verify unsealing → sync media →
  enable automation. Never apply the final automated template before these gates.
- **Proxmox restore:** an ordinary full-VM restore resumes Argo automatically,
  possibly against newer Git. Blocking Git or disconnecting the NIC is not a
  reliable pause: saved operations and in-cluster reconciliation can still run.
  Document a guarded recovery point using the existing backup facility: disable
  automation through the ApplicationSet, verify inherited policies, finish or
  terminate operations, check no deletions are pending, then record replica
  counts and scale `deployment/argocd-applicationset-controller` and
  `statefulset/argocd-application-controller` in `argocd` to zero. Wait for their
  Pods to disappear; take and identify a successful VM backup before resuming
  live controllers. This adds no backup system; application-data backup remains
  Proxmox's responsibility. On restoring that backup, verify controllers are
  still stopped before reapplying installation manifests. Review storage, Git
  revision and operations; restore missing sealing keys; start ApplicationSet
  with automation disabled, then application-controller. Verify diffs and apps
  before enabling automation. Backups without this preparation do not guarantee
  a pause before reconciliation. Never treat Git as an application-data backup.
- **Someone else's setup:** fork/change repo URL, hosts, NAS/storage and node
  assumptions; generate their own keys and replace our SealedSecrets with their
  own values. Our ciphertext cannot initialize their credentials. Start apps
  with empty data, or their own independently restored data.
- **Normal operations:** add `ns-<name>/overlays/prod`, declare its namespace,
  seal secrets, protect PVCs, review and push. Directory discovery enrolls it.
  Explain polling/manual refresh, direct pushes versus merged PRs, and why
  editing a generated Application's sync policy is overwritten. Pause/resume
  through the manually applied ApplicationSet template; verify every generated
  Application inherits `automated.enabled: false` and terminate active syncs.
  This stops new automatic syncs, not manual sync or ApplicationSet generation/
  deletion. Resolve Git before resume; a commit alone does not apply this template.
- **Intentional removal:** distinguish a service inside `media` from a whole
  discovered Application. For a service, remove its workload resources first,
  retaining PVCs. For a whole app, save its resource inventory, remove its
  discovered folder only after verifying the Application has no Argo deletion
  finalizer, wait for the Application to disappear, then explicitly
  delete only its workload resources. Preserve PVCs/namespaces by default;
  delete selected PVCs only after the owner explicitly chooses data erasure.
  Local PV `Delete` permits cleanup; NAS PV `Retain` does not erase NAS files.
  Never select cascading Application deletion, delete a shared namespace, or use
  a blanket `kubectl delete -k`. Folder preservation does not disable pruning
  while an Application still exists.
- **Recovery references:** exact NAS backup address and password-manager
  passphrase location; VM backup versus external NAS/repo/key backups; UniFi
  Security log checks for blocked Git/registry/API traffic.

Add root `README.md` links, update the conflicting manual-Secret/media steps in
existing `setup.MD`, and update relevant system records with verified facts.
Validate commands and links; mark TODO 1–3 complete only after their checks pass.
Commit/push only reviewed task files. Leave no plaintext secrets or test artifacts.

## References

- [ApplicationSet Go templates](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/GoTemplate/)
- [Application deletion](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Application-Deletion/)
- [Sync protection options](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/)
- [Automatic sync](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)
- [Sealed Secrets key backup and renewal](https://github.com/bitnami/sealed-secrets)
- [age encryption](https://github.com/FiloSottile/age)
- [Tested Kubernetes versions](https://argo-cd.readthedocs.io/en/stable/operator-manual/tested-kubernetes-versions/)
- [Single-node versus HA](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/)
- [Argo ingress](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/) and [CLI login flags](https://argo-cd.readthedocs.io/en/stable/user-guide/commands/argocd_login/)
- [Secret adoption](https://github.com/bitnami/sealed-secrets#managing-existing-secrets) and [plaintext annotation gotcha](https://github.com/bitnami/sealed-secrets/blob/main/RELEASE-NOTES.md)
- [Terminating active syncs](https://argo-cd.readthedocs.io/en/stable/user-guide/commands/argocd_app_terminate-op/) and [StatefulSet scaling](https://kubernetes.io/docs/tasks/run-application/scale-stateful-set/)
- [K3s stopping behavior](https://docs.k3s.io/upgrades/killall) and [Argo operation processing](https://github.com/argoproj/argo-cd/blob/v3.5.3/controller/appcontroller.go)
