# Node and storage facts

## Host and network

- `node-main` runs Proxmox VE 9.2.4 on VLAN `30` at `192.168.30.5`.
- `vmbr0` bridges `nic0` and uses gateway `192.168.30.1`. Its tagged VLAN `1`
  interface `vmbr0.1` has `192.168.1.39/24` and a direct route to the UPS at
  `192.168.1.34`; both interfaces are in `/etc/network/interfaces`.
- Ubuntu Server VM `103` (`vm-ubuntu-k3s`) is `192.168.30.103`, runs K3s
  `v1.36.2+k3s1` and is accessible over SSH. It has 8 vCPUs and approximately
  16 GiB RAM. Its only virtual NIC is attached to `vmbr0`.
- The guest's default route and route to the NAS (`192.168.1.32`) use `ens18`
  via `192.168.30.1`, not host interface `vmbr0.1`.
- K3s uses Flannel VXLAN with Pod CIDR `10.42.0.0/24`. Its Traefik image is
  `rancher/mirrored-library-traefik:3.7.4`; ServiceLB exposes HTTP/HTTPS at
  `192.168.30.103`.
- The VM has `nfs-common` installed and no `/dev/dri/renderD*` device;
  hardware transcoding is unavailable.
- K3s uses the configuration in
  [secrets-encryption/k3s-server-config.yaml](secrets-encryption/k3s-server-config.yaml).

## UPS

- Proxmox is the only physical UPS shutdown target. VM `102` (`vm-haos`) and
  VM `103` have working QEMU guest agents and Start at boot enabled.
  Validate guest shutdown behavior before relying on unattended outage recovery.
- NUT `nut-client` uses `MODE=netclient` and a `secondary` monitor for
  `ups@192.168.1.34:3493`, username `ups`, with TLS. The password is in
  `/etc/nut/upsmon.conf` and the owner's password manager.
- `nut-monitor.service` is enabled; the UPS certificate is not verified.
  See [infrastructure recovery](../../setup.MD#ups-shutdown-on-proxmox) for setup
  and connection checks.

## Git and ownership

- Public source: `https://github.com/jberndsen/homelab`, default branch `main`.
  The laptop's `origin` uses SSH; Argo reads
  `https://github.com/jberndsen/homelab.git` anonymously over HTTPS.
- Source files and the configured `kubectl` client live on the dev laptop,
  outside the K3s VM. Plaintext credentials and private keys stay outside Git.
- The hosted Renovate GitHub app is enabled. Repository configuration in
  `../../renovate.json` scans node-main Kubernetes manifests, excludes the
  inactive node-secondary stack, and keeps container image PRs ungrouped with
  automerge disabled. Digest-only images track the registry's `latest` digest;
  review releases and take application-native backups before merging updates.
- `cluster/` manually manages bootstrap namespaces and the NAS PV.
  `ns-argo/` manually manages Argo CD and its separately applied ApplicationSet;
  Argo does not manage itself.
- Argo uses the standard non-HA install pinned in `ns-argo/kustomization.yaml`,
  retaining upstream NetworkPolicies. Its ApplicationSet Deployment and
  application-controller StatefulSet each use one replica.
- `http://argocd.home.arpa` routes through Traefik to Argo's HTTP server with
  `server.insecure: "true"`. Restrict access to the trusted LAN. The `admin`
  password is in the owner's password manager; the initial-password Secret is
  absent.
- ApplicationSet `node-main` discovers `ns-*/overlays/prod`, excluding Argo.
  Applications are `media` → `media` and `sealed-secrets` → `kube-system`.
  Automatic sync, pruning and self-healing are enabled with `allowEmpty: false`.
  Resource preservation is enabled and Applications have no deletion finalizers.

## Media and Secrets

- `ns-media` defines eight Deployments and nine PVCs in namespace `media`.
  Eight local-path PVs use reclaim policy `Delete`; shared `media-nfs-pv`
  uses `Retain`. Preserve these policies and existing bindings.
- All media PVCs, both SealedSecrets and the Sealed Secrets CRD have
  `Prune=confirm,Delete=confirm`. Direct Kubernetes deletion bypasses these gates.
- Strict-scope SealedSecrets manage `homepage-widgets` and `transmission-rpc`.
  Their generated Secrets hold the plaintext credentials consumed by workloads.
- The Sealed Secrets controller version is pinned in
  `ns-sealed-secrets/base/kustomization.yaml` and uses default 30-day key renewal.
  Laptop tools are `kubeseal v0.40.0` and `age v1.3.2`.

## Recovery sources

- Application data recovery uses Proxmox backups of VM `103` with every
  application PVC disk included. External NAS media requires its own backup.
- Encrypted sealing-key backups and their recovery `README.md` live at
  `smb://192.168.1.32/backups/NUC/kubernetes`, mounted on the dev laptop at
  `/Volumes/backups/NUC/kubernetes`.
- The backup passphrase is in the owner's password manager under
  `Homelab Sealed Secrets key backup`. Keep all keys required by existing
  ciphertext; verify backup coverage before sealing with a renewed key.
- Before recovery, confirm backup freshness, disk/key coverage, restore method
  and independent copies for host or NAS loss. Git contains configuration,
  not application data. Fresh-cluster data restoration requires an explicit,
  validated storage mapping and restore procedure.
- Follow [Argo operations](ns-argo/SETUP.md) for bootstrap, ordinary or guarded
  VM restore, key handling and application checks.
