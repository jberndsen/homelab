# Current state

- `node-main` runs Proxmox VE 9.2 on VLAN `30` at `192.168.30.5`.
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
- K3s secrets config in `node-main/secrets-encryption/k3s-server-config.yaml` is applied.
- The owner keeps the source YAML files on the dev laptop, outside the K3s VM.
- As of 2026-10-06, the repository is also published at
  `https://github.com/jberndsen/homelab`, with default branch `main`.
  The dev laptop's `origin` uses SSH; the public HTTPS clone URL is
  `https://github.com/jberndsen/homelab.git`.
- The owner's intended baseline for application data recovery is Proxmox VM
  backups.
- The owner selected `smb://192.168.1.32/backups/NUC/kubernetes` for encrypted
  Sealed Secrets key backups and an adjacent recovery `README.md`. On the dev
  laptop this is mounted at `/Volumes/backups/NUC/kubernetes`; SMB connectivity
  and directory access were verified on 2026-10-06. The backup passphrase will
  be stored in the owner's password manager. The key backup is not yet created.
- On 2026-10-08, Argo CD `v3.5.4` was manually bootstrapped in namespace
  `argocd` using the standard non-HA upstream install and preserved upstream
  NetworkPolicies. Its six Deployments and application-controller StatefulSet
  were ready, with seven Ready Pods and no restarts. Argo does not manage itself;
  its manifests are in `ns-argo/` outside production directory discovery.
- `http://argocd.home.arpa` returned HTTP 200 through Traefik at
  `192.168.30.103`; the server runs with `server.insecure: "true"`. Access is
  intended for the trusted LAN. Admin login/password rotation is awaiting owner
  verification; the initial-password Secret has not yet been removed.
- No Argo Applications or ApplicationSets have been created yet. Media remains
  manually managed. Sealed Secrets and its independent encrypted key backup
  remain pending; TODO 1–3 are not complete.
- Immediately after bootstrap, the node used approximately 127m CPU and
  3069 MiB memory (19%); Argo Pods collectively used approximately 168 MiB.
  These are initial idle observations, not workload sizing guarantees.
- On 2026-10-08, local and GitHub `main` matched `371650a` before implementation.
  A pattern audit of all 27 reachable commits (189 distinct file blobs) found
  placeholders, structural matches and documented fake node-secondary values;
  no actual credentials were identified by that audit. A credential-free
  pre-install image/storage baseline is held privately under ignored
  `plans/runtime/` on the dev laptop.
