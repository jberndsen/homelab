# Step 4: Radarr

## Backup and cutover

In old Radarr, create **System → Backup → Backup Now**, download the ZIP
to `plans/runtime/backups/radarr/`, then:

```bash
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose stop radarr'
kubectl apply -f plans/manifests/04-radarr.yaml
kubectl -n media rollout status deployment/radarr --timeout=300s
```

## Restore and acceptance

At `http://radarr.home.arpa` restore the ZIP through **System → Backup →
Restore Backup**, then:

1. Add `/data/Video/Movies` as a root folder. Then use **Movies → Movie
   Editor** to select every existing movie and change its root folder to that
   path. Do not enable any option that moves existing files: this is a database
   path reassignment because the files are already on the NAS.
2. Run a refresh/scan after the reassignment. In **System → Tasks**, run
   **Check Health** and confirm there is no missing `/movies` root-folder
   warning before removing the old root-folder record.
3. Configure/test Transmission at `http://transmission:9091` with the new RPC
   credentials and a movie category if one was previously used.
4. Do not add a Remote Path Mapping: both apps see `/data/Downloads/...`.
5. In Prowlarr, add/update Radarr to `http://radarr:7878`, test it, then enable
   the intended sync level.
6. Rescan, import one controlled completed download, and confirm the source and
   movie destination have the same inode (`stat -c %i`) when hardlinking is
   expected. Verify owner/group/mode and health.

STOP before proceeding if restore, root paths, import, or hardlink validation
fails.

## Rollback

```bash
kubectl -n media scale deployment/radarr --replicas=0
ssh jeroen@192.168.1.33 'cd /home/jeroen && docker compose start radarr'
```

Disable the new Prowlarr integration before restarting old Radarr. Preserve the
PVC and backup.
