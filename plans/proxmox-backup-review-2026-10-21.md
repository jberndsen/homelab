# Proxmox backup review — 2026-10-21

Read-only inspection of the owner's logged-in Proxmox browser tab, supplemented
by repository records and `kubectl get nodes` / `kubectl get pv`. No Proxmox or
guest settings changed; no backup, restore, notification test, or SMART test ran.

## Assessment

VM 103 has a credible whole-VM recovery source for loss of its VM disk or host,
provided the NAS and required infrastructure survive. Successful creation,
retained archives, readable embedded configuration, matching disk coverage and
guest-agent freeze/thaw support this judgment. They do not prove that every
archive block is readable, that the restored guest boots, or that every
application/database recovers correctly. No restore has been tested.

Full service/disaster recovery is conditional on separate NAS media recovery,
host/network reconstruction and independent copies of backups and keys.

## Observed evidence (point-in-time, not a system inventory)

- UI reported Proxmox VE 9.2.21 at inspection. Verify the intended installation
  version and archive compatibility before a host rebuild; this dated report
  is not an installation pin.
- Enabled datacenter job: all guests/all nodes, daily 02:00, `unas-pro`,
  `keep-last=7`, snapshot mode, zstd, notification system, fleecing disabled.
  This is seven retained backups, not a time-based seven-day expiration policy.
- `unas-pro` is NFS storage mounted at `/mnt/pve/unas-pro`, separate from the
  local LVM-thin VM disks. It is not Proxmox Backup Server storage.
- Latest job: October 21, 02:00:00–02:08:22, completed successfully with two
  warnings. Recent task rows back through October 13 also show two warnings;
  only October 21's detailed output was inspected.
- VMs 100 and 101 were stopped. Their SCSI and EFI disks were included and
  their backups finished. Each produced one EFI certificate warning.
- VM 102: running snapshot backup, 32 GiB SCSI disk plus 4 MiB EFI disk,
  guest-agent freeze/thaw, completed at 02:03:16 without a warning in its section.
- VM 103: running snapshot backup, 02:03:16–02:08:18. Included
  `local-lvm:vm-103-disk-1` (scsi0, 200 GiB) and `vm-103-disk-0` (EFI, 4 MiB).
  Freeze and thaw were issued with no reported failures. Backup reached 100%,
  archive was finalized and the oldest backup pruned.
- VM 103's NAS listing contains October 15–21 archives, approximately
  32.17–32.82 GB each. Latest:
  `vzdump-qemu-103-2026_10_21-02_03_16.vma.zst`, 32.72 GB in the UI.
  The log's 30.47 GB figure is consistent with binary GiB versus decimal GB.
- Show Configuration successfully read the latest archive's embedded config.
  It matches live hardware: 8 cores, 16 GiB RAM, OVMF, x86-64-v2-AES CPU,
  virtio-scsi-single, scsi0 + EFI, net0 on vmbr0, guest agent and onboot enabled.
  EFI has `ms-cert=2023k`. No additional disks, passthrough devices or virtiofs
  mappings were shown. No external ISO is required (CD-ROM is empty).
- VM 103 also has an October 8 local backup, labelled pre-Argo; a backup on
  the Proxmox host is not independent protection against losing that host.
- Kubernetes PVs put application configuration and monitoring data under
  `/var/lib/rancher/k3s/storage` on the VM. The media PV is external NFS at
  `192.168.1.32:/var/nfs/shared/media`. Guest mount topology was not inspected;
  the expected local data coverage is supported, not independently proven at
  the filesystem level.
- Log says snapshots exist but are not included (VMs 100, 101, 103). These
  archives recover the backed-up current disk state, not the historical
  Proxmox snapshot tree or a running RAM session.
- Log says notification sent via `mail-to-root`; inbox delivery and a working
  external alert path were not verified.
