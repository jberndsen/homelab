# Current state

- Runs Proxmox VE 9.2.
- VLAN ID: `30`.
- IP address: `192.168.30.5`.
- Ubuntu Server VM: `192.168.30.103` (SSH accessible).
- K3s `v1.36.2+k3s1` runs on the Ubuntu Server VM; its control-plane node is `Ready`.
- Observed on 2026-07-28: K3s Secrets encryption at rest is enabled. Its final
  status reported `Current Rotation Stage: reencrypt_finished`, `Server
  Encryption Hashes: All hashes match`, and an active AES-CBC key. The K3s
  server configuration at `/etc/rancher/k3s/config.yaml` contains
  `secrets-encryption: true`.

## K3s operations

- Use the local `kubectl` installation with its configured kubeconfig to work with the cluster.
- Confirm cluster connectivity and node readiness with `kubectl get nodes -o wide`.
- Application UIs will use Pod → `ClusterIP` Service → Traefik Ingress, with one internal DNS hostname per application; direct per-application LAN ports will not be used.
- Traefik `3.7.4` is installed through packaged K3s Helm charts. ServiceLB currently advertises `192.168.30.103` on TCP 80 and 443.
- The cluster uses Flannel VXLAN with Pod CIDR `10.42.0.0/24`. Ordinary Pod and VM egress currently report the same public IPv4 and are intentionally routed through VLAN 30's NordVPN policy for this migration.
- The initial LAN DNS convention is one UniFi Gateway local A record per application under `home.arpa`, all resolving to the Traefik ingress IP.
- Initial internal Traefik Ingresses will use plain HTTP and application-native authentication. TLS/cert-manager is deferred until after the initial migration; the LAN is isolated.
- Media workload Pods will run as UID `1000` and GID `988`, matching the existing Compose workloads. NFS behavior will be validated with this identity before any migration.
- Initial workload manifests will be version-controlled Kustomize configuration applied manually with `kubectl` and pinned image digests. Argo CD/app-of-apps will be considered only after the initial migration and recovery path are proven.
- Credentials will initially be supplied in simple, untracked `secrets.yaml` files; version-controlled manifests reference only their Secret names. More complex secret management is deferred until after the baseline migration.
- During the migration, app-native backup exports will be retained as untracked operational data in this workspace on the laptop; SSH access is available for transfers to or from the homelab nodes.
- The initial media stack will use the `media` Kubernetes namespace.
- Each stateful media application will have a dedicated node-local `local-path` PVC for `/config` (including Plex metadata); NAS storage is reserved for shared media directories.
- Observed on 2026-07-10: the VM root filesystem has 97 GiB total, 8.0 GiB used, and 84 GiB available. `/var/lib/rancher/k3s/storage` does not yet exist because no local-path PVC has been provisioned.
- Initial local-state storage budget: 30 GiB for Plex metadata and approximately 20 GiB total for the other stateful applications. This is a planning budget, not a hard quota; monitor actual VM disk usage.
- Observed on 2026-07-10: the VM has 8 CPU cores, 15 GiB RAM (about 14 GiB available at observation), and 4 GiB unused swap.
- Observed on 2026-07-10: the VM exposes `/dev/dri/card0` (group `video`) but no `/dev/dri/renderD*` node.
- Initial Plex migration will rely on direct play and software transcoding only. Hardware transcoding is deferred; it is not expected to be available and no Plex Pass subscription is present.
- Plex is used through both browser and native clients. Remote Access and DLNA are not required. The initial replacement will be LAN-only through `http://plex.home.arpa` and Traefik, will not publish direct port `32400`, and must remain unpublished for remote access.
- Homepage will retain its five explicit Plex, Sonarr, Radarr, Lidarr, and Prowlarr widgets. Docker/Kubernetes discovery and container-status integration are not required, so Homepage receives no Kubernetes API RBAC and no runtime socket.
