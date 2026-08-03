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
- Jellyfin is the active K3s media server. Direct play works through
  `http://jellyfin.home.arpa` from the laptop on `192.168.1.118/24` and from
  the NVIDIA Shield Android client. Software-transcode capacity is
  unvalidated.
- Homepage is deployed to K3s in the `media` namespace at
  `http://homepage.home.arpa`, with a ClusterIP Service and Traefik Ingress.
  Its pod, Service, and ingress health checks pass, but browser/widget
  acceptance remains incomplete: the owner reported a `no available server`
  message. The legacy Homepage container remains the rollback path until this
  is diagnosed and the Jellyfin, Sonarr, Radarr, and Prowlarr widgets pass.
- The VM has no `/dev/dri/renderD*` device. Jellyfin has no GPU device access
  and hardware transcoding is not configured.
- Credentials are kept in untracked Kubernetes Secrets or the owner’s password
  manager; version-controlled manifests contain no credential values.
