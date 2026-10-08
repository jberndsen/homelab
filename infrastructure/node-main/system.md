# Current state

- `node-main` runs Proxmox VE 9.2.4 (owner screenshot, 2026-10-08) on VLAN
  `30` at `192.168.30.5`.
- `vmbr0` bridges `nic0` and uses gateway `192.168.30.1`. Its tagged VLAN `1`
  interface `vmbr0.1` has `192.168.1.39/24` and a direct route to the UPS at
  `192.168.1.34`; both interfaces are in `/etc/network/interfaces`.
- Proxmox is the only physical UPS shutdown target. VM `102` (`vm-haos`) and
  VM `103` (`vm-ubuntu-k3s`) have working QEMU guest agents and Start at boot
  enabled. Guest shutdown during an outage has not been tested.
- NUT `nut-client` is installed with `MODE=netclient` and a `secondary`
  monitor for `ups@192.168.1.34:3493` using username `ups` and TLS. The
  password is in `/etc/nut/upsmon.conf` and the owner's password manager.
  `nut-monitor.service` is enabled and running; authenticated UPS status
  queries work. The UPS certificate is not verified.
- Its Ubuntu Server VM (VM `103`) is `192.168.30.103`; it runs K3s
  `v1.36.2+k3s1` and is accessible over SSH. Its only virtual NIC is attached
  to `vmbr0`. The guest's default route and route to the Default VLAN NAS
  (`192.168.1.32`) both use `ens18` via `192.168.30.1`, not host interface
  `vmbr0.1`.
- Traefik is installed by K3s and ServiceLB exposes HTTP/HTTPS at
  `192.168.30.103`.
- On 2026-10-06, the K3s VM reported 8 vCPUs and approximately 16 GiB RAM,
  with no memory/disk/PID pressure. Traefik runs image
  `rancher/mirrored-library-traefik:3.7.4`; all media Ingress resources publish
  `192.168.30.103` in their status.
- The cluster uses Flannel VXLAN with Pod CIDR `10.42.0.0/24`.
- The VM has no `/dev/dri/renderD*` device; hardware transcoding is not
  available.
- Credentials are kept in untracked Kubernetes Secrets or the owner’s password
  manager; version-controlled manifests contain no credential values.
- `ns-media` is the manifest root for the Kubernetes `media` namespace.
- On 2026-10-06, all eight media Deployments were available and all nine PVCs
  were Bound. The eight local-path PVs use reclaim policy `Delete`; the shared
  NAS PV `media-nfs-pv` uses `Retain`. The owner chose to keep these policies.
- Ubuntu Server VM has package nfs-common installed.
- K3s secrets config in
  `infrastructure/node-main/secrets-encryption/k3s-server-config.yaml` is applied.
- The owner keeps the source YAML files on the dev laptop, outside the K3s VM.
- As of 2026-10-06, the repository is also published at
  `https://github.com/jberndsen/homelab`, with default branch `main`.
  The dev laptop's `origin` uses SSH; the public HTTPS clone URL is
  `https://github.com/jberndsen/homelab.git`.
- The owner's intended baseline for application data recovery is Proxmox VM
  backups.
- On 2026-10-08, the owner confirmed the pre-adoption backup gate: a fresh
  successful Proxmox backup of VM 103 with application PVC disk coverage.
  This is owner-confirmed; Execution 6 later identified the listed archive below.
  Recheck freshness if resuming much later or after data/storage changes. The
  owner later authorized checkpoint 3 using this recorded confirmation.
- The owner selected `smb://192.168.1.32/backups/NUC/kubernetes` for encrypted
  Sealed Secrets key backups and an adjacent recovery `README.md`. On the dev
  laptop this is mounted at `/Volumes/backups/NUC/kubernetes`; SMB connectivity
  and directory access were verified on 2026-10-06. The backup passphrase
  is stored in the owner's password manager as `Homelab Sealed Secrets key backup`
  (owner confirmed on 2026-10-08). See the verified backup below.
- On 2026-10-08, Argo CD `v3.5.4` was manually bootstrapped in namespace
  `argocd` using the standard non-HA upstream install and preserved upstream
  NetworkPolicies. Its six Deployments and application-controller StatefulSet
  were ready, with seven Ready Pods and no restarts. Argo does not manage itself;
  its manifests are in `ns-argo/` outside production directory discovery.
- `http://argocd.home.arpa` returned HTTP 200 through Traefik at
  `192.168.30.103`; the server runs with `server.insecure: "true"`. Access is
  intended for the trusted LAN. After an admin-password reset, the owner
  confirmed successful login, password rotation and replacement-password login.
  The final authentication audit corroborated successful password change at
  `2026-10-08T18:23:58Z` and successful subsequent logins. The initial-password
  Secret was verified absent; credentials remain outside Git and tool output.
- On 2026-10-08, Execution 3 created ApplicationSet `node-main` in `argocd`.
  It generated exactly `media` → namespace `media` and `sealed-secrets` →
  namespace `kube-system`, reading public GitHub `main` anonymously over HTTPS.
  At checkpoint 2 both Applications had automatic sync, pruning and self-healing
  disabled,
  with no resource-deletion finalizers. The set preserves resources on deletion.
  Argo's own manifests and bootstrap namespaces remain manually managed.
