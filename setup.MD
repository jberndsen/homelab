1. Restore the UniFi network, VPN routing, firewall access, and local DNS described in [the network record](infrastructure/network/system.md).
2. Make the NAS media share available and restore its media from backup if needed; use the share defined in [the storage manifest](infrastructure/node-main/cluster/nfs-storage/media-nfs-persistentvolume.yaml).
3. Download and install Proxmox on `node-main`, using [the server record](infrastructure/node-main/system.md) for its version and network settings.
4. Restore the Ubuntu Server VM from a Proxmox backup, or create a new Ubuntu Server VM with the recorded network settings and SSH access.
5. On a new VM, install `nfs-common` and the recorded K3s version, using [the K3s configuration](infrastructure/node-main/secrets-encryption/k3s-server-config.yaml).
6. Clone this repository onto the development laptop and configure local `kubectl` access to the VM.
7. For a fresh cluster, apply [the cluster configuration](infrastructure/node-main/cluster), then follow [the Argo bootstrap guide](infrastructure/node-main/ns-argo/SETUP.md). Checkpoint 2 has installed Sealed Secrets and verified encrypted NAS key recovery. Apply the ApplicationSet with automation disabled, restore verified sealing keys using `/Volumes/backups/NUC/kubernetes/README.md`, and sync only Sealed Secrets (restart its controller if importing keys after it starts). Media is still managed manually: recreate the untracked Kubernetes Secrets from the password manager and provide the ignored Secret files required by the current media Kustomizations before applying [the production media stack](infrastructure/node-main/ns-media/overlays/prod). Do not sync media through Argo until checkpoint 3 in [the plan](argocd-plan.md) passes. The ignored Secret file dependency will be removed during that checkpoint.
8. If rebuilding without a VM backup, restore application data from available backups or complete each application's initial setup and connections to the other services.
9. Run [the infrastructure checks](infrastructure/node-main/tests) and verify application access, media access, and VPN routing.
10. Configure Proxmox VM backups and separate backups for NAS media, this repository, and credentials needed for recovery.

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
