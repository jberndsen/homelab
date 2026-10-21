# Node and storage facts

## Host and network

- `node-main` runs Proxmox VE 9.2.4 on VLAN `30` at `192.168.30.5`.
- The owner confirmed on 2026-10-12 that the physical host has 32 GB RAM,
  with 16 GB allocated to Ubuntu/K3s VM `103` and 4 GB to HAOS VM `102`.
- `vmbr0` bridges `nic0` and uses gateway `192.168.30.1`. Its tagged VLAN `1`
  interface `vmbr0.1` has `192.168.1.39/24` and a direct route to the UPS at
  `192.168.1.34`; both interfaces are in `/etc/network/interfaces`.
- Ubuntu Server VM `103` (`vm-ubuntu-k3s`) is `192.168.30.103`, runs K3s
  `v1.36.2+k3s1` and is accessible over SSH. It has 8 vCPUs and approximately
  16 GiB RAM. Its only virtual NIC is attached to `vmbr0`.
- The guest's default route and route to the NAS (`192.168.1.32`) use `ens18`
  via `192.168.30.1`, not host interface `vmbr0.1`.
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
- Known unresolved risk: the distribution NUT client can shut down early after
  communication loss with stale calibration state. Shutdown protection remains
  enabled. The owner wants an official compatible Debian fix, not a locally
  maintained package or unstable dependencies. Communication-loss cause and
  automatic physical restart after power restoration remain unverified.
  See [the NUT investigation and future update plan](../../plans/nut-update.md).

## External services and access

- Source files and the configured `kubectl` client live on the dev laptop,
  outside the K3s VM. Plaintext credentials and private keys stay outside Git.
- The hosted Renovate GitHub app is enabled. Its repository settings are in
  [renovate.json](../../renovate.json); image updates remain manually reviewed.
- The Argo `admin` password is in the owner's password manager.
- Consult [Argo operations](ns-argo/SETUP.md) for bootstrap ownership,
  reconciliation and credential lifecycle, and
  [monitoring operations](ns-monitoring/README.md) for monitoring access.

## Recovery sources

- Application data recovery uses Proxmox backups of VM `103` with every
  application PVC disk included. External NAS media requires its own backup.
- The owner confirmed on 2026-10-21 that the entire VM is backed up nightly,
  with seven days of retention. These backups have not been restore-tested.
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