- Sealed Secrets `v0.40.0` was manually synced through Argo at commit `5f95703`
  after release/advisory review. All 11 resources synced successfully; its CRD
  is Established and protected by `Prune=confirm,Delete=confirm`. Its controller
  is Ready, with Application status Synced/Healthy and sync Succeeded. It uses
  upstream default 30-day key renewal. Laptop clients are `kubeseal v0.40.0`
  and `age v1.3.2`.
- Media adoption completed at checkpoint 3; checkpoint 4 subsequently enabled
  automatic sync, pruning and self-healing. Execution 6 subsequently completed
  the final operating/recovery documentation audit and closed TODO 1–3.
- On 2026-10-08 at 18:43:34 UTC, independent key recovery passed using NAS file
  `sealed-secrets-keys-2026-10-08T184312Z.yaml.age` in the backup directory above.
  Its ciphertext SHA-256 is
  `45937195f88f225d60b7ede777779ca032bb6b096a6c1378e6e0a68f6b2f3bbd`.
  Backup inventory contains key `sealed-secrets-keywg7lv`, with certificate
  SHA-256 `b77a5acd6d48fc4bf2d7620d1da9a443cd4c205789efbf0de1726c3e14e6b1a4`.
  All current controller keys and the active certificate match that inventory.
  Decrypting the actual NAS file matched the full key export; live controller
  validation and offline `kubeseal --recovery-unseal` recovered a harmless test.
  No test objects were applied, production keys were not replaced, and protected
  local plaintext was cleaned up. The adjacent NAS `README.md` contains exact
  backup/verification/restore/cleanup commands and the public fingerprint inventory.
  The owner confirmed password-manager storage; checkpoint 2 is complete.
- Immediately after bootstrap, the node used approximately 127m CPU and
  3069 MiB memory (19%); Argo Pods collectively used approximately 168 MiB.
  These are initial idle observations, not workload sizing guarantees.
- On 2026-10-08, local and GitHub `main` matched `371650a` before implementation.
  A pattern audit of all 27 reachable commits (189 distinct file blobs) found
  placeholders, structural matches and documented fake node-secondary values;
  no actual credentials were identified by that audit. A credential-free
  pre-install image/storage baseline is held privately under ignored
  `plans/runtime/` on the dev laptop.
- Checkpoint 1 passed on 2026-10-08: all three Argo CRDs Established, seven
  NetworkPolicies retained, UI/health/version endpoints returned HTTP 200,
  `cluster/` had no remaining diff, and no node pressure was present. All eight
  media Deployments remained available and all nine PVCs Bound. Media Deployment
  specifications/images and PVC/PV UIDs, bindings and PV specifications matched
  the private pre-install baseline. Final idle usage was approximately 93m CPU
  and 3012 MiB RAM (18%) for the node, with 144 MiB across Argo Pods.
- After controller installation, all eight media Deployments were available
  and all nine PVCs Bound. Their specs and UIDs, plus PV specs, UIDs and bindings,
  matched the private pre-Argo baseline. No node pressure was present. Idle
  observations were about 84m CPU / 3312 MiB RAM (20%) for the node and
  1m CPU / 11 MiB RAM for the Sealed Secrets controller.
- The final checkpoint-2 read-only confirmation on 2026-10-08 passed: node Ready without pressure,
  all seven Argo Pods Ready with zero restarts, UI/health HTTP 200 and seven
  upstream NetworkPolicies retained. Sealed Secrets remained Synced/Healthy,
  with its CRD protection intact. Both Applications had automatic sync disabled,
  no deletion finalizers and no pending operation; media had no sync history.
  All media Deployment/PVC/PV specs and UIDs still matched baseline, all eight
  Deployments were available, and all nine PVCs Bound. The NAS ciphertext
  checksum and coverage of every current key/active certificate were unchanged;
  no production SealedSecrets, checkpoint-2 test objects or plaintext temporary directory
  remained. Checkpoint 3 was unstarted at that audit.
- Checkpoint 3 completed on 2026-10-08 using media manifest commit `450da82`.
  Before sealing, the actual NAS backup checksum and coverage of every current
  key and active certificate matched the verified checkpoint-2 inventory.
  Strict-scope `homepage-widgets` and `transmission-rpc` SealedSecrets adopted
  the existing Secrets after managed annotations were added and historical
  last-applied annotations removed. Both generated Secrets retained their
  original names, types, data and UIDs; no plaintext was written to Git/output.
- Media's 37 desired resources were manually synced in seven groups: two
  SealedSecrets, nine PVCs, Homepage, Transmission, the four Arr apps, Seerr,
  then Jellyfin. Every group succeeded after diff review. All nine live PVCs
  now carry `Prune=confirm,Delete=confirm`; the controller CRD retains the same
  protection. Both SealedSecrets also have deletion confirmations because their
  generated Secrets are controller-owned. Reclaim policies are unchanged.
