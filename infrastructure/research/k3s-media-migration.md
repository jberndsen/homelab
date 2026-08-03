# K3s media-stack migration: evidence and recommended approach

Researched 2026-07-10. This is an evidence-based migration brief, not a
deployment manifest. It supplements the system-of-record notes in
`infrastructure/`; those notes deliberately leave several facts unrecorded,
which must be verified before a workload is applied.

## Known constraints

- The single K3s control-plane node runs in Ubuntu VM `192.168.30.103` on
  VLAN 30. The NAS is `192.168.1.32`; it exports `media` read-write to that
  VM and the VM can reach NFS on TCP 2049. The existing node-secondary mounts
  use NFSv3, while no NFS protocol/version or exported subpaths are recorded
  for the new VM.
- Existing containers use UID `1000` and GID `988`, which have proven
  read/write access to the existing media directories. The NAS remains the
  media authority. Keep application databases/configuration on local K3s
  storage: Servarr explicitly warns that SQLite AppData on NFS/SMB will
  eventually corrupt. [Radarr FAQ](https://wiki.servarr.com/radarr/faq)
- The desired outcome is a fresh K3s deployment for each app, using each
  app's supported export/backup restore path; Plex history is intentionally
  discarded. Do not copy Compose volumes or application configuration
  directories.

## UNAS Pro NFS identity semantics (officially documented)

Ubiquiti's current Linux NFS guide documents that NFS is enabled under
**UniFi Drive → Settings → Services**, with a squash mode selected **per Shared
Drive**. It gives the export form as
`[UNAS IP]:/var/nfs/shared/[Shared_Drive_Name]`. [Accessing UniFi Drive from
Linux using NFS](https://help.ui.com/hc/en-us/articles/26277250895895-Accessing-Your-UniFi-Drive-from-Linux-Desktop-Using-NFS)

- **Documented fact — Collaborative Mode (All Squash):** Ubiquiti's recommended
  mode maps *all remote users* to an anonymous user. Consequently, a client
  process configured as `1000:988` is not evidence that the server will apply
  ownership as `1000:988`; the anonymous identity's actual write/create/rename
  behaviour must be tested on this NAS/export.
- **Documented fact — Isolated Mode (No Root Squash):** remote root users retain
  root privileges on the NAS. Ubiquiti documents it for advanced root-requiring
  workloads such as Proxmox Backup Server and VMware datastores. It cannot be
  switched back to All Squash; the Shared Drive becomes inaccessible through
  the Drive UI and SMB, NFS is the only supported access method, SMB Trash and
  SMB Encryption are disabled, and a storage quota cannot be set. Ubiquiti's
  official Drive 4.3.6 release notes say this mode was added in that version,
  call it permanent, and state that UniFi permanently loses access to the
  drive's content. [UniFi Drive 4.3.6 release
  notes](https://community.ui.com/releases/UniFi-Drive-Application-4-3-6/09cbf906-c66f-4974-b87f-8f4c82be294c)

**Not documented by Ubiquiti in the above primary sources:** the numeric
anonymous UID/GID used by Collaborative Mode; arbitrary UID/GID mapping or
id-mapping controls; NFS version; server export options/ACL semantics;
allowed-client restrictions; and whether non-root identities are preserved in
Isolated Mode. Do not infer any of these from community posts or a prior
firmware version.

**Plan consequence:** record the installed UniFi Drive version and inspect the
current `media` Shared Drive's selected mode before changing anything. Do not
turn the existing UI/SMB-managed media share into Isolated Mode merely to solve
container ownership: that is an irreversible storage-management decision.
Collaborative Mode is compatible with the stated use case only after a
representative NFS test as the workload UID/GID proves required media
operations. If a workload genuinely requires root-preserving semantics,
evaluate a newly dedicated Shared Drive and first establish an independent,
tested NFS backup/restore path.

## Storage foundation: recommended design

1. **Prove NFS from the K3s node before defining storage.** From a temporary,
   privileged *node-level* diagnostic only, establish the NAS export path,
   NFS version, UID/GID mapping and read/write/create/rename behaviour for
   UID:GID `1000:988`. Record the result in `infrastructure/network/system.md`.
   Do not infer an NFSv3 mount from node-secondary: the Kubernetes NFS volume
   source requires a server and exported path, and NFS protocol selection is
   an explicit mount option. [Kubernetes NFS volume](https://kubernetes.io/docs/concepts/storage/volumes/#nfs)

2. **Use statically provisioned PVs and PVCs for the already-existing NAS
   directories** (or a single static `media` PV only if access isolation is
   intentionally not wanted). Kubernetes documents NFS as a multi-writer
   volume type, while PersistentVolumes/PVCs provide the storage contract
   independent of a Pod's lifecycle. Use `ReadWriteMany` only after validating
   the NAS export; set `persistentVolumeReclaimPolicy: Retain` so deleting a
   claim cannot make Kubernetes treat the NAS data as disposable. The reclaim
   policy controls what happens to a PV after its claim is released. Set the
   static PV/PVC `storageClassName` deliberately (usually `""`) so a default
   local-path StorageClass cannot bind the claim by accident. Access modes are
   matching constraints, not write-protection enforcement; keep every writer
   at one replica.
   [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

3. **Mount the same in-container hierarchy in Transmission and all importing
   Servarr apps.** A practical canonical root is `/data`, with `/data/downloads`,
   `/data/movies`, `/data/series`, and `/data/music` beneath it. This makes the
   downloader's returned path valid in the importer and keeps source and
   destination on the same filesystem, which is required for a hardlink rather
   than a copy. Servarr says a download client reports paths to the app and
   Remote Path Mappings exist only to translate paths the app cannot see;
   an incorrect mapping is a common import failure. [Radarr download-client
   settings](https://wiki.servarr.com/radarr/settings#download-clients),
   [Sonarr remote path mappings](https://wiki.servarr.com/sonarr/settings#remote-path-mappings)

4. **Keep `/config` off NFS and provision it per app on local storage.** A
   node-local PVC is suitable on this one-node cluster, with a backup/export
   retained outside that PVC before upgrades or removal. It is not HA storage:
   loss of the VM/node loses node-local configuration unless its backup is
   recovered. K3s's local-path provisioner creates local storage by default;
   use it only for replaceable app state, never the NAS media authority.
   [K3s local-path provisioner](https://docs.k3s.io/add-ons/storage)

5. **Do not assume `fsGroup` fixes NFS permissions.** Set pod/container
   `runAsUser`, `runAsGroup`, and (where useful) `fsGroup` deliberately, but
   validate them against NAS export/root-squash/id-mapping rules. Kubernetes
   describes `fsGroup` as a volume-ownership mechanism; the storage plugin and
   server ultimately determine whether ownership changes are possible. Avoid
   recursive permission changes against the media tree. [Kubernetes security
   context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

## Servarr paths: `/data` is the hardlink-safe layout

`/movies`, `/tv` (or `/series`), `/music`, and `/downloads` are **not** paths
required by Radarr or Sonarr. They are convenient optional paths in
some image examples. For example, LinuxServer's Sonarr example labels `/tv`
and `/downloads` optional and explicitly says that this easy layout loses
hardlinks and atomic moves; its Radarr example likewise exposes optional
`/movies` and `/downloads`. [LinuxServer Sonarr
documentation](https://docs.linuxserver.io/images/docker-sonarr/),
[LinuxServer Radarr documentation](https://docs.linuxserver.io/images/docker-radarr/)

The authoritative Servarr recommendation is a **single common volume mounted
at the same in-container path, such as `/data`, in Transmission and every
importing *arr Pod**. It gives the concrete layout `data/downloads/...` and
`data/media/{movies,music,tv}`, mounting all of it as `/data` in Radarr,
and Sonarr. Then download and library folders appear as one filesystem
to the application, making a hardlink and atomic rename possible. Separate
`/downloads` and `/movies`/`/tv` mounts can look like different filesystems
inside a container even when the host storage is one filesystem, forcing
copy-and-delete instead. [Servarr Docker Guide](https://wiki.servarr.com/docker-guide),
[Radarr Docker installation](https://wiki.servarr.com/radarr/installation/docker)

**Conclusion for this migration:** retain the proposed unified mount but adopt
the Servarr naming precisely: mount the relevant NAS common parent at `/data`,
set Transmission's download directory beneath `/data/downloads` (optionally
separate torrent categories), and configure Radarr/Sonarr roots beneath
`/data/media/movies` and `/data/media/tv`. The actual NAS
directory names may differ, but the internal hierarchy and the one shared
mount must be identical. This supersedes the earlier illustrative
`/data/movies`, `/data/series`, `/data/music` wording where the NAS permits the
recommended common parent.

No Remote Path Mapping should be configured when Transmission reports paths
under that same `/data` hierarchy and each *arr Pod mounts it identically.
Servarr defines a Remote Path Mapping as a host-specific string replacement
for a download path the app cannot access; it is normally rare when both the
download client and *arr run in containers. Add one only for a demonstrated
different reported path (for example, a remote/non-container downloader), and
test the actual import. [Sonarr troubleshooting: Remote Path
Mapping](https://wiki.servarr.com/sonarr/troubleshooting#remote-path-mapping)

## Network and security gates

- A Pod has its own IP in the Kubernetes networking model, while normal
  Kubernetes service networking and CNI behaviour are separate from the VM's
  VLAN attachment. Therefore the fact that the VM is on VLAN 30 does **not**
  prove Transmission's torrent traffic will match the UniFi NordLynx routing
  policy. Before cutover, test the policy from the actual Transmission Pod and
  record whether the gateway sees pod IPs or the VM source IP after SNAT.
  Kubernetes requires Pod-to-Pod communication without NAT, but does not make
  the home gateway's external-routing policy implicit. [Kubernetes cluster
  networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- Confirm the exact UniFi traffic-route match (source network/IP, destination,
  and protocol) and an observable VPN-egress test. If it only matches the VM
  address, normal CNI egress masquerading may be sufficient; if it expects
  VLAN-tagged Pod interfaces, that is a separate multi-network design and must
  be specified rather than guessed. K3s documents Multus as the supported way
  to attach multiple network interfaces to Pods. [K3s Multus
  guide](https://docs.k3s.io/networking/multus-ipams)
- Start restrictive only after reachability is known: Kubernetes NetworkPolicy
  selects Pods but has no effect unless the installed network plugin enforces
  it. Confirm K3s's current CNI/policy-controller first, then allow only DNS,
  NAS NFS from the node path as needed, app-to-app service calls, and the
  intended external egress. [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/),
  [K3s network policy](https://docs.k3s.io/networking/networking-services#network-policy-controller)
- Inventory K3s's default Traefik and ServiceLB before exposing UIs. ServiceLB
  uses host ports for LoadBalancer Services and does not attach a Service to a
  VLAN; prefer ClusterIP plus the selected ingress mechanism for UIs that need
  LAN access, and avoid conflicting host ports. [K3s networking
  services](https://docs.k3s.io/networking/networking-services)
- Put web/RPC credentials and API keys in Kubernetes Secrets, not ConfigMaps
  or manifests committed in plaintext. Secrets are not encrypted at rest by
  default, so also verify K3s API-server encryption-at-rest configuration and
  RBAC access before treating them as protected. [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/),
  [K3s secrets encryption](https://docs.k3s.io/security/secrets-encryption)

## Application migration order and restore gates

The prescribed dependency order is sound, with these concrete gates:

1. **Storage foundation**: bind the media PVCs; run a disposable write/rename
   test as `1000:988`; then delete the test item. Validate the exact canonical
   paths from every consuming Pod.
2. **FlareSolverr**: deploy as a stateless internal Service. Its official
   container is the recommended installation because it includes the browser;
   it listens on 8191 by default and provides a startup browser test through
   `TEST_URL`. Size memory conservatively: each request launches a browser.
   [FlareSolverr README](https://github.com/FlareSolverr/FlareSolverr#readme)
3. **Transmission (non-VPN image)**: use the upstream `transmission`
   container/image rather than an image that bundles a VPN client. Persist only
   its local configuration PVC; mount NAS downloads at the canonical path.
   Configure and test its RPC endpoint/authentication and `download_dir`.
   Transmission's own RPC specification defines the default endpoint as
   `/transmission/rpc` on port 9091 and exposes `download_dir`/`incomplete_dir`
   session settings. Do not move active torrent state by copying the old
   container directory: either leave old torrents on node-secondary until they
   finish, or re-add deliberately after validating the data and tracker policy.
   [Transmission RPC specification](https://github.com/transmission/transmission/blob/main/docs/rpc-spec.md)
4. **Prowlarr**: make a built-in backup in the old UI, download it outside the
   old host, deploy Prowlarr fresh, then use **System → Backup → Restore
   Backup**. Add FlareSolverr as an Indexer Proxy and apply the same tag to the
   proxy and each relevant indexer; it is disabled with no matching tag and is
   used only when Cloudflare is detected. Only after that, create/test each
   Prowlarr application integration. [Prowlarr FAQ](https://wiki.servarr.com/prowlarr/faq),
   [Prowlarr settings](https://wiki.servarr.com/prowlarr/settings)
5. **Radarr and Sonarr**: for each app, take/download its built-in backup
   before any upgrade/cutover. Deploy it fresh with local `/config`, canonical
   NAS mounts and a stable/current app version; restore through **System →
   Backup → Restore Backup**. A backup restored across changed paths does not
   work without correcting those paths, so use matching Linux paths from the
   start or explicitly plan the path migration. Then add Prowlarr and
   Transmission, test the connection, inspect an existing completed download,
   and perform one controlled import. Servarr's documented Restore Backup flow
   is available for [Radarr](https://wiki.servarr.com/radarr/faq),
   [Sonarr](https://wiki.servarr.com/sonarr/faq). Do **not** add a Remote Path
   Mapping when all Pods share the same canonical path; add one only when the
   downloader reports a different path, and test it with an actual import.
6. **Plex**: deploy a fresh local config/metadata store and mount media
   read-only unless Plex is intentionally allowed to modify media-side files.
   Claim the server, recreate libraries, and scan the NAS paths. Plex confirms
   that scanning finds new/unmatched files and fetches their metadata. This
   deliberately avoids copying its database/watch history. [Plex scanning vs.
   refreshing](https://support.plex.tv/articles/200289306-scanning-vs-refreshing-a-library/)
7. **Homepage**: create it only after service URLs, auth and health checks are
   stable; recreate links/widgets from source-controlled configuration rather
   than copying its old runtime directory. Confirm each widget with its
   least-privilege app credential.

## Cutover, validation, and rollback

- Treat every application as a separate reversible change: retain the old
  Compose workload and its *manual exported backup* untouched until the new
  app passes its gate. Do not run the old and new download clients against the
  same active queue/path at once.
- For each deployment, wait for Pods to be Ready, inspect Events/logs, verify
  the Service from its intended client network, and make an app-native API/UI
  connection test. Kubernetes readiness prevents a Service from routing to a
  not-ready Pod. [Kubernetes readiness probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- Use a synthetic, legal test torrent or known completed sample to validate:
  Transmission writes on the NAS; each *arr sees the exact downloader path;
  a completed import is a hardlink (same filesystem/inode) when desired; the
  resulting media has correct owner/mode; Plex sees it after scan. Stop on the
  first failure and preserve logs/events before changing settings.
- Roll back by scaling/deleting only the new workload and returning traffic to
  the still-intact Compose service; do not delete NAS content or configuration
  PVCs during diagnosis. Deployment revisions support `kubectl rollout undo`
  for a failed image/spec rollout, but an app database restore remains the
  recovery path for a bad application-level migration. [Kubernetes Deployment
  rollback](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment)
- For singleton writers, use a rollout strategy that never has two writers
  active against the same app configuration/NAS target. Deployment rollback
  restores the Pod template, not NAS files or an application database; retain
  the manual backup for data-level recovery. Kubernetes reports a stalled
  rollout but does not automatically roll it back. [Kubernetes Deployment
  progress](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#failed-deployment)

## Image update decision (replace Watchtower separately)

Start with **pinned, immutable image digests**, a written change that updates
one workload at a time, the pre-change app backup, `kubectl rollout status`,
and the app's smoke test. Kubernetes supports rolling updates and rollback of
Deployments; mutable tags make the deployed artifact less auditable.
[Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

After the stack is stable, choose one of two explicitly operated models:

- **Manual/Git-reviewed updates:** update an image digest in the manifest,
  apply it, validate, and retain the prior digest for rollback. This is the
  lowest-complexity replacement for Watchtower on a single-node cluster.
- **GitOps image automation:** if a Git reconciliation controller is adopted,
  Flux Image Automation can scan registries and commit image updates to Git;
  it must be bounded by image policies and still needs backup/health gates.
  [Flux image update automation](https://fluxcd.io/flux/guides/image-update/)

Do not enable unattended updates for the stateful *arr/Plex/Transmission
workloads before restore and rollback are rehearsed.

## Home-LAN ingress names and DNS

### Naming standard

Use names below `home.arpa.` for LAN-only services, e.g.
`plex.home.arpa`, `radarr.home.arpa`, and `home.home.arpa` (or an unambiguous
alternative for Homepage). This is not an invented pseudo-TLD: RFC 8375 is an
IETF Standards Track RFC that designates `home.arpa.` as a special-use,
non-unique domain for residential home networks and says names below it are of
local significance. It replaced the deprecated `.home` recommendation. Do
not use `.local` for these unicast records. [RFC
8375](https://www.rfc-editor.org/rfc/rfc8375.html), [IANA special-use domain
registry](https://www.iana.org/assignments/special-use-domain-names/special-use-domain-names.xhtml)
RFC 8375 further requires local `home.arpa` queries not to be recursively
forwarded outside the logical home-network boundary; never make public DNS the
fallback for this zone.

### What current UniFi documentation confirms

- A UniFi Gateway can create local DNS records: A, AAAA, CNAME, MX, TXT and
  SRV, and can **Forward Domain** queries for a domain to another DNS server.
  Its UI location varies by Network version: Network 9.3 uses **Settings →
  Policy Engine → DNS**; Network 9.4 uses **Settings → Policy Table → Create
  New Policy → DNS**. CNAME support requires UniFi OS 4.3+ and Network 9.3+.
  [Ubiquiti: DNS Records and Local
  Hostnames](https://help.ui.com/hc/en-us/articles/15179064940439-UniFi-DNS-Records-and-Local-Hostnames)
- A client with a DHCP reservation can also receive a local hostname, but it
  is a Gateway cache record and resolves only for clients using the Gateway as
  DNS. A dedicated DNS server can instead be supplied to clients via the
  per-network DHCP DNS Server setting. The same Ubiquiti guide warns that
  content/domain filtering normally redirects DNS to the Gateway, so its
  documented integration steps must be followed if a custom resolver is used.
  [Ubiquiti: Content and Domain
  Filtering](https://help.ui.com/hc/en-us/articles/12568927589143-Content-and-Domain-Filtering-in-UniFi)
- The cited official UI documentation does **not** document wildcard DNS-record
  creation or a general DNS-zone editor. Although wildcard resource records
  are standardized DNS syntax, their support must be verified in the selected
  DNS backend rather than inferred from unrelated UniFi wildcard filtering
  features. [RFC 4592](https://www.rfc-editor.org/rfc/rfc4592.html)

### Recommended decision path

1. **Start with individual Gateway A records (recommended for this small,
   fixed stack).** First select and validate the K3s ingress/LAN IP. Create
   one `A` record per exposed service, all pointing to that address; make
   ingress route by Host header. This gives explicit, auditable names and
   requires no additional DNS workload. It depends on every relevant VLAN
   using the UniFi Gateway DNS service. Before applying records, capture the
   actual gateway model, UniFi OS version, Network version, client DNS
   settings, and the ingress IP; none are in the system-of-record yet.
2. **Use a dedicated authoritative internal DNS backend** when a wildcard
   (`*.home.arpa`), DNS-as-code, split-horizon rules, or broader zone control
   is genuinely required. Supply it by DHCP to client VLANs, or use UniFi's
   documented **Forward Domain** capability to forward `home.arpa` to it while
   retaining the Gateway resolver. Confirm the forwarder is reachable from the
   Gateway (including the necessary static route if it is reached through a
   VPN client). Choose an implementation only after confirming its backup,
   wildcard, update and failure behaviour.

For either option, keep Kubernetes service discovery (`*.svc.cluster.local`)
internal to the cluster and publish only intentional ingress names to LAN DNS.
DNS only maps a name to ingress; it does not provide TLS, authentication or
network authorization.

### Effect of owning `notech.foo`

Owning `notech.foo` adds a valid second naming option: use an **internal
split-horizon zone** such as `home.notech.foo`, with names such as
`plex.home.notech.foo`. This is preferable only if the intended longer-term
architecture needs the same owned namespace for both internal and external
service naming. It is not required for this LAN-only migration; `home.arpa`
remains the IETF-designated residential-only choice.

Do not publicly delegate `home.notech.foo` (via NS records) to a DNS server
that is reachable only on the LAN: external resolvers would be unable to use
that delegation. Instead, either create individual `A` records for
`*.home.notech.foo` names in the UniFi Gateway's **local** DNS facility, or
run a dedicated local authoritative/split-horizon DNS backend and direct or
Forward Domain `home.notech.foo` to it. The latter is required for a desired
wildcard because UniFi's current official DNS-record documentation does not
promise wildcard record creation. Keep private ingress addresses out of the
public `notech.foo` zone and decide separately which, if any, names should
have public DNS/ingress.

## Facts required before applying manifests

Ask and document these rather than guessing:

1. Exact NAS NFS export path(s), NFS version/options, root-squash/id-map
   behaviour, and whether the NAS export permits the VM to mount the required
   subdirectories.
2. The concrete NAS directory layout for downloads (complete/incomplete),
   movies, series and music; whether all sit on one filesystem/share; and
   whether hardlinks are wanted/possible.
3. Compose image names/tags, ports, data paths, existing app versions, current
   Transmission settings/active torrents, and each app's current backup status.
4. The exact UniFi traffic-route selector and observed egress source for a Pod,
   plus required inbound port-forwarding/remote-access behaviour.
5. K3s CNI/flannel configuration, network-policy enforcement, ingress choice,
   local storage capacity/backup destination, DNS, and the desired exposure
   model for each UI.
