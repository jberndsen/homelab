# Home Assistant operations

Owner decisions: [system record](system.md). Current execution evidence and
follow-ups: [stability plan](../../../plans/home-assistant-stability.md).

## Reviewed updates

Review intervening release notes against installed integrations and automations.
Check Repairs, disk space, current sensor updates and lighting control. Create a
full HA-native backup, verify completion and retain a copy outside VM 102 before
updating. Keep the encryption key privately in the owner's password manager.
Update one component through Settings → System → Updates, wait for startup,
then check versions, Repairs, logs, P1/solar freshness, Hue and cloud access.
Stop on regressions. Keep unattended software updates disabled.

## Backup handling and recovery

Settings → System → Backups contains the native archives. Include settings and
history plus all applicable folders/apps. Download to an owner-controlled
location outside the VM. Agent downloads belong in ignored repository `tmp/`;
they must never enter Git. UI downloads may be decrypted on the fly even when
the server copy is encrypted: treat the downloaded archive as sensitive.

For a working installation: open Backups → Show all backups, upload the saved
archive if necessary, select it and the components to restore, then Restore.
Supply its key privately if requested. Restoration overwrites selected current
data and restarts Home Assistant; use the credentials from the backup afterward.
For replacement HAOS, upload the archive during onboarding. Ensure the original
instance cannot run concurrently against the same devices/IPs. Do not simply
downgrade Core against migrated data.

VM-level fallback is the Proxmox VM 102 archive on unas-pro; inspect freshness
and disk coverage before use. Avoid accidentally restoring VM 103 instead.
Restoration may require working internet access to reinstall components.
Neither archive creation nor inspecting a file proves a successful restore.
Schedule an isolated restore test with production device/network access blocked,
then validate boot, login, data and integrations before planning cutover.

References: [official backup/restore instructions](https://www.home-assistant.io/common-tasks/general/)
and [backup review](../../../plans/proxmox-backup-review-2026-10-21.md).

## HomeWizard P1 authentication recovery

Use the existing integration's Repair/reconfiguration flow to preserve its
entities and history. At the Authorize step, the owner presses the P1 meter's
front white button for two seconds, then submits the dialog promptly (the
pairing window is short). The owner completed this migration on October 23,
2026. Do not delete/re-add a working integration merely to clear a Repair.
After reconfiguration, check Repairs, fresh power readings, cumulative
import/export continuity and the existing Energy sources; observe the next
overnight boundary. Keep device cloud connectivity as chosen by the owner.

Reference: [HomeWizard integration](https://www.home-assistant.io/integrations/homewizard/).
