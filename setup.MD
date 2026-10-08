# Infrastructure recovery

Run Kubernetes commands from this repository's root with the laptop's configured
`kubectl`; start with `kubectl get nodes -o wide`. Read [node facts](infrastructure/node-main/system.md)
and [network facts](infrastructure/network/system.md) before recovery.

1. Restore the UniFi network, VPN routing, firewall access and local DNS from the network record.
2. Make the NAS media share available using [the storage manifest](infrastructure/node-main/cluster/nfs-storage/media-nfs-persistentvolume.yaml). Restore lost media from a separate NAS backup; VM backups do not contain it. Confirm the NAS backup source and restoration procedure before recovery.
3. Install Proxmox on `node-main` using the recorded version and network settings if the host was lost. Keep recovery backups outside the host as well.
4. Choose the recovery path **before starting the VM**. An [ordinary full-VM restore](infrastructure/node-main/ns-argo/SETUP.md#ordinary-vm-restore) may resume Argo immediately against newer Git. For a pause before reconciliation, use a backup prepared by the [guarded procedure](infrastructure/node-main/ns-argo/SETUP.md#guarded-proxmox-backup-and-restore); loss of Git/network access is not a reliable pause. Restore VM 103 with its application-data disks. Select a successful backup, verify availability and freshness, and validate its restore procedure.
5. If creating a new VM instead, use the recorded network/SSH settings, install `nfs-common` and the recorded K3s version with [the K3s configuration](infrastructure/node-main/secrets-encryption/k3s-server-config.yaml). Clone this repository on the laptop and configure its `kubectl` access. Confirm the intended data/storage restoration with the owner; a fresh cluster does not recover databases from Git.
6. For a fresh cluster, follow [Fresh cluster](infrastructure/node-main/ns-argo/SETUP.md#fresh-cluster): manually apply `cluster/`, bootstrap Argo, then apply the **disabled copy** of the ApplicationSet. The tracked template enables automation and must wait until recovery checks pass. Restore verified sealing keys using `/Volumes/backups/NUC/kubernetes/README.md`; sync only Sealed Secrets, restarting its controller if keys were imported after startup. Restore application data before starting media. Sync media's two SealedSecrets first and verify decryption, then review storage and manually sync workload groups. Tracked files render without ignored plaintext Secrets.
7. For existing-cluster adoption, follow the [Secret ownership and workload adoption checks](infrastructure/node-main/ns-argo/SETUP.md#sealed-secrets). Stop on unexpected storage differences; never delete/recreate storage or use force/replace. If application backups are unavailable, obtain an explicit owner decision before initializing empty data and configuring apps again.
8. Run the [read-only recovery and smoke checks](infrastructure/node-main/ns-argo/SETUP.md#recovery-checks), including app access, bindings, Homepage widgets and Jellyfin playback from NAS. Check VPN routing against the network record. The [infrastructure test manifests](infrastructure/node-main/tests) include NAS writes and are not read-only checks. Enable automation only after the recovered cluster's keys, data, storage and smoke checks pass and resuming is approved.
9. Verify continuing Proxmox backups with application PVC disk coverage, separate NAS-media and repository backups, and independent encrypted key recovery with its password-manager passphrase. Git configuration, VM application data, NAS media and sealing keys each need their own recovery source; backups on a lost host/NAS cannot recover that device.

## UPS shutdown on Proxmox

1. Connect the UniFi UPS 2U management port and Proxmox to the same
   UPS-powered switch. Keep the UPS on Default VLAN `1` at `192.168.1.34`.
   Set the Proxmox switch port to native Servers VLAN `30` with tagged VLAN
   `1` allowed. Enable the UPS NUT server with ID `ups`, TCP port `3493`, and
   username `ups`; store its password in the password manager.
2. Keep Proxmox `vmbr0` on `192.168.30.5/24`, bridged to `nic0`, with gateway
   `192.168.30.1`. Add this stanza to `/etc/network/interfaces` for direct
   UPS access, without another gateway:

   ```text
   auto vmbr0.1
   iface vmbr0.1 inet static
           address 192.168.1.39/24
   ```

   Run `ifup vmbr0.1` to activate it without restarting the existing bridge.

3. On Proxmox, run `apt-get install nut-client`. Set `MODE=netclient` in
   `/etc/nut/nut.conf`. In `/etc/nut/upsmon.conf`, keep `MINSUPPLIES 1` and
   `SHUTDOWNCMD "/sbin/shutdown -h +0"`, and set:

   ```text
   MONITOR ups@192.168.1.34:3493 1 ups <password-from-password-manager> secondary
   FORCESSL 1
   ```

   Keep `/etc/nut/upsmon.conf` readable by root and group `nut` only. Run
   `systemctl enable --now nut-monitor.service`. `secondary` monitors UPS
   status and shuts down Proxmox when NUT declares the UPS critical; Proxmox
   handles its VMs.
4. Check `ifquery --check vmbr0.1`, then confirm `ip route get 192.168.1.34`
   uses `vmbr0.1` with source `192.168.1.39` and no gateway. Check TCP `3493`
   with `nc -nvz -w 3 192.168.1.34 3493`, and inspect
   `systemctl status nut-monitor.service` and its journal for a TLS connection
   without login errors. An unauthenticated `upsc` query returns access denied.
   Enable Start at boot for VMs `102` and `103` in Proxmox Options, and confirm
   their guest agents with `qm agent 102 ping` and `qm agent 103 ping`.
