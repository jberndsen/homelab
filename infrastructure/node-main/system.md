# Current state

- `node-main` runs Proxmox VE 9.2 on VLAN `30` at `192.168.30.5`.
- Its Ubuntu Server VM is `192.168.30.103`; it runs K3s `v1.36.2+k3s1` and is
  accessible over SSH.
- K3s Secrets encryption at rest is enabled.
- Traefik is installed by K3s and ServiceLB exposes HTTP/HTTPS at
  `192.168.30.103`. Application UIs use ClusterIP Services and Traefik
  Ingresses; no application has a direct LAN port.
- The cluster uses Flannel VXLAN with Pod CIDR `10.42.0.0/24`. K3s workload
  internet egress follows VLAN 30's UniFi NordVPN policy.
- Internal application DNS uses the UniFi `*.home.arpa` wildcard record,
  which resolves to `192.168.30.103`. HTTP is intentionally used on the
  isolated LAN; TLS and external authentication are not configured.
- K3s media workloads use the `media` namespace. Application state is
  node-local `local-path` storage; shared media is the NAS NFS export. The
  NAS behavior needed by writers has been validated with UID `1000` and GID
  `988`.
- Jellyfin is the accepted active K3s media server. Direct play works through
  `http://jellyfin.home.arpa` from the laptop on `192.168.1.118/24` and from
  the NVIDIA Shield Android client. The owner accepts a direct-play-only media
  policy: transcoding capacity is neither required nor assumed, and media that
  requires transcoding must be manually re-downloaded in a directly playable
  format. The legacy Plex Compose service was stopped on 2026-08-04 and is
  retained intact as a rollback path.
- Bazarr is accepted as complete for this migration and is deployed in the
  `media` namespace at `http://bazarr.home.arpa`,
  using LinuxServer `v1.6.0-ls356` pinned to manifest digest
  `sha256:ab401a0f361cfad328e444838b13d5b334b189d0f556fc91a3623eb581df36df`.
  It has healthy local `bazarr-config` storage, a ClusterIP Service, and a
  Traefik Ingress. Bazarr uses an English-only profile with embedded-subtitle
  detection, connects to the K3s Radarr, Sonarr, and Jellyfin Services, and
  uses OpenSubtitles.com as its sole provider. Manual movie and episode
  subtitle downloads and embedded-subtitle detection passed. The initial
  existing-library search downloaded 21 subtitles before the OpenSubtitles.com
  daily limit was reached; the provider is temporarily throttled. Jellyfin's
  connection test passed, while an automatic metadata-refresh after a subtitle
  download has not yet been observed. On 2026-08-04, the owner accepted both
  as non-blocking operational follow-up items; create separate plans if either
  causes a practical problem.
- Homepage is deployed to K3s in the `media` namespace at
  `http://homepage.home.arpa`, with a ClusterIP Service and Traefik Ingress.
  On 2026-08-04, its container was corrected to provide a writable ephemeral
  `/app/config` directory for Homepage-created optional configuration files;
  the source-controlled configuration remains read-only overlays. The owner
  then visually confirmed the intended K3s Jellyfin, Sonarr, Radarr, and
  Prowlarr panes and widgets work. The legacy Homepage container was stopped
  on 2026-08-04 and remains intact as the rollback path.
- The VM has no `/dev/dri/renderD*` device. Jellyfin has no GPU device access
  and hardware transcoding is not configured.
- Credentials are kept in untracked Kubernetes Secrets or the owner’s password
  manager; version-controlled manifests contain no credential values.
- Final migration acceptance was completed on 2026-08-04. The K3s node was
  `Ready`; Transmission, Prowlarr, Radarr, Sonarr, Jellyfin, Homepage, and
  Bazarr were each `1/1` available with bound local PVCs; `media-nfs-pv` was
  Bound with `Retain`; and every application hostname responded through
  Traefik. The owner confirmed the UIs from both Default and Servers VLAN
  clients, Prowlarr indexers and Radarr/Sonarr integrations, the prior
  Radarr/Sonarr hardlink-import checks, and all Homepage links/widgets. The VM
  root filesystem had 77 GiB available (`97G` total, `16G` used).
- During that acceptance check, the Transmission Pod's observed public egress
  IP was `86.104.22.245`. A deliberate UniFi NordLynx-outage fail-closed test
  was not repeated. The owner accepted this risk, relying on UniFi's enabled
  VLAN 30 NordLynx Kill Switch; testing from another VLAN produced the
  expected non-VPN public IP. A shortly preceding Homepage Pod restart
  produced a BackOff event, but Kubernetes replaced it, the current Pod was
  healthy, and the owner reconfirmed the Homepage UI and widgets. The prior
  Pod's failure logs were no longer available.
