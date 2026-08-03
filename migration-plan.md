# Docker Compose to K3s migration plan

## Purpose and scope

Move the media stack from Docker Compose on `node-secondary` to the existing
single-node K3s cluster on `node-main`, one application at a time. This is a
minimal first migration: add no GitOps controller, CSI driver, VPN sidecar,
TLS automation, external authentication proxy, or advanced secret-management
system until the complete migration and recovery path have been proven.

This plan is the agreed design. The detailed primary-source reasoning is in
[infrastructure/research/k3s-media-migration.md](infrastructure/research/k3s-media-migration.md).
Current infrastructure facts are the system-of-record files in
`infrastructure/`.

## Agreed operating model

- Use a `media` namespace.
- Keep step-numbered, multi-document deployment manifests in this workspace
  and apply them manually with `kubectl`, one migration step at a time.
- Pin every deployed image by digest. Argo CD and app-of-apps are deferred
  until the complete initial migration and recovery process work.
- Keep simple per-environment `secrets.yaml` files untracked. Versioned
  manifests refer only to Secret names.
- Retain manual app-native backup exports as untracked operational files in
  this workspace. SSH is available for transfers when needed.
- Migrate one application at a time. For every stateful application, take and
  download its app-native backup *before* stopping the old container. Stop
  the old container for the final restore and validation window. Keep it
  stopped and intact as the rollback path until the K3s replacement passes all
  checks.
- Never run old and new Transmission instances against the same active queue
  or media paths. Before its cutover, explicitly remind the owner to drain
  Transmission's active download queue.
- Do not copy Docker volumes or application configuration directories. Plex
  is a fresh server/library scan; watch history is intentionally not retained.

## Storage and identity design

- NAS media remains authoritative. Its NFSv3 export is
  `192.168.1.32:/var/nfs/shared/media`.
- Use retained, static NFS-backed Kubernetes storage. Do not introduce a CSI
  driver for this initial single-node deployment.
- Mount the NAS `media` parent export exactly once at the common in-container
  path `/data` in Transmission, Radarr, and Sonarr. Use its real,
  case-sensitive subdirectories:
  - `/data/Downloads`
  - `/data/Video/Movies`
  - `/data/Video/TV Shows`
- This common path is required for hardlink-safe imports. After each *arr
  backup restore, correct its root folders to the new `/data/...` locations;
  do not add a Remote Path Mapping when Transmission and the importer already
  see the same path.
- For LinuxServer images, retain `PUID=1000` and `PGID=988`, matching the known
  working Compose application identity. Do not force Pod-level `runAsUser` on
  these images because their init process normally starts as root and drops
  the application to PUID/PGID. The NAS may all-squash remote identities, so
  this is a testable convention, not proof of server-side ownership semantics.
- Keep every app's `/config` (including Plex metadata) on a separate K3s
  `local-path` PVC, never NFS. Plan 30 GiB for Plex metadata and about 20 GiB
  total for the other stateful apps; monitor the VM filesystem because this is
  not a hard quota.

## Network and exposure design

- VLAN 30 routes internet traffic through the UniFi NordVPN NordLynx client.
  Its internet egress must fail closed if that VPN client is unavailable,
  while retaining necessary LAN, NAS NFS, DNS, and administrative access.
- A successful VM route does not prove Pod routing. Transmission may not
  download until an actual Transmission Pod proves NordVPN egress and the
  intended fail-closed behavior.
- The initial design intentionally sends every K3s workload through this
  VLAN-wide route. A dedicated Transmission network identity is deferred.
- Expose each UI through Pod → `ClusterIP` Service → Traefik Ingress. Do not
  use direct per-application LAN ports.
- Use the UniFi Gateway local wildcard DNS A record `*.home.arpa`, pointing to
  the verified Traefik ingress IP. Traefik routes each application by its Host
  rule (for example, `radarr.home.arpa`).
- Initial ingress is plain HTTP on the isolated LAN with the application’s
  own authentication. TLS/cert-manager and external authentication are
  explicitly deferred.

## Foundation gates — complete these before any app migration

1. Verify the K3s node is Ready.
2. Confirm the Ubuntu VM has the NFS client support needed by K3s and that the
   static NFS claim can bind the `media` export.
3. Run a disposable NFS test Pod as UID:GID `1000:988`. Verify it can read,
   create, write, rename, delete, and hardlink a disposable test file in the
   required media directories. Remove the test data afterward. Stop and
   diagnose if any check fails; do not change NAS share mode casually.
4. Inspect the installed K3s Traefik/ServiceLB configuration, choose and
   verify the ingress LAN IP, then create the individual UniFi `home.arpa`
   records and test resolution from a LAN client.