- Clean committed-checkout rendering passed without ignored Secret files.
  All existing media resource UIDs/specs and ConfigMap contents matched their
  pre-adoption state. Deployment/PVC/PV specs, UIDs and bindings matched the
  private pre-Argo baseline. Ten selected application settings files were
  byte-for-byte unchanged; the same eight media Pods remained Ready with zero
  restarts. No application restart was necessary.
- Seven application web endpoints returned HTTP 200. Transmission rejected
  anonymous RPC and accepted its original credentials for read-only session-get.
  Homepage reached all four widget backends through Kubernetes service DNS with
  its existing API keys. The owner confirmed all four widgets show data and
  Jellyfin plays existing NAS media. Both Applications are Synced/Healthy,
  without active operations or deletion finalizers. At that checkpoint-3 pause,
  automatic sync, pruning and self-healing were still disabled.
- Checkpoint 4 completed on 2026-10-08. Initial local/GitHub main was `9238f7b`;
  both Argo diffs were empty and storage matched the private pre-Argo baseline.
  ApplicationSet policy commit `529fc09` enables automatic sync, pruning and
  self-healing for exactly `media` and `sealed-secrets`, retaining allowEmpty=false,
  preserveResourcesOnDeletion=true and no Application deletion finalizers.
  All nine PVCs, both SealedSecrets and the controller CRD retain
  `Prune=confirm,Delete=confirm`; no deletion approval was added.
- Disposable ConfigMap `argocd-reconciliation-check` demonstrated Git deployment
  from `e51d741` (19:35:29 UTC), correction of a live `manual-test` value back to
  `from-git` with the same UID (19:38:02 UTC), and pruning after Git removal
  `c58fa7f` (19:39:20 UTC). All operations were automatic; comparison refreshes
  were used, without manual sync. Exactly that ConfigMap was pruned. It had no
  workload references and held no credentials; no test resource/manifest remains.
- Final checkpoint-4 checks: both Applications Synced/Healthy, no pending
  operation/diff/deletion, unchanged baseline Deployment/PVC/PV specs, UIDs and
  bindings, eight original Ready media Pods with zero restarts, unchanged Secret
  values/types/UIDs, existing ConfigMaps and ten selected settings files.
  Seven web endpoints, Transmission authenticated read-only RPC and Homepage's
  four backend APIs passed. Argo HTTP 200, seven Ready Argo Pods with zero
  restarts, four Established CRDs, seven retained NetworkPolicies, ready sealing
  controller, node health and empty manual cluster diff passed. Node use was
  about 80m CPU / 3991 MiB RAM (24%), an idle observation only. The owner's
  widget/playback confirmation is from checkpoint 3. Paused after checkpoint 4;
  automation stays enabled and Execution 6 was not started.

- Execution 6 reviewed the owner's Proxmox screenshot on 2026-10-08. VM 103's
  Backup view lists `vzdump-qemu-103-2026_10_08-20_53_04.vma.zst` on storage
  `local`, displayed date `2026-10-08 20:53:04` (timezone not verified).
  This identifies the archive alongside the earlier owner-confirmed success
  and application PVC disk coverage. The screenshot does not establish a
  restore test, off-host copy or controllers stopped at backup time.
- Execution 6's read-only audit on 2026-10-08 started from local/GitHub `main`
  `a51b474`. Both Applications were Synced/Healthy with enabled/prune/selfHeal=true,
  allowEmpty=false, resource preservation=true, no finalizers, operations,
  pending pruning/deletions or deletion approvals. All nine PVCs, both
  SealedSecrets and the controller CRD retained their confirmation annotations.
  All 41 baseline media object identities, Deployment/PVC/Service/Ingress specs,
  and all nine PV specs/UIDs/bindings matched the private baseline. The original
  eight media Pods remained Ready with zero restarts. Node health, seven Ready
  Argo Pods without restarts, four Established CRDs and seven NetworkPolicies passed.
- The Execution 6 audit rechecked the actual encrypted NAS key-backup checksum,
  complete live-key inventory and active certificate against the checkpoint-2
  independent recovery receipt. No backup was decrypted/restored, no keys were
  changed and no controllers were paused. VM restore, NAS-media recovery and
  off-host copies of the VM/key backups remain unverified. The NAS recovery
  README's old pre-checkpoint-4 status was corrected; Git documentation now
  distinguishes fresh-cluster, ordinary VM and guarded VM recovery.

- Execution 6's read-only smoke checks passed again: seven web endpoints,
  Transmission anonymous rejection and authenticated `session-get`, all four
  Homepage backend APIs, unchanged Secret values/types/UIDs, existing ConfigMaps
  and ten selected settings files. Argo HTTP returned 200 and manual `cluster/`
  diff was empty. Both reconciliation controllers still had one replica.
  All four manifest roots rendered from a clean tracked checkout; the disabled
  recovery copy passed server dry-run without persistence. Local links and
  shell/embedded Python syntax passed. The owner's widget/playback confirmation
  remains from checkpoint 3. Documentation changed; live configuration did not.
