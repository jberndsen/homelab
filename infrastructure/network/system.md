# Network configuration

This file holds details for the UniFi network configuration.

- UniFi gateway: Cloud Gateway Max.
- Observed on 2026-07-10: UniFi OS `5.1.19` and Network `10.4.57`.
- VLAN ID `30` is routed through the configured UniFi VPN client using NordVPN's NordLynx client.
- The policy-based route source selector is `192.168.30.0/24` and its Kill Switch is enabled.
- VLAN `30` must fail closed for internet egress if the NordLynx client is unavailable, while retaining explicitly required LAN, NAS NFS, DNS, and administration paths.
- Owner decision on 2026-07-28: UniFi Network owns and manages this
  fail-closed behavior; it is accepted as an external network-service
  dependency and is out of scope for K3s migration validation. A K3s control
  Pod used the NordVPN egress address `86.104.22.245` while the client was
  enabled. Manually disabling the client produced normal-WAN egress
  `31.201.5.179`; that action may deactivate the route itself, so it is not
  treated as a valid Kill Switch failure test and does not establish its
  behavior.
- For the initial migration, every K3s workload intentionally uses this VLAN-wide VPN route. UniFi cannot distinguish ordinary Flannel Pod egress because it is masqueraded through the K3s VM. A dedicated Transmission network identity is deferred until after functional parity is established.
- Read-only observation on 2026-07-10: the K3s VM and an existing Flannel Pod both reported public IPv4 `86.104.22.245`, confirming that ordinary Pod egress currently follows the VM path.
- Application UI clients on the Default and Servers VLANs use the UniFi Gateway for DNS and must be able to reach Traefik at `192.168.30.103` on TCP 80.
- Observed on 2026-07-28: Traefik LAN ingress was verified end to end from a
  client using the UniFi Gateway DNS resolver. The temporary Host (A) record
  `ingress-test.home.arpa` resolved to `192.168.30.103`; an HTTP request
  returned the expected test response with that Host header. Application DNS
  records under `home.arpa` may therefore use `192.168.30.103` as their
  Traefik ingress target.
- Owner configured the UniFi Gateway wildcard Host (A) record
  `*.home.arpa` to `192.168.30.103` on 2026-07-28. The owner confirmed it by
  opening `http://ingress-test.home.arpa`, which reached the temporary Traefik
  ingress test successfully.

## Public DNS and VPN DDNS

- The `notech.foo` domain is registered and managed in Cloudflare.
- Its current public DNS record is the `A` record `vpn.no.tech.foo`, used by the UniFi Cloud Gateway VPN server.
- The UniFi Cloud Gateway manages DDNS for that record, updating Cloudflare whenever the home's public IP address changes.

## NAS

- UniFi UNAS Pro at `192.168.1.32`.
- `node-main` (Proxmox) and `node-secondary` can reach the NAS.
- Main shares:
  - `media` for the media stack.
  - `photos` for a future Immich deployment.
  - `proxmox` for Proxmox ISO images and backups.
- The NAS grants NFS read-write access to `192.168.30.103` for `media` and `photos`.
- The `media` NFS export is `192.168.1.32:/var/nfs/shared/media`.
- UniFi Drive version is `4.3.6`. The `media` share's squash mode could not be confirmed; migration must rely on an in-Pod behavioral permission and hardlink test and must not change the share mode speculatively.
- `showmount -e` from `node-secondary` on 2026-07-10 advertised the underlying `media` export to both `192.168.30.103` and `192.168.1.33`. Existing mounts continue to use the stable client path above.
- Relevant paths beneath that export are:
  - `Downloads`
  - `Video/Movies`
  - `Video/TV Shows`
  - `Music/Lossless`
- NFS shares are not yet mounted on the Ubuntu VM or configured for use by K3s.
- The K3s VM can reach the NAS NFS service at `192.168.1.32:2049`.
- Existing media mounts on `node-secondary` use NFSv3.
- Observed on 2026-07-28: `nfs-common` is installed on the K3s VM, and a
  temporary read-only mount of `192.168.1.32:/var/nfs/shared/media` with
  `nfsvers=3` succeeded.
- Observed on 2026-07-28: that export reports `8.2T` total, `3.5T` used, and
  `4.8T` available.
- Observed on 2026-07-28: a K3s Pod running as UID `1000` and GID `988`
  successfully listed/traversed, created, wrote, fsynced, renamed, deleted,
  and hardlinked disposable files across `Downloads`, `Video/Movies`,
  `Video/TV Shows`, and `Music/Lossless`. A final read-only scan confirmed no
  test files remained. The existing directories reported owner UID `977`, GID
  `988`, and mode `770`; this behavioral test does not establish the NAS
  squash mapping.
- The NAS remains the authoritative home for all media. K3s workloads will mount the existing media directories directly over NFS; media data will not be copied during migration.