5. Verify K3s Secrets encryption at rest before creating credentials.
6. Treat UniFi's fail-closed NordLynx routing as an external network-service
   responsibility. Record the exact route configuration and the owner's
   acceptance of that dependency in the network system record; it is not
   validated as a K3s migration gate.
7. Install `nfs-common` on the Ubuntu VM; it is currently absent.

## Application sequence

### 1. Transmission

1. Remind the owner to confirm the old queue is empty; do not proceed until
   confirmed.
2. Deploy a non-VPN Transmission image with a fresh local `/config` PVC and
   `/data/downloads` on NAS storage. Create RPC credentials in the untracked
   Secret file.
3. Initially prevent normal downloading.
4. From the real Pod, verify public egress is NordVPN and that the expected
   failure of the VPN route cannot fall back to the normal WAN.
5. Verify RPC authentication and a controlled, legal test download to the NAS.
6. Only after these checks, stop the old container and enable normal use.

### 2. Prowlarr

1. In the old UI, create a backup and download it to this workspace.
2. Stop the old container.
3. Deploy Prowlarr on a pinned **stable** release newer than the observed
   `2.3.1.5238` develop build, with a fresh local `/config` PVC.
4. Restore the backup through **System → Backup → Restore Backup**.
5. Test every restored indexer.
6. Add/test Prowlarr’s integrations only after the destination apps exist.

### 3. Radarr and Sonarr — one at a time

For each application:

1. Create/download its built-in backup while the Compose container is live.
2. Stop the Compose container.
3. Deploy a fresh K3s instance with local `/config`, the single shared `/data`
   parent mount, a
   ClusterIP Service, and its Ingress.
4. Restore through **System → Backup → Restore Backup**.
5. Update root folders to the `/data/media/...` convention.
6. Configure and test Transmission and Prowlarr connections.
7. Use a controlled completed download to validate import behavior. Confirm a
   hardlink rather than a copy when that is expected, then verify resulting
   ownership/mode and application health.
8. Keep the old Compose service stopped but recoverable until the new service
   has passed these checks.

### 4. Bazarr — new K3s service

1. Do not create a backup, stop a legacy container, or restore configuration:
   Bazarr is a fresh service.
2. Deploy it with a local `/config` PVC and the shared writable `/data` NFS
   mount, behind a ClusterIP Service and `bazarr.home.arpa` Traefik Ingress.
3. Create a fresh application administrator, connect only to the accepted
   K3s Radarr and Sonarr Services using their existing API keys, and configure
   OpenSubtitles.com as its single initial provider.
4. Apply the agreed English-only default profile to all existing and future
   Radarr/Sonarr items. Embedded English tracks satisfy the requirement, so
   external subtitles are downloaded only when needed.
5. Test one movie and one episode before enabling the automatic search for the
   full existing library. Keep generated subtitle files and the PVC when
   diagnosing or rolling back.

### 5. Plex

1. Do not export or copy Plex configuration, metadata, or watch history.
2. Deploy a fresh server with its own local metadata PVC and NAS media mounted
   read-only where possible.
3. Use direct play and software transcoding only. Hardware transcoding is
   deferred: the VM does not expose a render node and no Plex Pass is present.
4. Claim the new server, recreate movie and TV libraries using the `/data`
   paths, then perform a fresh scan.
5. Test playback from a LAN client before stopping the old Plex container.
6. Use LAN-only `http://plex.home.arpa` through Traefik for both browser and
   native-client validation. Remote Access, DLNA, and direct port `32400`
   publication remain disabled.

### 6. Homepage

1. Deploy only after application URLs and health checks are stable.
2. Recreate its configuration from source-controlled files; do not copy the
   Docker runtime config or mount the Docker socket.
3. Create least-privilege app credentials as needed for widgets, stored in
   the untracked Secret file.
4. Test every link and widget through the `home.arpa` names.

## Per-change validation and rollback

For every deployed application:

1. Check Pod readiness, Events, and logs.
2. Test the `ClusterIP` Service and the intended Ingress hostname from a LAN
   client.
3. Perform the app-native connection/health test.
4. For apps that touch media, test actual NAS access before accepting the
   cutover.
5. If a check fails, stop/scale down the new K3s workload and return to the
   still-intact old Compose service. Preserve logs and the manual backup.
6. Never delete NAS media or local configuration PVCs during diagnosis.

## Deferred work

- Argo CD and app-of-apps.
- Automated image updates / Watchtower replacement. Start with manually
  reviewed pinned-digest updates, app backup, rollout status, and smoke test.
- TLS, cert-manager, and external authentication.
- Advanced encrypted/sealed secret management.
- Dedicated DNS, wildcard records, or split-horizon `home.notech.foo`.
- Plex hardware transcoding and GPU passthrough.
- NetworkPolicy hardening after the current CNI enforcement behavior is
  verified.
