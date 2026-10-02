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
- The cluster uses Flannel VXLAN with Pod CIDR `10.42.0.0/24`.
- The VM has no `/dev/dri/renderD*` device; hardware transcoding is not
  available.
- Credentials are kept in untracked Kubernetes Secrets or the owner’s password
  manager; version-controlled manifests contain no credential values.
- `ns-media` is the manifest root for the Kubernetes `media` namespace.
- Ubuntu Server VM has package nfs-common installed.
- K3s secrets config in `node-main/secrets-encryption/k3s-server-config.yaml` is applied.
- The owner keeps the source YAML files on the dev laptop, outside the K3s VM.
- The owner's intended baseline for application data recovery is Proxmox VM
  backups.
