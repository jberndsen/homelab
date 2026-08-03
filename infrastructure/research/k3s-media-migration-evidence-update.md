# K3s media migration: primary-source evidence update

Researched 2026-07-10. This note reviews
[`migration-plan.md`](../../migration-plan.md) against the current Compose file
and current first-party documentation. It is an evidence update for the
step-by-step implementation plans, not a replacement for the agreed plan and
not a deployment manifest.

## Verdict

The dependency order and the main architecture are sensible: establish NFS,
then FlareSolverr, Transmission, Prowlarr, each importing Servarr application,
Plex, and Homepage; retain `/config` on node-local storage; use application
backups rather than copied Docker volumes; and replace Watchtower initially
with reviewed digest updates.

The plan is **not yet mechanically complete enough to turn into final
manifests without addressing the gates below**. The most important corrections
are:

1. Use one NFS mount of the common `media` parent at `/data` in Transmission,
   Radarr and Sonarr. Separate download/library PVs or separate mounts
   recreate the filesystem boundary that the Servarr layout is intended to
   avoid and can make hardlinks fail even when the server directories reside
   on the same NAS filesystem. Servarr's documented layout is a single common
   volume, mounted at the same container path, with downloads and libraries
   beneath it. [Servarr Docker Guide](https://wiki.servarr.com/docker-guide)
2. Do not blindly set Pod-level `runAsUser: 1000` on LinuxServer images. Their
   normal model starts the container's init as root and uses `PUID`/`PGID` to
   run the application with the requested identity. LinuxServer explicitly
   says its images generally use this mapping and warns that forcing a custom
   container user is not its normal supported path. Preserve `PUID=1000` and
   `PGID=988`, then verify the application process and created files; harden
   each image only after its init behavior is proven.
   [LinuxServer PUID/PGID](https://docs.linuxserver.io/general/understanding-puid-and-pgid/),
   [LinuxServer support policy](https://docs.linuxserver.io/misc/support-policy/)
3. A hostname-only HTTP Ingress is not automatically equivalent to the old
   Plex server exposed directly on TCP 32400. Plex bridge/container discovery,
   claiming, client connection advertisement, and any remote access must be
   tested explicitly. Plex documents **Custom server access URLs** for reverse
   proxies/unusual networking, and LinuxServer says a bridge-networked fresh
   server needs a short-lived claim token. [Plex network settings](https://support.plex.tv/articles/200430283-network/),
   [LinuxServer Plex image](https://docs.linuxserver.io/images/docker-plex/)
4. Homepage's Docker socket was a functional integration, not just a mount.
   Removing it is correct for K3s, but exact behavior requires inventorying the
   existing Homepage YAML and deciding whether each Docker status/discovery
   feature becomes a Kubernetes integration, a normal service widget, or is
   intentionally removed. Homepage's Kubernetes mode requires a ServiceAccount
   and read RBAC; grant only the resources actually needed.
   [Homepage Kubernetes installation](https://gethomepage.dev/installation/k8s/)
5. Stateful SQLite applications need singleton, graceful replacement. Use one
   replica, `strategy.type: Recreate`, a sufficient termination grace period,
   and verify clean termination before accepting rollout. A Deployment
   rolling update can otherwise create a surge Pod; Kubernetes also warns that
   manually deleting Pods can temporarily produce more Pods than the desired
   count. [Kubernetes Deployment strategy](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)
6. A fresh Transmission instance does not preserve the effective behavior of
   the existing `haugene/transmission-openvpn` merely by reproducing the three
   directory environment variables. Before cutover, inventory the effective
   download/incomplete/watch paths, RPC and host whitelists, peer port,
   encryption, ratio/seeding, queue, blocklist, speed, and proxy behavior.
   Recreate the desired settings manually in the selected non-VPN image.
7. Resolve whether NordVPN routing is meant for **only Transmission** or for
   every workload on the K3s VM. The VM itself is recorded on VLAN 30. If the
   UniFi route selects the whole VLAN/VM and normal Pod egress is masqueraded
   to the VM, Plex, Homepage, FlareSolverr, and all Servarr traffic will also
   use NordVPN. That is not equivalent to the old Compose host and can affect
   Plex connection advertisement. A control Pod and the Transmission Pod must
   both be tested before choosing ordinary CNI networking versus a dedicated
   Transmission network identity.

## Foundation requirements

### NFS client and static storage

- Install/verify Ubuntu's `nfs-common` package on the K3s node before creating
  an NFS-consuming Pod. Ubuntu identifies it as the client-side NFS support
  package. A successful TCP check to port 2049 is not a substitute for an
  actual NFSv3 mount of the exact export; validate the full mount operation
  from the VM and then through kubelet with a test Pod.
  [Ubuntu NFS client documentation](https://documentation.ubuntu.com/server/how-to/networking/install-nfs/index.html)
- Create one static NFS PV for
  `192.168.1.32:/var/nfs/shared/media`, with `mountOptions` explicitly selecting
  the protocol proven by the test (currently expected to be `nfsvers=3`),
  `ReadWriteMany`, `persistentVolumeReclaimPolicy: Retain`, and
  `storageClassName: ""`. Bind one matching PVC in `media`. Kubernetes states
  that NFS supports multi-writer access, that `Retain` requires manual
  reclamation, and that mount options are not validated—an invalid option
  simply causes mounting to fail. [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- Mount that single claim at `/data` in Transmission and the three importing
  Servarr Pods. Because this is one parent mount, its case-sensitive container
  paths are the NAS paths themselves: `/data/Downloads/{completed,incomplete,watch}`,
  `/data/Video/Movies`, `/data/Video/TV Shows`, and `/data/Music/Lossless`.
  The earlier lowercase `/data/downloads` and `/data/media/...` names cannot be
  "mapped" to those existing directories with separate Kubernetes mounts
  without reintroducing mount boundaries. Do not implement each subtree as a
  separate PV or a separate volume mount if hardlinks are required. Renaming
  NAS directories or introducing server-side links is out of scope unless the
  owner explicitly chooses it after a NAS backup.
- The PV/PVC requested `capacity` is matching metadata for this pre-existing
  NFS export; it does not impose a NAS quota. The node-local `local-path` PVC
  size is likewise not guaranteed to act as a hard filesystem quota. K3s
  documents local-path as its default local storage provisioner.
  [K3s storage](https://docs.k3s.io/add-ons/storage)
- `/config` for Transmission, Prowlarr, Radarr, Sonarr, and Plex must
  remain on separate local-path PVCs. Servarr warns that its SQLite database on
  NFS/SMB will eventually corrupt. [Radarr FAQ](https://wiki.servarr.com/radarr/faq)
- For Plex, mount only the movie and TV paths it needs and set those
  `volumeMounts` read-only. Kubernetes notes that `readOnly` is per container
  mount, not a change to the underlying volume. [Kubernetes volumes](https://kubernetes.io/docs/concepts/storage/volumes/#read-only-mounts)

The NFS acceptance test must run inside a Pod using the same effective
application identity and must prove, in every required subtree: list/traverse,
read, create, write, fsync/close, rename, delete, and creation of a hardlink
from a completed-download test file into the applicable library subtree. Check
that the two names have the same inode and that deleting the test names leaves
no data behind. This is also the only reliable answer to the currently unknown
UNAS squash/anonymous identity behavior.

### UID, GID, and volume ownership

- Keep `PUID=1000`, `PGID=988`, `TZ=Europe/Amsterdam`, and an explicitly chosen
  `UMASK` for LinuxServer images. The Compose file proves the IDs, but not the
  desired umask; Servarr recommends a shared-group layout equivalent to files
  `664`, directories `775`, and umask `002` when multiple services must write.
  Confirm the current effective modes before choosing it.
  [Sonarr permissions](https://wiki.servarr.com/sonarr/faq#permissions),
  [LinuxServer Radarr image](https://docs.linuxserver.io/images/docker-radarr/)
- Avoid Pod-level `runAsUser`/`runAsGroup` for LinuxServer containers unless a
  test of that exact digest demonstrates non-root startup. `PUID`/`PGID` govern
  the application process; inspect PID 1 and the app process separately, and
  verify ownership on both `/config` and a disposable NAS file.
- Do not rely on `fsGroup: 988` to repair a squashed NFS export. Kubernetes may
  recursively change ownership/permissions for supported volumes, which is
  expensive on a large media tree and may be refused by the NFS server. If
  `fsGroup` is needed for a local PVC, use it deliberately; do not use it as a
  speculative recursive fix for the media PV. Kubernetes documents
  `fsGroupChangePolicy: OnRootMismatch`, but only for volume types that support
  fsGroup-controlled ownership. [Kubernetes security context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
  The Kubernetes NFS volume API specifically reports no ownership-management
  support for NFS volumes. [Kubernetes NFSVolumeSource API](https://kubernetes.io/docs/reference/kubernetes-api/core-resources/persistent-volume-v1/#NFSVolumeSource)
- Homepage's own official Kubernetes example runs as UID/GID 1000 and drops
  capabilities, so it can use a conventional non-root container security
  context independently of the LinuxServer images.
  [Homepage Kubernetes installation](https://gethomepage.dev/installation/k8s/)

### VLAN 30, VPN routing, and fail-closed behavior

K3s defaults to Flannel VXLAN and IPv4 egress masquerading. Therefore Pods
normally have cluster addresses internally while off-cluster IPv4 traffic may
be seen as coming from the node. This makes the actual UniFi policy selector
and observed source address a release gate, not an assumption.
[K3s basic network options](https://docs.k3s.io/networking/basic-network-options)

Read-only cluster inventory on 2026-07-10 confirmed that this cluster uses
Flannel VXLAN with Pod CIDR `10.42.0.0/24`. It also confirmed that packaged
Traefik `3.7.4` is exposed by ServiceLB as a `LoadBalancer` whose current
external address is the node address `192.168.30.103`, on ports 80 and 443.
These are live observations, not a promise that UniFi routing or LAN DNS is
already correct.

This is a design blocker, not just a Transmission smoke test. Because the K3s
VM is `192.168.30.103` on VLAN 30, a VLAN-wide UniFi route is expected to apply
to every workload whose egress is presented as that VM. Test public IPv4 from
both Transmission and a neutral control Pod. Ask the owner whether routing all
K3s workloads through NordVPN is intended. If only Transmission must use the
VPN and ordinary masqueraded egress cannot be distinguished, it needs a
dedicated address/interface or a narrower UniFi-selectable identity. K3s
documents Multus for attaching a second interface, but it should be introduced
only after the VM/Proxmox VLAN topology and desired routing boundary are known.
[K3s Multus](https://docs.k3s.io/networking/multus-ipams)

The network-preliminary step must record and verify:

1. The installed UniFi OS and Network versions and the exact Policy-Based
   Route: source selector, destination, selected NordVPN client interface, and
   **Kill Switch enabled**. Ubiquiti documents that enabled means traffic stops
   if the selected interface goes down; disabled allows fallback to another
   interface. [UniFi Policy-Based Routing](https://help.ui.com/hc/en-us/articles/12566175125783-UniFi-Gateway-Policy-Based-Routing)
2. Whether the route selects the complete VLAN 30 network, the VM device/IP, or
   something else. From an actual Transmission Pod **and a neutral control
   Pod**, record the public IPv4 and gateway-observed source. Compare them with
   the VM and normal WAN public IPs.
3. A controlled failure test in which the VPN client is made unavailable by
   the owner in UniFi. Confirm the Pod loses public internet and does not use
   normal WAN, then confirm DNS, LAN administration, and NAS access still work
   as intended. This is a manual, potentially disruptive confirmation; the
   migration step must pause for the owner before and after it.
4. The precise cross-VLAN firewall allowances required for VM/Pod operation,
   including the actual NFSv3 mount transaction, DNS, LAN client access to the
   ingress IP, and administration. Ubiquiti documents that zone rules are
   directional and that explicit allows must precede a block policy.
   [UniFi zone-based firewalls](https://help.ui.com/hc/en-us/articles/115003173168-Zone-Based-Firewalls-in-UniFi)

Do not add Multus or a VLAN interface to Transmission unless the above test
proves ordinary node-masqueraded egress cannot match the UniFi rule. That would
be a materially different network design, not a prerequisite implied by K3s.

### Services, DNS, ingress, and probes

- Use `ClusterIP` Services. Internal consumers in the same namespace can use
  `http://flaresolverr:8191` and `http://transmission:9091`; fully qualified
  names are `<service>.media.svc.cluster.local`. Kubernetes creates these DNS
  records for normal Services. [Kubernetes Service DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- Inspect the existing Traefik LoadBalancer Service before creating LAN DNS.
  K3s deploys Traefik and ServiceLB by default; ServiceLB normally uses host
  ports 80/443 and advertises node addresses. In the current single-node design
  the likely ingress A-record target is `192.168.30.103`, but it must be read
  from and tested against the live Service rather than copied into plans as an
  assumption. [K3s networking services](https://docs.k3s.io/networking/networking-services)
  The 2026-07-10 inventory found `192.168.30.103`; the implementation plan must
  still test port 80 from every intended client VLAN before creating DNS.
- Use individual `home.arpa` A records and Host-based Ingress rules only after
  the ingress target responds from the client VLAN. Keep FlareSolverr internal;
  it does not need LAN Ingress.
- Define named container ports and Services for every app. Prefer a generous
  TCP `startupProbe` plus TCP `readinessProbe` for restored Servarr apps and
  Transmission during the baseline migration. Restored authentication can
  turn an otherwise healthy HTTP endpoint into `401`, and an aggressive
  liveness probe can restart an app during database migration. Add a liveness
  probe only when a first-party, authentication-stable endpoint and failure
  semantics are known. Kubernetes documents that startup probes suppress
  liveness/readiness until startup succeeds and that failed readiness removes
  a Pod from Service endpoints. [Kubernetes probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- Homepage is the exception with an official health endpoint,
  `/api/healthcheck`. Its `HOMEPAGE_ALLOWED_HOSTS` must include the exact
  `homepage.home.arpa` Host and the Pod-IP host form used by the probe; `*` is
  explicitly discouraged. [Homepage allowed hosts](https://gethomepage.dev/installation/),
  [Homepage Kubernetes example](https://gethomepage.dev/installation/k8s/)
- Set all SQLite/config writers to one replica and `Recreate`. Give them enough
  termination grace to exit cleanly; during validation, delete a Pod once and
  verify the old process terminates and the replacement opens its database
  cleanly. A crashing process is restarted by Kubernetes without a liveness
  probe, so omitting an untrustworthy liveness probe does not disable basic
  restart behavior.

## Application-specific evidence and plan consequences

### FlareSolverr

- Use the project's GHCR image pinned by digest, port 8191, no PVC, and an
  internal ClusterIP Service. The project recommends its container because it
  includes the required browser. [FlareSolverr README](https://github.com/FlareSolverr/FlareSolverr#installation)
- `TEST_URL` performs a real browser request at startup. Keep a generous
  startup threshold and validate logs plus one controlled `POST /v1`
  `request.get`; merely opening the TCP port does not prove browser operation.
  Each request launches a browser and can use substantial memory.
  [FlareSolverr usage and environment](https://github.com/FlareSolverr/FlareSolverr#usage)
- The README states that the listed captcha solvers currently do not work.
  Retaining `CAPTCHA_SOLVER=none` matches current behavior; do not claim this
  migration can solve captcha challenges.

### Transmission

- Select a concrete non-VPN image before manifest generation. A low-change
  choice is `lscr.io/linuxserver/transmission` pinned by digest: it uses the
  same LinuxServer PUID/PGID model as most of this stack, exposes Web/RPC on
  9091, supports `USER`/`PASS`, `WHITELIST`, `HOST_WHITELIST`, and `PEERPORT`,
  and keeps configuration in `/config`. This is a LinuxServer-maintained image,
  not an image published by the Transmission project.
  [LinuxServer Transmission image](https://docs.linuxserver.io/images/docker-transmission/)
- Set credentials via Secret-backed `USER` and `PASS` for that image; do not
  reuse the plaintext password currently committed in `.env`. Rotate it during
  migration. LinuxServer explicitly says not to hand-edit username/password in
  `settings.json`; it also says the container must be stopped before editing
  other settings or changes will not persist.
- Set the effective paths to
  `/data/Downloads/completed`, `/data/Downloads/incomplete`, and
  `/data/Downloads/watch`. Confirm incomplete-directory enablement, not just
  its path. Transmission's RPC is normally
  `http://host:9091/transmission/rpc`, uses a session-ID anti-CSRF exchange,
  optional HTTP Basic authentication, and host-header whitelisting.
  [Transmission RPC specification](https://github.com/transmission/transmission/blob/main/docs/rpc-spec.md),
  [Transmission configuration](https://github.com/transmission/transmission/blob/main/docs/Editing-Configuration-Files.md)
- A read-only inspection on 2026-07-10 confirmed the current effective values:
  download `/data/completed`, incomplete `/data/incomplete` with incomplete
  mode enabled, watch `/data/watch` with watch mode enabled, peer port `51413`,
  RPC port `9091`, RPC URL `/transmission/`, authentication required, RPC IP
  and Host whitelists disabled, and umask `002`. The new case-sensitive paths
  above are intentional path changes required by the one-parent-mount design;
  all restored Servarr download-client paths must be updated in the same
  cutover step.
- Inventory whether the old image's `WEBPROXY_ENABLED=true` behavior is used.
  The Compose file does not publish port 8888 to the host, but another Compose
  container could still have used it over the internal network. Do not deploy a
  replacement proxy unless current use is demonstrated; if it is used, that is
  a separate behavior that `linuxserver/transmission` does not promise.
- The current Compose publishes only RPC/UI port 9091, not a torrent peer port.
  Decide explicitly whether preserving that behavior is desired. Exposing TCP
  and UDP 51413 would be a new behavior, not a requirement inferred from the
  LinuxServer example.
- Drain and confirm the old active queue, disable normal downloading on the new
  instance until VPN tests pass, then perform one legal controlled download.
  Verify path, owner/group/mode, RPC from each Servarr app, public egress, and
  fail-closed behavior before enabling normal use.

### Prowlarr

- Take and download a built-in backup, then stop old Prowlarr before restoring
  the new instance. Use a stable build whose application version is at least
  the observed develop database version. Servarr warns that a database from a
  newer application version cannot be used by an older version; the tag name
  alone is not proof. [Prowlarr FAQ](https://wiki.servarr.com/prowlarr/faq)
- A restored backup may contain application URLs, API keys, and sync settings
  that still target the old Radarr/Sonarr endpoints. Review these before
  permitting a sync. Point in-cluster application entries at ClusterIP DNS as
  each destination exists, test them, and only then enable the intended sync
  level. Prowlarr documents that Full Sync overwrites many destination indexer
  settings. [Prowlarr application settings](https://wiki.servarr.com/prowlarr/settings#applications)
- Configure FlareSolverr as `http://flaresolverr:8191`. Give the proxy and each
  intended indexer the same tag; an untagged proxy is disabled and it is used
  only when Cloudflare is detected. [Prowlarr FlareSolverr settings](https://wiki.servarr.com/prowlarr/settings#indexer-proxies)

### Radarr and Sonarr

- For each app separately: trigger its built-in backup while healthy, download
  the ZIP off the host, then stop the old container before restoring. The
  built-in backup is the supported live workflow; the shutdown requirement in
  Servarr's documentation applies when copying AppData/database files directly,
  which this migration intentionally does not do.
  [Radarr backup/restore](https://wiki.servarr.com/radarr/faq#how-do-i-backuprestore-radarr)
  and [Sonarr backup/restore](https://wiki.servarr.com/sonarr/faq#how-do-i-backuprestore-sonarr)
- Restore only into the same or newer application/database version. After
  restore, authentication and API keys come from the backup, not a Kubernetes
  Secret, so verify access before judging an HTTP probe unhealthy.
- Change roots to `/data/Video/Movies` and `/data/Video/TV Shows`; configure
  Transmission to report paths under
  `/data/Downloads`. Do not add a Remote Path Mapping when both sides see the
  same path. Servarr defines mappings as translation for a path the app cannot
  otherwise access. [Sonarr Remote Path Mapping](https://wiki.servarr.com/sonarr/settings#remote-path-mappings)
- A backup does not move library files. After changing a root, ensure all
  existing entries point at the new root and perform an app rescan. Test a
  completed import, verify source and destination inode equality where
  hardlinking is expected, and verify seeding/removal policy. Servarr documents
  that completed torrents remain for seeding and imports hardlink when the
  layout supports it. [Radarr torrent import behavior](https://wiki.servarr.com/radarr/faq#why-are-there-two-files-why-is-there-a-file-left-in-downloads)

### Plex

- Use the LinuxServer Plex image pinned by digest with fresh local `/config`,
  `PUID=1000`, `PGID=988`, `VERSION=docker`, and read-only movie/TV mounts.
  Budgeting must account for LinuxServer's warning that `/config` can grow
  beyond 50 GB for a large library; the current 30 GiB estimate is a starting
  observation threshold, not a safe universal ceiling.
  [LinuxServer Plex image](https://docs.linuxserver.io/images/docker-plex/)
- Before migration, manually record the existing library names, types,
  language/agent choices, folder paths, and whether any clients rely on local
  discovery, DLNA, or Remote Access. Watch history is intentionally excluded,
  but these are service behaviors needed to recreate a comparable fresh server.
- For a fresh bridge-networked server, obtain the short-lived `PLEX_CLAIM`
  immediately before applying its Secret/Deployment; LinuxServer states claim
  tokens expire in four minutes. Treat the token as a Secret and remove/rotate
  it from the workload after the server is claimed.
- Add and test `http://plex.home.arpa` as a Plex **Custom server access URL** if
  Ingress is the intended client endpoint. Test the actual Plex Web UI and at
  least one representative native LAN client. Do not stop old Plex until those
  clients can discover and play from the new server.
- If Remote Access, DLNA, GDM discovery, or direct `:32400` client access is a
  requirement, ClusterIP plus HTTP Ingress alone may not preserve it. The
  current Compose only publishes 32400, but the user's actual client/remote
  behavior is not documented. Decide and test this before finalizing exposure;
  Plex documents TCP 32400 as the fixed internal port for manual remote
  forwarding. [Plex remote-access troubleshooting](https://support.plex.tv/articles/200931138-troubleshooting-remote-access/)

### Homepage

A narrow, read-only inspection on 2026-07-10 found the standard Homepage files
(`bookmarks.yaml`, `custom.css`, `custom.js`, `docker.yaml`, `kubernetes.yaml`,
`services.yaml`, `settings.yaml`, and `widgets.yaml`), five explicit widget
blocks, and no `server:` or `container:` Docker-integration references. The
Compose services also have no Homepage discovery labels. This strongly
suggests that the mounted Docker socket is currently unused, but the full
non-secret dashboard definitions and widget credential requirements still
need to be recorded before recreation.

- Before stopping old Homepage, inspect and manually record its actual
  `services.yaml`, widgets, bookmarks, settings, custom assets, and Docker
  discovery/status behavior. The source-controlled replacement does not yet
  exist in this repository, so equivalent behavior cannot be inferred from the
  Compose mount alone.
- Store non-secret Homepage YAML in ConfigMaps/source control. Store widget API
  keys in Secret-backed `HOMEPAGE_VAR_*` or `HOMEPAGE_FILE_*` values and use
  Homepage's documented placeholders in the YAML; never commit the restored
  app credentials. [Homepage environment secrets](https://gethomepage.dev/installation/docker/#using-environment-secrets)
- Do not mount a Docker or containerd socket. If Kubernetes discovery/status is
  wanted, use Homepage's `mode: cluster`, a dedicated ServiceAccount, and a
  narrowed read-only Role/ClusterRole. The official example reads namespaces,
  Pods, nodes, ingresses, routes, gateways, and metrics; omit any resources the
  recreated dashboard does not consume. If only explicit service widgets are
  required, do not grant cluster discovery RBAC at all.
- Set `HOMEPAGE_ALLOWED_HOSTS=homepage.home.arpa,<pod-ip>:3000` in the form
  needed by its probe, use `/api/healthcheck`, and verify every link/widget
  against the final `home.arpa` names. Homepage has no built-in authentication;
  LAN-only HTTP is therefore an explicit trust decision, not app-native auth.
  Its project requires an authenticating reverse proxy/VPN for untrusted
  exposure. [Homepage security notice](https://github.com/gethomepage/homepage#security-notice-)

## Backup, rollout, and image-update rules

- Every stateful step must name the backup ZIP and record that it was
  downloaded before the old container is stopped. Keep the old container and
  its volume intact but stopped through validation. Never copy live SQLite
  files; Servarr's file-level procedure explicitly requires stopping the app
  to prevent corruption. [Radarr backup/restore](https://wiki.servarr.com/radarr/faq#how-do-i-backuprestore-radarr)
- Use immutable image digests in the rendered Pod spec. Kubernetes notes that a
  digest uniquely identifies an image version even when a tag moves.
  [Kubernetes images](https://kubernetes.io/docs/concepts/containers/images/#image-names)
- Replace Watchtower initially with a manual, one-app-at-a-time procedure:
  application backup, update reviewed digest, apply, wait for rollout, inspect
  events/logs, run the app smoke test, and retain the previous digest and
  backup. LinuxServer explicitly does not recommend unattended container
  updates and recommends update notification instead.
  [LinuxServer Radarr updating guidance](https://docs.linuxserver.io/images/docker-radarr/#updating-info)
- `kubectl rollout undo` reverts a Deployment Pod template; it does not revert
  a migrated SQLite database or media changes. Application backup/restore is
  the data rollback. [Kubernetes Deployment rollback](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment)

## Critical uncertainties that must be answered before final manifests

These are not safe to guess from the repository or official documentation:

1. **UniFi route:** exact gateway model, UniFi OS/Network versions, VLAN 30
   Policy-Based Route selector, VPN client interface, Kill Switch state, and
   whether a real Pod's egress is observed as the VM/VLAN source.
2. **NFS:** installed UNAS Drive version and share mode; successful mount of
   the exact parent export from the VM using NFSv3; whether the parent export
   permits the common `/data` layout; observed squash UID/GID, modes, rename,
   delete, and cross-directory hardlink results.
3. **LinuxServer identity:** whether each selected digest initializes correctly
   with PUID/PGID under K3s without a forced `runAsUser`; observed application
   UID/GID and file modes; chosen shared umask.
4. **Transmission:** the core effective paths, enablement flags, RPC settings,
   peer port, and umask are now recorded above. Still determine the remaining
   queue, ratio/seeding, encryption, speed, blocklist and proxy behavior;
   whether the internal port-8888 proxy is used; whether incoming peer ports
   are intentionally absent; and obtain explicit owner confirmation that the
   active queue is drained.
5. **Servarr/Prowlarr:** exact running versions and a stable target version not
   older than each backup's database; current Prowlarr application integrations
   and sync mode; successful backup ZIP downloads.
6. **Plex:** current library definitions and the clients/features that must
   continue working—web only, native LAN discovery, direct 32400, DLNA, or
   Remote Access. This determines whether Ingress-only exposure is sufficient.
7. **Homepage:** the narrow inventory found five widgets and no Docker
   integration references. Record the actual non-secret dashboard definitions,
   widget targets, and credential needs; confirm that omission of Docker and
   Kubernetes discovery/status is acceptable. This determines whether any
   Kubernetes RBAC is needed at all.
8. **Ingress/DNS:** Traefik currently advertises `192.168.30.103`, but LAN
   client reachability, exact DNS record names (including the chosen Homepage
   name), and whether all intended client VLANs use the UniFi gateway resolver
   remain unverified.

Final step files should encode each item as either an autonomous read-only
check or an explicit owner confirmation gate. No application cutover should
continue past a failed foundation, backup, VPN, storage, or client-behavior
gate.
