# Current state

- `node-main` runs Proxmox VE 9.2 on VLAN `30` at `192.168.30.5`.
- Its Ubuntu Server VM is `192.168.30.103`; it runs K3s `v1.36.2+k3s1` and is
  accessible over SSH.
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