- Capacity during inspection: local about 37%, local-lvm about 12%, NAS about
  50%. No obvious storage-capacity emergency. VM memory showed about 93–94%;
  that alone does not prove memory pressure and can include guest cache.
- Physical disk SMART, thin-pool metadata, repositories/available updates,
  notification routing, NFS export details and host backup were not validated.
  Browser control became unreliable while opening the disk page, so exploration
  stopped after the core backup evidence had been collected.

## Warnings and proposed resolution (not executed)

Both warnings concern missing `ms-cert=2023k` markers on the EFI disks of
VMs 100 and 101. They warn about the transition from Microsoft 2011 Secure Boot
certificates to the 2023 certificates. New bootloaders signed by the new CA can
be rejected if the EFI trust store lacks that CA. This is a boot compatibility
maintenance issue, not evidence of failed archive creation. VM 103 already has
the full updated marker in both live and archived configuration.

For each affected VM, retain a backup, select Hardware → EFI Disk → Disk Action
→ Enroll Updated Certificates, then validate its next boot and take a fresh
backup. The CLI alternative is `qm enroll-efi-keys 100` / `101` while shut down.
Do not simply edit the marker or disable Secure Boot. If either guest actually
uses Windows BitLocker, follow the documented BitLocker precautions first.
Older archives preserve the older EFI state and can need enrollment on restore.

## Prioritized follow-ups

1. Confirm an independent, preferably off-site or offline/immutable copy of VM
   archives and encrypted keys, and the separate media-backup source. These
   facts remain unknown; do not assume that NAS RAID or same-NAS snapshots
   protect against NAS loss. Record the owner's confirmed recovery sources in
   the system record when known.
2. Perform full archive integrity validation on an independent copy (zstd
   stream test plus VMA verification), with sufficient temporary space. Reading
   embedded configuration is not full payload verification. PBS with scheduled
   verification is a possible later improvement, not a prerequisite to repair
   these two warnings.
3. Add application-native backups for critical databases and the appropriate
   K3s datastore recovery method. Filesystem freeze improves consistency but is
   not evidence that applications were explicitly quiesced. Do not assume an
   etcd snapshot procedure without checking which datastore is actually used.
4. Verify that warning/failure notifications actually reach the owner. No test
   message was sent in this read-only review.
5. Consider weekly/monthly retention in addition to seven recent backups, for
   corruption or accidental deletion discovered more than a week later.
6. Preserve a secure host recovery bundle/runbook: Proxmox configuration,
   network bridge/VLAN setup, storage definitions, backup jobs, firewall/access,
   UPS configuration and necessary credentials outside public Git. VM archives
   are not a bare-metal backup of the Proxmox host. Replacement hardware must
   support the selected CPU model and provide suitable storage/networking.
7. Resolve the EFI warnings on 100/101. Review whether those stopped legacy VMs
   need nightly backups; preserve any required archival copies before changing
   retention or retiring them.
8. Eventually restore an isolated VM with production network access blocked,
   avoiding duplicate IPs, NAS writes and automatic Argo reconciliation. Check
   boot, K3s state, Secrets, PVCs and application behavior before claiming a
   tested recovery. This was explicitly outside today's scope.

The gateway is not UPS-powered, even though VM/NAS traffic crosses it. The NUT
stale-CAL defect was subsequently fixed on November 5 with an official Debian
package; see [NUT operations](../infrastructure/node-main/nut/README.md).
Communication-loss cause and automatic physical restart remain separate
[TODOs](TODO.md), not evidence that these archives are corrupt or restore-tested.

## References

- [Proxmox backup documentation](https://github.com/proxmox/pve-docs/blob/master/vzdump.adoc)
- [Proxmox Secure Boot certificate guidance](https://github.com/proxmox/pve-docs/blob/master/qm.adoc#secure-boot-certificate-expiration)
- [Existing VM and guarded recovery procedures](../infrastructure/node-main/ns-argo/SETUP.md#ordinary-vm-restore)
